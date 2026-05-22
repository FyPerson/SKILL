# memory-keeper

**Language**: [English](#english) | [中文](#中文)

---

<a id="english"></a>

A disciplined session-memory keeper for Claude Code. Distills decisions, preferences, gotchas, todos and milestones from your conversation into structured worklog files — with **per-item confirmation** and a **long-term vs one-off** distinction.

> Built to override the LLM reflex of "dump the full draft in one message". Counter-intuitive on purpose.

## Why another memory skill?

Claude Code already has `/memory`. So does every "save my chat" plugin out there. They mostly do one of two things:

1. **Save the whole conversation** (you re-read a wall of text later)
2. **Auto-summarize on every turn** (silent drift, no provenance)

`memory-keeper` does neither. It is **manually triggered**, **per-item confirmed**, and **distinguishes long-term rules from one-off context**. It produces a worklog you can actually grep two months later.

## What it gives you

- **Daily session logs** under `~/.claude/worklog/<project>/session_YYYYMMDD.md` using a CAR structure (Context, Action, Result)
- **A separate `MEMORY.md` index** with one-line hooks linking to `memory/feedback_<topic>.md` detail files — so long-term rules don't pollute the daily log
- **Recurrence detection**: if the same theme shows up across ≥ 2 sessions but isn't in MEMORY.md yet, the skill flags it and offers promotion
- **A review mode** (`/memory-keeper review`) that stitches your last session + unfinished todos + recent git commits into a 30-line handoff summary on a fresh session

## Install

```bash
# Clone into your Claude Code skills directory
git clone https://github.com/<you>/memory-keeper ~/.claude/skills/memory-keeper
```

On Windows: `%USERPROFILE%\.claude\skills\memory-keeper`

The skill auto-creates `~/.claude/worklog/` on first run (with confirmation). Override with `$CLAUDE_WORKLOG_ROOT` if you want a different root.

## Usage

```
/memory-keeper                  # distill the current session (default)
/memory-keeper review           # read recent logs + git log + todos, output handoff
/memory-keeper <project-name>   # distill to a specific project subdir
```

That's the whole surface area.

## The two non-obvious design choices

Both came from real-world failures where the skill was violated by an LLM "helpfully" dumping the entire draft. Both are explicitly designed to be **counter-intuitive** for the model:

### 1. Startup constraint restatement

The first line of any `/memory-keeper` run is the LLM repeating the protocol back to you in its own words:

> "Running /memory-keeper per the skill's per-item protocol: **one item shown per message + wait for user reply before next + no full draft pre-dumped to chat + no 'approve all' escape hatch**."

Missing this line = violation. You can interrupt and ask for a restart.

**Why this works**: system messages are easy to skim past. A constraint restated *in the conversation stream* is impossible to ignore on the next turn.

### 2. Pre-output three-question self-check

Before any draft item is sent to you, the model self-asks:

1. Am I about to send **exactly 1 item** in this message?
2. Does the message contain an **"approve all" escape hatch**?
3. Am I organizing the output as a **list dump** (multiple categories at once)?

Any "yes" → rewrite, show only item #1, end with "Look at this one first?"

**Why this works**: LLMs have strong bias toward "list all items" for analysis tasks (training-data reflex). The three-question gate is the cheapest possible intervention that catches the reflex before it ships.

## Storage layout

```
~/.claude/worklog/
├── INDEX.md                              # Global index (subdir map + cross-project preferences)
├── <project-a>/
│   ├── INDEX.md                          # Project-level index (session log links)
│   ├── MEMORY.md                         # Long-term preferences/state INDEX (no rule bodies)
│   ├── session_20260522.md               # Daily session log (CAR structure)
│   └── memory/
│       ├── feedback_<topic>.md           # Long-term preference detail (Why + How to apply)
│       └── <topic>.md                    # Long-term project state detail
└── <project-b>/...
```

**Layering invariant**: `INDEX.md` and `MEMORY.md` are *index files*. They contain one-line hooks linking to detail files in `memory/`. Rule bodies never live in indexes. Iterating a rule = edit one detail file, all indexes auto-stay in sync.

## What gets extracted

Five categories, CAR-structured:

| Category | Confirmation style | Notes |
|---|---|---|
| Key decisions | per-item | Includes rejected alternatives' reasons when clear |
| Personal preferences | per-item + long-term/one-off | The long-term/one-off split is the one that prevents MEMORY.md from rotting |
| Gotchas | per-item | Symptom + root cause + verified fix |
| Technical implementation | batch | Code records, file paths |
| Milestones / Todos | batch | Completed vs P1/P2 outstanding |

## Recurrence detection

On every distill, the skill scans the last 3 sessions + INDEX.md keywords. If a theme in the new draft matches keywords seen in ≥ 2 historical sessions but is NOT yet in MEMORY.md, you get a ⚠️ flag and a "promote to long-term?" offer at confirmation time.

This is the mechanism that prevents "I've told you this before" repetition across weeks.

## Review mode

`/memory-keeper review` on a fresh session reads:

- Latest session log of the inferred project
- Aux subdirs' keyword-matched sessions (cross-project recurrence)
- `git log --oneline -10` if the project has a linked code dir
- MEMORY.md unfinished todos

…and outputs a ≤ 30-line handoff with: last session summary / top 3 decisions / aux hits / milestones / unfinished todos / new commits / promotion suggestions / suggested next step.

**Review mode writes nothing.** Pure read.

## Non-goals

- Does not auto-trigger from "remember this" / "save it" / "let's stop here". Manual invocation only.
- Does not replace Claude Code's built-in `/memory`. It is a separate, opinionated workflow.
- Does not silently create directories or rewrite index structure. If something is ambiguous, it asks.

## License

MIT

---

<a id="中文"></a>

# memory-keeper（中文）

**语言**：[English](#english) | [中文](#中文)

一个给 Claude Code 用的会话记忆沉淀工具。把你对话里的关键决策、偏好、踩坑、待办、里程碑结构化地写入工作日志——**逐条确认**，且**区分"长期偏好"和"一次性需求"**。

> 设计目标是压住 LLM 那个"一次性甩完全部草稿"的反射。反直觉是刻意的。

## 为什么再造一个记忆 skill？

Claude Code 原生有 `/memory`，市面上"保存对话"的插件也很多。它们基本是两种模式：

1. **保存整段对话**（你两个月后翻一堵文字墙）
2. **每轮自动摘要**（黑盒漂移，无溯源）

`memory-keeper` 两个都不是。它**仅手动触发**、**逐项确认**、**强制区分长期/临时**。它给你的工作日志，是两个月后能 grep 出具体决策的那种。

## 它给你什么

- 按天归档的会话日志，路径 `~/.claude/worklog/<project>/session_YYYYMMDD.md`，CAR 结构（Context 现状 / Action 行动 / Result 结论）
- 独立的 `MEMORY.md` 索引文件，一行钩子指向 `memory/feedback_<topic>.md` 详情文件——长期规则不会污染日常日志
- **复发检测**：同一主题在 ≥ 2 个 session 出现但还没进 MEMORY.md，skill 会标 ⚠️ 并主动提议"升级到长期偏好"
- **回顾模式**（`/memory-keeper review`）：新会话开头读最近日志 + 未完成待办 + 最近 git 提交，输出 30 行内的衔接摘要

## 安装

```bash
# 克隆到 Claude Code 的 skills 目录
git clone https://github.com/FyPerson/SKILL ~/.claude/skills/memory-keeper
```

Windows：`%USERPROFILE%\.claude\skills\memory-keeper`

skill 在首次运行时会询问你是否在 `~/.claude/worklog/` 创建工作日志根目录。想换路径用环境变量 `$CLAUDE_WORKLOG_ROOT` 覆盖。

## 使用

```
/memory-keeper                  # 沉淀当前会话（默认）
/memory-keeper review           # 读最近日志 + git log + 待办，输出衔接摘要
/memory-keeper <project-name>   # 沉淀到指定项目子目录
```

接口面就这些。

## 两个反直觉的设计

两条都来自真实"翻车"——LLM"贴心"地把整份草稿一次性甩出来。两条都**故意设计成反 LLM 反射**：

### 1. 启动时复述约束

任何 `/memory-keeper` 调用的第一句输出，必须是 LLM 用自己的话把协议复述给你：

> "本次 /memory-keeper 按 SKILL 走逐项确认：**每次只展示一条 + 等用户回复后再下一条 + 不预生成全部草稿到对话 + 不给'全部按建议'类逃生口**。"

少这一句就是违反。你可以打断让它重来。

**为什么有效**：system 消息容易被扫过，但在对话流里复述出来的约束，下一轮无法忽略。

### 2. 输出前的三问自检

任何草稿条目即将发给你之前，模型自问三个问题：

1. 我即将输出的这条消息**只包含 1 条**吗？
2. 末尾**有没有"全部按建议"类逃生口**？
3. 我是否在用**列清单风格**组织输出（多分类标题一次铺开）？

任一答 yes → 重写，只输出第 1 条，末尾问"先看这条吗？"

**为什么有效**：LLM 对"分析类任务列清单"有强训练偏差。三问闸是最便宜的干预，能在反射动作上线之前拦住。

## 存储结构

```
~/.claude/worklog/
├── INDEX.md                              # 全局索引（子目录映射表 + 跨项目偏好）
├── <project-a>/
│   ├── INDEX.md                          # 项目级索引（会话日志链接）
│   ├── MEMORY.md                         # 长期偏好/状态的索引（不写规则正文）
│   ├── session_20260522.md               # 当日会话日志（CAR 结构）
│   └── memory/
│       ├── feedback_<topic>.md           # 长期偏好详情（Why + How to apply）
│       └── <topic>.md                    # 长期项目状态详情
└── <project-b>/...
```

**分层铁律**：`INDEX.md` 和 `MEMORY.md` 是 *索引文件*。它们只放一行钩子，链接到 `memory/` 里的详情文件。规则正文绝不出现在索引里。改一个规则 = 改一个详情子文件，所有索引自动保持同步。

## 提取的内容

5 类信息，CAR 结构：

| 分类 | 确认方式 | 备注 |
|---|---|---|
| 关键决策 | 逐条 | 若有放弃备选，写出原因 |
| 个人偏好 | 逐条 + 长期/一次性 | 长期/一次性的区分是防 MEMORY.md 腐烂的关键 |
| 踩坑记录 | 逐条 | 现象 + 根因 + 验证过的解法 |
| 技术实现 | 整体 | 代码记录、文件路径 |
| 里程碑/待办 | 整体 | 已完成 vs P1/P2 未结 |

## 复发检测

每次沉淀时，skill 扫描最近 3 个 session + INDEX.md 关键词。如果新草稿里某主题匹配到 ≥ 2 个历史 session 但**没在 MEMORY.md 出现过**，确认环节会给你 ⚠️ 标记 + "升级为长期偏好？"的询问。

这是跨周防止"我之前不是说过这事"重复的核心机制。

## 回顾模式

新会话开头跑 `/memory-keeper review`，它会读：

- 推断出的主项目最近 session 日志
- 辅项目按关键词命中的 session（跨项目复发）
- 主项目关联代码目录的 `git log --oneline -10`
- MEMORY.md 未完成待办

…然后输出 ≤ 30 行的衔接摘要：上次会话总结 / 关键决策 Top 3 / 辅目录命中 / 里程碑 / 未完成待办 / 新提交 / 升级建议 / 建议下一步。

**回顾模式不写任何文件。** 纯读。

## 非目标

- 不接受"帮我记一下"/"先到这里"等自然语言自动触发。仅手动调用。
- 不替代 Claude Code 原生 `/memory`。是另一条独立的、有立场的工作流。
- 不静默创建目录、不重写索引结构。模糊就问。

## 协议

MIT
