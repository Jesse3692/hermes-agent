# Hermes Skills 提取、优化与去重机制分析

> 基于 hermes-agent 源码（截至 2026-07-19 main 分支）的深度分析。
> 核心结论：Hermes 的 skills 维护是一个「前台轻量提取 → 后台用量跟踪 → 周期性 LLM 伞形合并」的三层联动系统，设计上严格遵循「不删除只归档、确定性优先、写来源溯源、fail-closed 多重守卫」四大原则。

---

## 目录

- [1. 架构总览](#1-架构总览)
- [2. Skills 提取：从会话中沉淀](#2-skills-提取从会话中沉淀)
- [3. Skills 优化：生命周期管理](#3-skills-优化生命周期管理)
- [4. Skills 去重：伞形合并（Umbrella Consolidation）](#4-skills-去重伞形合并umbrella-consolidation)
- [5. 关键设计哲学与取舍](#5-关键设计哲学与取舍)
- [6. 核心文件清单](#6-核心文件清单)
- [7. 配置参考](#7-配置参考)
- [8. 数据流时序图](#8-数据流时序图)

---

## 1. 架构总览

Hermes 把 skills 维护拆成三个解耦的子系统，分别由不同模块承担：

```
┌─────────────────────────────────────────────────────────────────────┐
│  前台会话（每轮结束）                                                │
│  AIAgent.run_conversation()                                         │
│         │                                                           │
│         ▼                                                           │
│  ┌─────────────────────────┐    fork daemon thread                 │
│  │ _spawn_background_review│─────────────────────────────┐         │
│  └─────────────────────────┘                             │         │
│         │ foreground agent                               │         │
│         ▼                                                ▼         │
│  正常对话循环                                  ┌──────────────────┐ │
│  （prompt cache 不受影响）                     │ background_review│ │
│                                                │ .py             │ │
│                                                │                  │ │
│                                                │ replay 会话快照  │ │
│                                                │ + 提取 prompt    │ │
│                                                └────────┬─────────┘ │
│                                                         │           │
│                                                         ▼           │
│                                          skill_manage(action=create)│
│                                          + provenance 标记           │
│                                                         │           │
│                                                         ▼           │
│                                          ~/.hermes/skills/.usage.json│
│                                          (sidecar 遥测 + 来源标记)   │
│                                                         │           │
│  ┌─────────────────────────────────────────────────────┘           │
│  │                                                                  │
│  ▼  周期触发（默认 7 天 + 空闲 2h）                                  │
│  ┌────────────────────────────────────────────┐                     │
│  │ agent/curator.py                           │                     │
│  │                                            │                     │
│  │ 路径 A: apply_automatic_transitions()      │ ← 确定性，零 LLM     │
│  │   active → stale → archived（时间窗口）    │                     │
│  │                                            │                     │
│  │ 路径 B: _run_llm_review() [opt-in]         │ ← LLM 伞形合并      │
│  │   fork AIAgent + CURATOR_REVIEW_PROMPT     │                     │
│  │   聚类 → 合并/降级 → 归档                   │                     │
│  └────────────────────────────────────────────┘                     │
└─────────────────────────────────────────────────────────────────────┘
```

三个子系统的关系：

| 子系统 | 触发频率 | 成本 | 产出 |
|--------|----------|------|------|
| 提取（background_review） | 每轮对话后 | 守护线程，复用热缓存或冷写摘要 | 新建/patch skill，带 `created_by: agent` 标记 |
| 优化-确定性迁移 | 每次 curator run | 零 LLM（纯时间窗口判断） | 状态迁移 active→stale→archived |
| 优化-LLM 去重 | opt-in（`curator.consolidate: true`） | aux-model，50-100 次 API 调用 | 伞形合并 + 分类调和 + cron 引用迁移 |

---

## 2. Skills 提取：从会话中沉淀

### 2.1 触发机制

提取发生在**每一轮对话结束后**，由 `AIAgent._spawn_background_review()` 触发：

- **入口**：`run_agent.py:1642` `_spawn_background_review()`
- **实际逻辑**：`agent/background_review.py:956` `spawn_background_review_thread()`
- **线程模型**：daemon 线程，主对话循环不阻塞，prompt cache 完全不受影响

```python
# run_agent.py:1642-1669
def _spawn_background_review(self, messages_snapshot, review_memory=False, review_skills=False):
    from agent.background_review import spawn_background_review_thread
    from tools.thread_context import propagate_context_to_thread
    target, _prompt = spawn_background_review_thread(
        self, messages_snapshot,
        review_memory=review_memory,
        review_skills=review_skills,
    )
    t = threading.Thread(
        target=propagate_context_to_thread(target),
        daemon=True, name="bg-review",
    )
    t.start()
```

### 2.2 路由感知的 replay 策略

`_resolve_review_runtime()`（background_review.py:46-110）根据「是否路由到不同模型」选择 replay 策略：

| 配置 | replay 策略 | 理由 |
|------|-------------|------|
| 同模型（默认/auto） | 完整 replay 会话 | 复用父进程热缓存，成本几乎为零 |
| 异模型（`auxiliary.background_review.{provider,model}`） | 只 replay 最近 24 条 + 更早折叠成摘要 | 冷写不同模型缓存无意义，减少 token |

摘要折叠逻辑见 `_digest_history()`（background_review.py:122-163）：
- 保留最近 `tail=24` 条消息原文
- 更早的 user/assistant 消息折叠成单条合成 user 消息
- 保持 role 交替（tool 消息不会被截断在前）

### 2.3 提取信号（_SKILL_REVIEW_PROMPT）

提取 prompt 定义在 `background_review.py:181-284`，明确列出了必须捕获的信号类型：

**必捕获信号**（任一触发即应行动）：
1. 用户纠正了风格/语调/格式/冗长度 → 写进对应 skill 的 pitfall
2. 用户纠正了工作流/方法/步骤顺序 → 编码为 pitfall 或显式步骤
3. 出现非平凡技巧/修复/workaround/调试路径/工具使用模式 → 捕获
4. 本次会话加载的 skill 有错漏/过时 → 立即 patch

**明确禁止捕获**（避免变成持久的自我约束）：
- 环境相关失败（缺二进制、fresh-install 错误、路径不匹配）
- 对工具的负面断言（"browser 工具不工作"）——会硬化成数月的拒绝
- 会话特定的瞬态错误（重试成功后，教训是重试模式而非原始失败）
- 一次性任务叙述（"总结今天的市场"不构成一类工作）

### 2.4 偏好顺序（避免碎片化）

提取 prompt 强制以下优先级（background_review.py:206-241）：

1. **patch 已加载的 skill** —— 本次会话 `/skill-name` 加载或 `skill_view` 读过的
2. **patch 已有 umbrella** —— `skills_list` + `skill_view` 找到覆盖该领域的类级别 skill
3. **给 umbrella 加 support 文件**：
   - `references/<topic>.md` —— 会话特定细节 + 浓缩知识库
   - `templates/<name>.<ext>` —— 可复制修改的起手文件
   - `scripts/<name>.<ext>` —— 可重复运行的确定性动作
4. **最后才是新建类级别 umbrella** —— 名字必须是类级别，禁止 PR 号/错误字符串/codename

### 2.5 写入来源标记（Provenance）

这是 hermes 区分「自动提取的 skill」vs「用户手动写的 skill」的核心机制：

```python
# tools/skill_provenance.py
_write_origin: contextvars.ContextVar[str] = ContextVar(
    "skill_write_origin", default="foreground",
)
BACKGROUND_REVIEW = "background_review"

def is_background_review() -> bool:
    return get_current_write_origin() == BACKGROUND_REVIEW
```

- **前台 agent**：默认 `foreground`，创建的 skill **不**打 `created_by: agent` 标记，curator 永不动
- **后台 review fork**：`_memory_write_origin = "background_review"`（run_agent.py 中 fork 时设置），创建的 skill 打上 `created_by: agent`，进入 curator 管理范围

只有 `created_by == "agent"` 或 `agent_created is True` 的记录才被 curator 管理（`tools/skill_usage.py:473-477` `_is_curator_managed_record`）。

### 2.6 提取的护栏

- **保护性 skill 不可编辑**：bundled/hub-installed skill 在 prompt 中明确标注 "DO NOT edit"
- **pinned skill 可改进但不可删**：pin 只阻止删除/归档/合并，不阻止内容更新
- **负面前提检查**：如果只有 protected skill 需要更新，回复 "Nothing to save." 并停止

---

## 3. Skills 优化：生命周期管理

### 3.1 两条独立路径

优化由 `agent/curator.py` 编排，分两条路径：

| 路径 | 函数 | 触发 | 成本 | 动作 |
|------|------|------|------|------|
| A. 确定性迁移 | `apply_automatic_transitions()` | 每次 curator run | 零 LLM | active↔stale→archived |
| B. LLM 伞形合并 | `_run_llm_review()` | opt-in（`curator.consolidate: true`） | aux-model 50-100 次调用 | 聚类/合并/降级/归档 |

**核心成本控制点**：`DEFAULT_CONSOLIDATE = False`（curator.py:78）。路径 A 每次都跑且免费；路径 B 默认关闭，用户显式 opt-in 或 `hermes curator run --consolidate` 才触发。

### 3.2 触发门控（should_run_now）

`should_run_now()`（curator.py:233-283）检查：

1. `curator.enabled`（默认 True）
2. 未 paused（`~/.hermes/skills/.curator_state` 的 `paused` 字段）
3. `last_run_at` 存在且超过 `interval_hours`（默认 7 天）

**首次运行特殊处理**：没有 `last_run_at` 时，**不立即运行**，而是 seed `last_run_at = now`，推迟一个完整周期。避免 `hermes update` 后第一次 tick 就自动整理库。用户想立即预览可 `hermes curator run --dry-run`。

`maybe_run_curator()`（curator.py:1998-2016）额外检查 `idle_for_seconds >= min_idle_hours * 3600`（默认 2 小时空闲）。

### 3.3 路径 A：确定性状态迁移

`apply_automatic_transitions()`（curator.py:305-383）是纯函数式的时间窗口判断：

```python
# curator.py:321-322
stale_cutoff = now - timedelta(days=get_stale_after_days())    # 默认 30 天
archive_cutoff = now - timedelta(days=get_archive_after_days())  # 默认 90 天
```

**状态机**：
```
active  ──(last_activity > stale_after_days)──►  stale
stale   ──(last_activity > archive_after_days)──►  archived（移到 .archive/）
stale   ──(重新被使用)──►  active（reactivated）
```

**`last_activity_at` 推导**（`tools/skill_usage.py:146-163` `latest_activity_at`）：
```python
# 取 max(last_used_at, last_viewed_at, last_patched_at)
# 注意：created_at 不计入 activity，便于区分「从未活跃」的 skill
```

**保护机制**（curator.py:331-369）：
1. **pinned skill**：跳过所有自动迁移
2. **cron 引用的 skill**：跳过（`_cron_referenced_skills()` 读取所有 cron job 的 skill 引用，含 paused/disabled）
3. **首次见到的 eligible skill**：seed 记录，时钟锚定到 now，推迟一个周期
4. **从未使用（use=0）且创建未满 stale_after_days**：完全不动（「absence of evidence ≠ evidence of staleness」）

**归档动作**：调用 `skill_usage.archive_skill(name)`，把目录移到 `~/.hermes/skills/.archive/<name>/`，可 `hermes curator restore <name>` 恢复。**绝不删除**。

### 3.4 路径 B：LLM 伞形合并

详见下一节 [§4](#4-skills-去重伞形合并umbrella-consolidation)。

### 3.5 运行前快照

`run_curator_review()`（curator.py:1494-1755）在执行前先做 tar.gz 快照：

```python
# curator.py:1549-1558
from agent import curator_backup
snap = curator_backup.snapshot_skills(reason="pre-curator-run")
```

快照失败不阻塞运行（best-effort），但会 log。这保证任何误判都能整体回滚。

### 3.6 运行报告

每次 run 写入 `~/.hermes/logs/curator/{YYYYMMDD-HHMMSS}/`：
- `run.json` —— 机器可读，完整保真（含 tool_calls、llm_final、分类结果、cron 重写记录）
- `REPORT.md` —— 人类可读，含 auto-transitions / consolidated / pruned / 新增 / 状态迁移 / cron 重写各节
- `cron_rewrites.json` —— 仅当有 cron job 被改时写入

---

## 4. Skills 去重：伞形合并（Umbrella Consolidation）

### 4.1 核心理念

去重**不是基于相似度阈值**，而是基于一个判断标准（curator.py:458-462）：

> "would a human maintainer write this as N separate skills, or as one skill with N labeled subsections?"
> （人类维护者会写成 N 个独立 skill，还是 1 个带 N 个子章节的 skill？）

当答案是后者时，合并。目标形态是**类级别 skill + references/templates/scripts 支持文件**，明确反对「一个 session 一个 skill」的碎片化。

### 4.2 去重 prompt（CURATOR_REVIEW_PROMPT）

定义在 `curator.py:417-568`，核心流程：

#### 步骤 1：前缀聚类（curator.py:464-468）
扫描候选列表，找共享首词/领域关键词的 cluster。预期示例：
- `hermes-config-*`, `hermes-dashboard-*`, `gateway-*`, `codex-*`
- `ollama-*`, `anthropic-*`, `gemini-*`, `mcp-*`
- `salvage-*`, `pr-*`, `competitor-*`, `python-*`, `security-*`

预期 10-25 个 cluster。

#### 步骤 2：对每个 2+ 成员的 cluster 判断 umbrella 类
不问「这些 pair 是否重叠」，问「这些 skill 服务的是什么 umbrella 类？维护者会命名这个类并写一个 skill 吗？」

#### 步骤 3：三种合并方式（curator.py:474-496）

| 方式 | 适用场景 | 操作 |
|------|----------|------|
| a. MERGE INTO EXISTING UMBRELLA | cluster 中已有一个够宽的 | patch umbrella 加子章节，归档 siblings |
| b. CREATE NEW UMBRELLA | 没有够宽的 | `skill_manage action=create` 新建类级别 skill |
| c. DEMOTE TO SUPPORT FILE | 窄但有会话特定价值 | 移到 umbrella 的 `references/`/`templates/`/`scripts/` 下 |

#### 硬规则（curator.py:431-462）
1. 不动 bundled/hub-installed/external-dir skill
2. 不删除任何 skill，归档是最大破坏性动作
3. 不动 pinned skill
4. 不用 use_count 作为跳过合并的理由（counter 是新的，常为 0）
5. 不用「每个 skill 有不同 trigger」作为拒绝合并的理由
6. 名字过窄（含 PR 号/codename/错误字符串/audit/diagnosis/salvage）必须降级

#### 包完整性检查（curator.py:497-515）
降级或归档前必须检查**整个目录包**，不只是 SKILL.md。skill 根可能含 `references/`、`templates/`、`scripts/`、`assets/`。禁止：
- 只把 SKILL.md 扁平化到 `<umbrella>/references/<old>.md`
- 留下指向已移动文件的悬空链接

三种安全路径：
- 保持独立 skill
- 完整合并（每个 support 文件 re-home 到 umbrella 对应目录 + 重写路径引用）
- 整个原始 skill 包原样归档

### 4.3 去重的安全护栏

这是 hermes 区别于普通 LLM 整理工具的核心。`tools/skill_manager_tool.py` 实现了多层 fail-closed 守卫：

#### 4.3.1 absorbed_into 强制声明（_curator_consolidation_delete_guard）

`skill_manager_tool.py:438-485`：后台 curator fork 调用 `delete` 时，**必须**传：
- `absorbed_into=<umbrella>` —— 合并，目标必须存在
- `absorbed_into=""` —— 明确 prune（无转发目标）

**不传直接拒绝**。这是为修复 issue #29912：之前 consolidation pass 归档了整簇 active skill 却没有任何合并证据，导致 cron job 指向不存在的 skill。

```python
# skill_manager_tool.py:469-485
declared = isinstance(absorbed_into, str) and absorbed_into.strip()
if declared:
    return None  # 允许
return {
    "success": False,
    "error": f"Refusing background curator delete of skill '{name}': "
             "the consolidation pass may only archive a skill it has absorbed into "
             "an umbrella. Pass absorbed_into=<umbrella>...",
    "_fail_closed": True,
}
```

#### 4.3.2 目标存在性验证（_delete_skill）

`skill_manager_tool.py:1077-1099`：声明 `absorbed_into` 时，目标 umbrella **必须在磁盘上存在**，否则拒绝。防止 LLM 幻觉不存在的 umbrella 名。

```python
# skill_manager_tool.py:1084-1099
if is_consolidation:
    target_name = absorbed_target
    if target_name == name:
        return {"success": False, "error": "cannot equal the skill being deleted."}
    target = _find_skill(target_name)
    if not target:
        return {"success": False, "error": f"absorbed_into='{target_name}' does not exist."}
```

#### 4.3.3 后台写权限守卫（_background_review_write_guard）

`skill_manager_tool.py:297-396`：后台 fork 对以下 skill 的写操作全部拒绝：
- external skill path（`skills.external_dirs`）
- protected built-in（如 `plan`）
- hub-installed
- bundled
- 非 agent-created（`created_by != "agent"`）

前台 agent 可做用户指导的编辑，后台不行。

#### 4.3.4 读后写守卫（_background_review_read_before_write_guard）

`skill_manager_tool.py:399-426`：后台 fork 必须**先 `skill_view` 读取目标文件内容**才能 patch/edit。防止基于推断改内容。

```python
# skill_manager_tool.py:413-426
if _background_review_has_read(target):
    return None  # 已读，允许
return {
    "success": False,
    "error": f"the current {file_label} content has not been loaded in this review turn. "
             "Call skill_view(name) for SKILL.md, or skill_view(name, file_path=...) ...",
    "_read_before_write_required": True,
}
```

`skill_view` 返回内容后会调用 `mark_background_review_skill_read(path)`（skill_manager_tool.py:56-78）记录已读路径到 ContextVar。

#### 4.3.5 curator 走 archive 不走 rmtree

`skill_manager_tool.py:1122-1133`：`is_background_review()` 为真时，delete 路由到 `archive_skill()`（移到 `.archive/`，可恢复），**绝不** `shutil.rmtree`：

```python
# skill_manager_tool.py:1116-1133
if curator_pass:
    from tools.skill_usage import archive_skill
    ok, archive_msg = archive_skill(name)
    if not ok:
        return {"success": False, "error": archive_msg}
    message = f"Skill '{name}' archived ({archive_msg})."
    if is_consolidation:
        message += f" Content absorbed into '{absorbed_target}'."
    return {"success": True, "message": message, "_archived": True}
```

#### 4.3.6 删除路径遍历守卫（_validate_delete_target）

`skill_manager_tool.py:209-267`：在 `shutil.rmtree` 前的最后防线，拒绝：
1. 不在任何已知 skills root 内的路径
2. skills root 自身（会删全部 skill）
3. symlink/junction（防止 poisoned tree 把 rmtree 重定向到 skills 树外）

这是为防御 Kilo Code #11227 类问题（built-in sentinel 解析到 cwd，递归删除整个工作目录）。

### 4.4 去重分类的信号调和

一次 curator run 结束后，要把「被移除的 skill」分成 **consolidated**（合并进 umbrella）vs **pruned**（纯过期归档）。这个分类决定了 cron 引用如何迁移、报告如何呈现。

`_reconcile_classification()`（curator.py:872-1000）用**三级信号**，优先级递减：

| 优先级 | 信号 | 来源 | 可信度 |
|--------|------|------|--------|
| 1 | model-declared `absorbed_into` at delete time | `_extract_absorbed_into_declarations()` 扫 tool calls | 最权威，模型在删除瞬间的明确意图 |
| 2 | structured YAML block | `_parse_structured_summary()` 解析 final response 的 ```yaml 块 | 模型事后声明，有幻觉风险 |
| 3 | heuristic tool-call audit | `_classify_removed_skills()` 扫 write_file/patch 目标引用 | 客观证据，但可能遗漏 |

**调和规则**（curator.py:913-999）：

1. **模型声明 `absorbed_into` 且目标存在于 destinations** → consolidated（最权威）
2. **模型声明 `absorbed_into=""`** → explicit prune
3. **模型声明 `absorbed_into=Y` 但 Y 不在 destinations** → 幻觉，降级用启发式或归为 pruned
4. **模型 YAML 说 consolidated 且 into 存在** → consolidated
5. **模型 YAML 说 consolidated 但 into 不存在** → 幻觉，用启发式或 pruned
6. **启发式找到合并但模型没提** → consolidated（标 `source="tool-call audit"`）
7. **其他** → pruned

每个被删 skill 必须落进且仅落进一个桶。

### 4.5 Cron 引用迁移

合并后，`cron/jobs.py::rewrite_skill_refs` 会重写所有 cron job 的 skill 引用（curator.py:1190-1221）：

- **consolidated**：`X -> Y` 映射，job 的 skill 列表里 X 被 Y 替换
- **pruned**：X 从 skill 列表里直接 drop

避免定时任务指向已归档的 skill。`cron_rewrites.json` 记录每次改写，写入 run dir 供审计。

### 4.6 用户可见的归档摘要

`_build_rename_summary()`（curator.py:1003-1090）生成「where did my skills go?」摘要，附加到 `final_summary`：

```
archived 4 skill(s):
  • pdf-extraction -> document-tools
  • docx-extraction -> document-tools
  • flaky-thing - pruned (stale)
  • old-utility -> spreadsheet-ops
full report: hermes curator status
keep an umbrella stable: hermes curator pin document-tools
```

最多显示 10 条，完整列表在 REPORT.md。pruned-only 的 run 不显示 pin 提示。

---

## 5. 关键设计哲学与取舍

### 5.1 不删除，只归档

- `.archive/` 全可恢复，`hermes curator restore <name>` 一键还原
- curator 路径强制走 `archive_skill()` 而非 `rmtree`
- 这是文档化的硬约束：「Archives are recoverable; deletion is not.」

### 5.2 确定性优先

- 时间窗口判断（路径 A）零 LLM 成本，每次 curator run 都跑
- LLM 合并（路径 B）是 opt-in 的「锦上添花」，默认 `consolidate: false`
- 用户想立即预览可 `hermes curator run --dry-run`（不修改库，只写报告）

### 5.3 写来源溯源

- ContextVar 区分前台用户写入 vs 后台自动提取
- 只有 `created_by == "agent"` 的 skill 可被 curator 动
- 用户手动写的 skill（URL install / 直接 SKILL.md 编辑）永远不被自动整理
- 这避免了「自动系统删了用户精心写的内容」的事故

### 5.4 fail-closed 多重守卫

每种失败模式都有对应拒绝路径：

| 失败模式 | 守卫 |
|----------|------|
| LLM 幻觉不存在的 umbrella | `_delete_skill` 目标存在性验证 |
| 删除时未声明意图 | `_curator_consolidation_delete_guard` 拒绝 |
| 后台写 external/bundled/hub skill | `_background_review_write_guard` 拒绝 |
| 后台未读就写 | `_background_review_read_before_write_guard` 拒绝 |
| 路径遍历攻击 | `_validate_delete_target` 拒绝 |
| cron 引用断裂 | `rewrite_skill_refs` 自动迁移 |
| 误判归档 | 运行前 tar.gz 快照 + `.archive/` 可恢复 |

### 5.5 类级别而非实例级别

prompt 反复强调「一个 broad umbrella with subsections beats five narrow siblings」。明确反对：
- 一个 session 一个 skill 的碎片化
- 名字含 PR 号/错误字符串/codename 的实例级 skill
- 用「每个 skill 有不同 trigger」作为拒绝合并的理由

### 5.6 缓存友好

- 提取在守护线程，主会话 prompt cache 不受影响
- 同模型 review 复用热缓存
- 异模型 review 用摘要减少冷写
- curator fork `skip_memory=True`、`skip_context_files=True`，不污染主上下文

---

## 6. 核心文件清单

按角色分两层：**主清单**是 skills 维护链路的 6 个核心文件，**辅助文件**是周边接入层（CLI、cron 迁移、fork 入口）。

### 6.1 主清单（核心链路）

| 文件 | 角色 | 关键函数 |
|------|------|----------|
| `agent/background_review.py` | 提取（每轮 fork review） | `spawn_background_review_thread`, `_SKILL_REVIEW_PROMPT`, `_digest_history` |
| `tools/skill_provenance.py` | 来源标记（ContextVar） | `is_background_review`, `set_current_write_origin` |
| `tools/skill_manager_tool.py` | 写入 + 多重守卫 | `_create_skill`, `_delete_skill`, `_curator_consolidation_delete_guard`, `_background_review_write_guard`, `_background_review_read_before_write_guard` |
| `tools/skill_usage.py` | 遥测 sidecar + agent_created 判定 | `latest_activity_at`, `is_curation_eligible`, `_is_curator_managed_record`, `archive_skill`, `list_agent_created_skill_names` |
| `agent/curator.py` | 优化/去重编排 | `apply_automatic_transitions`, `run_curator_review`, `_run_llm_review`, `_reconcile_classification`, `_build_rename_summary`, `CURATOR_REVIEW_PROMPT` |
| `agent/curator_backup.py` | 运行前 tar.gz 快照 | `snapshot_skills` |

### 6.2 辅助文件（接入层）

| 文件 | 角色 | 关键函数 / 入口 |
|------|------|-----------------|
| `run_agent.py` | 提取 fork 入口（调用 background_review） | `AIAgent._spawn_background_review` (line 1642) |
| `cron/jobs.py` | cron 引用迁移（去重后改写 job 的 skill 引用） | `rewrite_skill_refs`, `referenced_skill_names` |
| `hermes_cli/curator.py` | CLI 子命令层（`hermes curator <verb>` 的 argparse 接线） | `hermes curator status/run/pin/archive/restore/...` |

---

## 7. 配置参考

### 7.1 curator 配置（~/.hermes/config.yaml）

```yaml
curator:
  enabled: true                    # 默认开
  consolidate: false               # 默认关，LLM 伞形合并 opt-in
  interval_hours: 168              # 7 天，curator run 间隔
  min_idle_hours: 2                # 代理空闲 2h 才触发
  stale_after_days: 30             # 闲置 30d 标记 stale
  archive_after_days: 90           # 闲置 90d 归档
  prune_builtins: true             # 允许归档 bundled built-in（默认开）
  backup:
    # 运行前 tar.gz 快照配置
```

### 7.2 auxiliary 路由（可选）

```yaml
auxiliary:
  curator:
    provider: openrouter           # 或 anthropic / auto（用主模型）
    model: claude-sonnet-4
    # base_url, api_key, extra_body 等同其他 aux task
  background_review:
    provider: openrouter           # 提取 fork 的模型
    model: claude-haiku-4          # 用便宜模型做提取
```

### 7.3 关键路径

| 路径 | 用途 |
|------|------|
| `~/.hermes/skills/` | skill 库根目录 |
| `~/.hermes/skills/<name>/SKILL.md` | skill 主文件 |
| `~/.hermes/skills/<name>/{references,templates,scripts,assets}/` | 支持文件 |
| `~/.hermes/skills/.usage.json` | 遥测 sidecar（use_count/view_count/state/pinned/created_by） |
| `~/.hermes/skills/.curator_state` | curator 调度状态（last_run_at/paused/run_count） |
| `~/.hermes/skills/.archive/` | 归档目录（可恢复） |
| `~/.hermes/skills/.curator_suppressed` | 已 prune 的 built-in 列表（防止 hermes update 重新 seed） |
| `~/.hermes/skills/.bundled_manifest` | bundled skill 清单（name:hash） |
| `~/.hermes/skills/.hub/lock.json` | hub-installed skill 清单 |
| `~/.hermes/logs/curator/{timestamp}/` | 每次 run 的报告（run.json + REPORT.md + cron_rewrites.json） |

### 7.4 CLI 命令

```bash
hermes curator status              # 查看状态
hermes curator run                 # 立即运行（仅确定性迁移）
hermes curator run --consolidate   # 立即运行（含 LLM 合并）
hermes curator run --dry-run       # 预览，不修改库
hermes curator pause/resume        # 暂停/恢复
hermes curator pin <name>          # 固定 skill，跳过所有自动迁移
hermes curator unpin <name>
hermes curator archive <name>      # 手动归档
hermes curator restore <name>      # 恢复归档
hermes curator prune               # 清理旧归档
hermes curator backup              # 手动快照
hermes curator rollback            # 回滚到快照
```

---

## 8. 数据流时序图

### 8.1 提取流程（每轮对话后）

```
用户消息 → AIAgent.run_conversation()
              │
              ├─ 正常对话循环（工具调用、LLM 响应）
              │
              └─ 轮次结束
                  │
                  ▼
          _spawn_background_review(messages_snapshot)
                  │
                  ▼ (daemon thread)
          spawn_background_review_thread()
                  │
                  ├─ _resolve_review_runtime()  ← 决定同模型/异模型
                  │
                  ├─ [同模型] 完整 replay
                  │  [异模型] _digest_history() 折叠
                  │
                  ▼
          fork AIAgent (memory_write_origin="background_review")
                  │
                  ▼
          执行 _SKILL_REVIEW_PROMPT
                  │
                  ├─ skills_list / skill_view  (读)
                  │
                  ├─ skill_manage(action=patch)  ← 优先 patch 已有
                  ├─ skill_manage(action=write_file)  ← 加 support 文件
                  └─ skill_manage(action=create)  ← 最后才新建
                        │
                        ▼
                  skill_provenance 标记 created_by="agent"
                        │
                        ▼
                  写入 ~/.hermes/skills/<name>/
                  更新 ~/.hermes/skills/.usage.json
```

### 8.2 优化与去重流程（周期触发）

```
会话启动 / gateway tick
        │
        ▼
maybe_run_curator(idle_for_seconds)
        │
        ├─ should_run_now()?  ── No ──► 返回
        │   (enabled & !paused & >interval_hours)
        │
        ├─ idle >= min_idle_hours?  ── No ──► 返回
        │
        ▼
run_curator_review()
        │
        ├─ curator_backup.snapshot_skills()  ← tar.gz 快照
        │
        ├─ 路径 A: apply_automatic_transitions()  ← 零 LLM
        │   ├─ 遍历 agent_created skills
        │   ├─ pinned / cron-referenced 跳过
        │   ├─ use=0 且年轻 跳过
        │   └─ 时间窗口判断 → stale / archived / reactivated
        │
        ├─ consolidate? (config 或 --consolidate)
        │   │
        │   ├─ No  ──► 只写报告，返回（零 LLM 成本）
        │   │
        │   ▼ Yes
        │   _run_llm_review(prompt)
        │       │
        │       ├─ fork AIAgent (max_iterations=9999, skip_memory=True)
        │       │   review_agent._memory_write_origin = "background_review"
        │       │
        │       ▼
        │   执行 CURATOR_REVIEW_PROMPT
        │       │
        │       ├─ 前缀聚类
        │       ├─ 三种合并方式（merge/create/demote）
        │       │   ├─ skill_manage(action=patch)   ← patch umbrella
        │       │   ├─ skill_manage(action=create)  ← 新建 umbrella
        │       │   ├─ skill_manage(action=write_file) ← 加 support
        │       │   └─ skill_manage(action=delete, absorbed_into=Y)
        │       │       │
        │       │       ▼
        │       │   多重守卫检查：
        │       │     ├─ _curator_consolidation_delete_guard (必须声明 absorbed_into)
        │       │     ├─ _background_review_write_guard (不动 external/bundled/hub/pinned)
        │       │     ├─ _background_review_read_before_write_guard (必须先读)
        │       │     ├─ _pinned_guard (pinned 拒绝)
        │       │     ├─ _validate_delete_target (路径遍历守卫)
        │       │     └─ archive_skill() (走 archive 不走 rmtree)
        │       │
        │       └─ 输出 structured YAML (consolidations/prunings)
        │
        ▼
   _write_run_report()
        │
        ├─ _classify_removed_skills()      ← 启发式审计
        ├─ _parse_structured_summary()     ← 解析 YAML 块
        ├─ _extract_absorbed_into_declarations()  ← 最权威信号
        │
        ▼
   _reconcile_classification()  ← 三级信号调和
        │
        ├─ consolidated: [{name, into, source, reason}]
        └─ pruned: [{name, source, reason}]
        │
        ▼
   cron/jobs.py::rewrite_skill_refs()
        ├─ consolidated: X → Y 替换
        └─ pruned: X 从列表 drop
        │
        ▼
   写入 ~/.hermes/logs/curator/{timestamp}/
        ├─ run.json
        ├─ REPORT.md
        └─ cron_rewrites.json (如有改写)
        │
        ▼
   更新 ~/.hermes/skills/.curator_state
        (last_run_at, run_count, last_run_summary)
```

---

## 附录：关键 issue 与历史教训

| Issue | 问题 | 修复 |
|-------|------|------|
| #29912 | consolidation pass 归档整簇 active skill 无合并证据，cron 断裂 | 引入 `absorbed_into` 强制声明 + fail-closed 守卫 |
| #25839 | 后台 review fork 编辑 pinned skill | `_background_review_write_guard` 对 pinned 也拒绝（比前台更严） |
| #3575 | 硬编码 `~/.hermes` 路径破坏 profiles | 全局改用 `get_hermes_home()` |
| #5295 | honcho 插件硬编码 argparse 在 main.py | 移除，改用 `ctx.register_cli_command` |
| Kilo Code #11227 | sentinel 解析到 cwd，递归删除工作目录 | `_validate_delete_target` 路径遍历守卫 |

---

*文档生成时间：2026-07-19*
*基于源码版本：main 分支 09109fec9*
