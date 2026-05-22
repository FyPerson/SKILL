---
name: memory-keeper
description: Manually-triggered session memory distillation skill for Claude Code. Extracts decisions, preferences, gotchas, todos, and milestones from the current conversation into structured worklog files, with per-item confirmation and long-term-vs-temporary distinction. Use only when the user explicitly runs `/memory-keeper`, `/memory-keeper review`, or `/memory-keeper <subdir>`. 手动触发的会话记忆沉淀技能。仅在用户显式运行 `/memory-keeper` 时使用。
disable-model-invocation: false
compatibility: Designed for Claude Code. Storage root resolves in this order — $ARGUMENTS as subdir → $CLAUDE_WORKLOG_ROOT env var → ~/.claude/worklog/ (default).
---

# Memory Keeper

A disciplined session-memory keeper for Claude Code. Only invoke this skill when the user explicitly runs `/memory-keeper`. Do not auto-trigger on phrases like "remember this", "save it", "let's stop here" — that is a different reflex and breaks the contract.

一份给 Claude Code 用的会话记忆沉淀工具。仅当用户显式输入 `/memory-keeper` 时启动；自然语言"帮我记一下""先到这里"等不触发。

---

## Non-goals / 非目标

- Do not auto-trigger from natural language. / 不根据"帮我记住""先到这里"等自然语言自行介入
- Do not present this skill as Claude Code's built-in `/memory`. / 不把本 skill 描述成 Claude Code 系统自带 memory
- Do not silently create a new `INDEX.md` or `worklog/` tree when the path is ambiguous. / 路径不明时不静默创建

---

## Modes / 模式

| Invocation | Mode | Purpose |
|---|---|---|
| `/memory-keeper` | **Distill** (default) | End-of-session: extract decisions/gotchas/milestones into a session log |
| `/memory-keeper review` | **Review** | Start-of-session: read recent logs + todos + git log, output a handoff summary |
| `/memory-keeper <subdir>` | Distill (specific project) | Same as default, but write to the specified project subdir |

### Mode resolution / 模式判定

1. `$ARGUMENTS` empty or does not contain `review` → Distill (default behavior)
2. `$ARGUMENTS` starts with `review` → Review mode
3. `$ARGUMENTS` is any other value → Treated as a subdir name, Distill mode

---

## Storage layout / 存储结构

All worklogs live under a single root, organized by project subdirectory:

```
<worklog-root>/
├── INDEX.md                      ← Global index (cross-project preferences + subdir map)
├── <project-a>/
│   ├── INDEX.md                  ← Project-level index
│   ├── MEMORY.md                 ← Long-term preferences/state index (links to detail files)
│   ├── session_YYYYMMDD.md       ← Daily session logs
│   └── memory/
│       ├── feedback_<topic>.md   ← Long-term preference detail
│       └── <topic>.md            ← Long-term project state detail
├── <project-b>/
│   └── ...
```

### Root resolution / 根路径解析

Resolution order:

1. If `$ARGUMENTS` is a path that exists, treat it as the project subdir directly.
2. Else if env var `$CLAUDE_WORKLOG_ROOT` is set, use it as the root.
3. Else use the default: `~/.claude/worklog/` (resolves to `$HOME/.claude/worklog/` or `%USERPROFILE%\.claude\worklog\` on Windows).

**First-run bootstrap**: If the resolved root does not exist, do **not** silently create it. Ask the user:

> 📍 **First-time setup required**
>
> Worklog root not found at `<resolved path>`.
>
> A. Create it here (recommended)
> B. Pick a different path
> C. Cancel
>
> Which?

After the user picks A or provides a path, create the root, then create a starter `INDEX.md` (template below), then a starter project subdir based on the current working directory's basename.

### INDEX.md template (root-level)

```markdown
# Global Worklog Index

## Subdir map (source of truth for /memory-keeper path resolution)

| Subdir | Display name | Type | Linked code dir | Status |
|---|---|---|---|---|
| [<subdir>](<subdir>/) | <display name> | <code project / topic / experiment> | <path or "none"> | <active / archived> |

## Workspace → worklog subdir mapping

`/memory-keeper` with no args picks a default subdir based on the current working directory:

| Current workspace | Default worklog subdir |
|---|---|
| `<path>` | `<subdir>` |
| Other | Ask user to confirm |

## Cross-project preferences

(Empty until /memory-keeper promotes the first long-term preference)

## Maintenance rules

- When creating a new subdir: add one row to the subdir map AND add one row to the workspace mapping
- When archiving a subdir: change its status to "archived" (do not delete the row)
- The skill MUST prompt the user to register the subdir when writing to an unregistered target
```

---

## ⭐ Path inference + explicit confirmation (mandatory on every run)

In **both** distill and review mode, **the skill's first user-visible output MUST be a path confirmation block**.

### Single-context (high confidence)

> 📍 **Target for this [distill / review]**
> - Main: [`<worklog-root>/<subdir>/`](<worklog-root>/<subdir>/) — <display name>
>
> **Reasoning**: <one sentence, e.g., "~80% of this session was code changes in <project>">
> **Confidence**: high
>
> Proceed? Or redirect to another subdir?

### Mixed-context (medium/low confidence)

> 📍 **Target for this [distill / review]** (mixed content — dual-write suggested)
> - Main: [path] — display name (~X%)
> - Aux: [path] — display name (~Y%, suggest adding a pointer)
>
> **Reasoning**: <explanation>
>
> A. Dual-write   B. Main only   C. Aux only   D. Adjust split

### Review mode aux scan

> 📍 **Review scan range**
> - Main: [path] — display name (latest session)
> - Aux: [path] — display name (keywords [a, b, c], estimated N matches)
>
> Proceed? Or adjust keywords / aux subdirs?

### Inference logic

1. **`$ARGUMENTS` wins**: if the user passes a subdir name, use it (confidence = high)
2. **Workspace mapping fallback**: look up the current cwd in INDEX.md's "Workspace → worklog subdir mapping"
3. **Session content scan** (key step): scan the last ~20 turns of conversation for key nouns, compare against each subdir's INDEX.md scope/keywords. If the session topic does not match the default subdir, confidence drops to medium/low and the mixed-context format is mandatory.
4. **Unregistered subdir reminder**: when the target subdir is not in INDEX.md's subdir map, proactively ask "Register it in the subdir map?"

### Display name requirement

- Every path display **must include a display name**: `[path] — display name`
- Display names come from `<worklog-root>/INDEX.md`'s subdir map
- If the subdir is not registered: show `[path] — ⚠️ unregistered, please add display name`

Throughout this document, relative paths are relative to the resolved project subdir.

---

## What to extract / 提取的信息

After triggering, extract 5 categories from the current session, using **CAR structure** (Context, Action, Result):

- **Key decisions** / 关键决策: tech choices, trade-offs, semantic rules. If a rejected alternative is clear, write the reason.
- **Personal preferences** / 个人偏好: collaboration habits, code style, formatting rules, interaction dislikes. Must distinguish **long-term** vs **one-off**.
- **Gotchas** / 踩坑记录: error symptom, root cause, verified fix.
- **Todos** / 待办事项: only explicit follow-ups, blockers, or P1/P2 items.
- **Milestones** / 里程碑: only things already agreed on, completed, or delivered.

---

## ⚠️ Pre-execution self-check (mandatory every run — do not skip)

**Known failure mode**: this skill's "per-item confirmation + single-item display" is **counter-intuitive for LLMs** — training data biases toward "list all items at once" for analysis tasks. The skill has been violated multiple times by dumping the full draft in one message (most recent: a session that dumped 14 items in one block with an "approve all" escape hatch).

**The fix**: lift the constraint from system messages (easy to skim past) into the conversation stream (impossible to ignore).

### Distill mode startup (the first conversational output MUST be this restatement)

> "Running /memory-keeper per the skill's per-item protocol: **one item shown per message + wait for user reply before next + no full draft pre-dumped to chat + no 'approve all' escape hatch**. First I'll read INDEX.md / run recurrence detection / generate the internal draft, then show only Key Decision #1."

Failure to restate = violation. The user may interrupt and ask you to restart if the first output is missing this line.

### Reverse self-check after draft generation

When the draft is ready and about to be presented, **before output**, ask three questions:

1. Does the message I'm about to send contain **exactly 1 draft item** (with recommendation + reasoning)? Or ≥ 2?
2. Does the message end with an **"approve all" / batch-approve escape hatch**?
3. Am I organizing the output in a **list-dump style** (multiple category headers + bullets all at once)?

Any "yes" (≥ 2 items / has escape hatch / list-dump style) → **stop and rewrite**, showing only item #1 and ending with "Look at this one first?"

### Mid-distill reminder (when switching categories)

When moving from one category to the next (e.g., finishing Key Decisions and starting Personal Preferences), **the first output of the new category MUST restate**:

> "Entering [category name], still per-item: single display + wait + no batching."

This guards against drift in the middle of long sessions.

### Special handling for batch-confirm categories

Three categories use **batch display + batch confirmation** (not per-item): technical implementation, milestones, todos. **But they must still each be a separate message** — not merged into one combined message — because the judgment for each category is independent.

---

## Execution flow / 执行流程

1. Parse and validate the memory root.
2. Read `INDEX.md`.
3. Compute today's session file path: `session_YYYYMMDD.md`.
4. If today's session file already exists, read it first, then generate the draft; apply the "same-day re-append rule" (below).
5. **Recurrence pre-detection**: scan the most recent 3 session files (not just the latest) + the INDEX.md index lines, extracting tool names / workflow names / concept keywords already in use. For each decision/preference/gotcha in the new draft, check against this keyword set and annotate "appeared in X past sessions". This gives the user grounds to decide "promote to long-term preference / MEMORY.md".
6. Generate a draft containing at minimum:
   - Key decisions
   - Personal preferences
   - Gotchas
   - Technical implementation / code records
   - Completed milestones
   - Todo updates
   - Conflict warnings (if any)
   - Same-day duplicates (if any)
   - **Recurrence flags**: items appearing in ≥ 2 historical sessions get a ⚠️ marker plus a "suggest promotion" hint at confirmation time
7. In the draft, explicitly flag:
   - Conflicts with `INDEX.md`
   - Duplicates or updates against today's existing session file
   - Recurring themes (≥ 2 occurrences)
8. **Per-item confirmation**: after the draft is generated, walk through with the user category by category. Show **one item per message**, each with **"my recommendation + reasoning for both sides"** (see the single-item format section). Wait for "keep / drop / edit" before showing the next. Order:
   - Key decisions (per-item, with rec + reasoning)
   - Personal preferences (per-item, with rec + reasoning, AND ask "long-term / one-off" with reasoning for both)
   - Gotchas (per-item, with rec + reasoning)
   - Technical implementation / code records (batch confirm)
   - Completed milestones (batch confirm)
   - Todo updates (batch confirm)
   - Conflict warnings (per-item, ask "override?", with rec + reasoning)
   - **Long-term promotion offer** (when the user picks "long-term" on a preference): immediately ask "Also write to MEMORY.md? Draft is `feedback_<topic>.md` titled <X>"
9. Once everything is confirmed, prepare the final write payload:
   - Append block for today's session file
   - Updated `INDEX.md` content
   - **(If any long-term promotion confirmed)** New/updated `memory/feedback_<topic>.md` detail file + MEMORY.md index line
10. Execute writes in a fixed order for consistency:
    - First append `session_YYYYMMDD.md` (frontmatter on first write — see template; same-day re-append per the rule below):
      ```markdown
      ---
      name: YYYY-MM-DD Session Log
      description: One-line summary of this session's core content
      type: project
      ---
      ```
    - Then update `INDEX.md`
    - **(If promotion confirmed)** Write `memory/feedback_<topic>.md` + insert an index line in the appropriate `MEMORY.md` section ("Constraints" / "Collaboration patterns" / topic group)
11. If any step fails, tell the user clearly **what was written and what was not**. Do not claim full success.

### Same-day re-append rule / 同日二次写入规则

1. **Do not duplicate frontmatter**: keep the original (name/description/type).
2. **Whether to update description**: default no; only update if the user explicitly says "today took a major turn, update the description".
3. **Append position**: at the file's end, blank line, then a `---` separator, then a timestamped heading:
   ```markdown
   ---

   ## Appended at HH:MM

   (new content)
   ```
4. **Same-day duplicate detection**: before write, diff the new draft against existing content. Match by title keyword (first ~10 chars of decision/gotcha title). On match, prompt at confirmation time: "merge / override / skip".

---

## Review mode / 回顾模式（/memory-keeper review）

### Goal

At the start of a new session, quickly stitch context from prior work so the user can decide "continue last thread" or "start new topic".

### Data sources (priority order)

1. **Main subdir's most recent session log**: latest `session_YYYYMMDD.md` (read fully)
2. **Aux subdir keyword hits** (cross-subdir scan): pull keywords from the main subdir's INDEX.md "📍 Historical related sessions" section + "Cross-subdir scan keyword library", then scan other registered subdirs' sessions for matches (read titles/descriptions + matched snippets only, not full files)
3. **INDEX.md todo section**: extract unfinished items from the "Todo priority" or "Todo updates" sections
4. **git log**: if the main subdir has a linked code project (from the subdir map's "Linked code dir" column), run `git log --oneline -10`
5. **MEMORY.md todo section**: if the main subdir has a linked auto-memory, read its MEMORY.md todo priority section

### Execution flow

1. **Show the path confirmation block** (shared with distill mode, see "Path inference" section), wait for the user to confirm main + aux scope.
2. Parse memory root + aux scan list (same rules as distill, plus user-adjusted aux subdirs).
3. Read the main subdir's `INDEX.md`, find the most recent session log filename and date.
4. Read that session log in full.
5. **Cross-subdir aux scan**: for each aux subdir, scan the most recent 5 sessions' titles/descriptions/first 30 lines by keyword. Output an "aux hit list" (filename + matched keyword + one-line description).
6. Read the last 10 git commits (if main subdir is linked to a code project).
7. Read MEMORY.md's todo section (if main subdir is linked to auto-memory).
8. **Recurrence detection (cross-subdir extended)**: read **the main subdir's last 3 sessions + matched aux sessions**, extract key decisions and gotchas, detect themes that **appear in multiple sessions but are not yet in MEMORY.md**. Match criterion: same theme (matching tool/workflow/concept keyword) in ≥ 2 distinct sessions with no MEMORY.md index. **Cross-subdir recurrence must be specially flagged** (e.g., "Leadership communication methodology has shown up 3 times in smart-data-hub").
9. Synthesize all of the above and output a **handoff summary** (template below).
10. **Review mode writes nothing.** Pure read. If there are promotion suggestions, prompt the user to handle them in the next `/memory-keeper` distill run.

### Output template

> ### 📋 Work handoff summary
>
> **Scan range**
> - Main: [path] — display name
> - Aux: [path list] (if any, else "none")
>
> **Last session** (YYYY-MM-DD, main subdir)
> - One-line summary (from session file's description)
>
> **Top 3 key decisions** (default: title + one-line Result only, CAR folded)
> 1. [title] — Result
> 2. [title] — Result
> 3. [title] — Result
>
> > Want the full Context/Action/Result for any decision? Say "expand #N".
>
> **Aux historical hits** (cross-subdir scan results, omit entire section if none)
> - [aux-subdir/session_xxx.md] — one-line topic (matched: abc)
> - [aux-subdir/session_yyy.md] — one-line topic (matched: xyz)
>
> **Completed milestones** (list)
>
> **Unfinished todos**
> - [P1] ...
> - [P2] ...
>
> **Code changes since last session** (git commits dated later than the last session)
> - commit1
> - commit2
> - (If no new commits: "no new commits"; if main subdir has no linked code project: omit the entire section)
>
> **Promotion suggestions** (recurring items not yet in MEMORY.md)
> - [Theme A] — appears in session_xxxx and session_yyyy, suggest promotion to MEMORY.md (at `feedback_<topic>.md`)
> - **[Cross-subdir theme B]** — appears in main session_xxx + aux session_yyy/session_zzz, suggest promotion (flag: cross-subdir recurrence)
> - (If none: "no promotion suggestions")
>
> **Suggested next step**
> - Continue: [highest-priority todo]
> - Or start a new topic?

### Constraints

- Review mode is **read-only**. It modifies nothing.
- It does not auto-execute todos — only surfaces them.
- If the session log is empty (no prior /memory-keeper ever run), prompt the user to run distill first.
- Keep the summary under 30 lines. Do not paste entire session logs.

---

## File maintenance rules / 文件维护规则

### Content layering / 内容分层与写入路径

| Content type | Write target | Notes |
|---|---|---|
| Session snapshot (decisions/gotchas/tech impl/milestones/todos) | `session_YYYYMMDD.md` | Archived by day, captures the judgment at the time |
| Session index (one-line summary) | `INDEX.md`'s "Session log" section | Date-ordered index file, **not a rule library** |
| **Long-term preferences / constraints** ⭐ | `MEMORY.md` index line + `memory/feedback_<topic>.md` detail | Cross-session stable rules, judgment principles |
| **Long-term project state / facts** ⭐ | `MEMORY.md` topic section + `memory/<topic>.md` detail | Cross-session stable project background |

### Layering details

- **Technical implementation details** default to today's `session_YYYYMMDD.md`.
- Only use `memory/snippets/` when the user explicitly asks for it, or when a code chunk is genuinely too large for the session log.
- **When updating `INDEX.md`, only maintain "session index lines"** — no rules, no preferences. `INDEX.md` is a date-ordered index file; polluting it breaks its index function.
- **Long-term preference / constraint write flow** (triggered when the user picks "long-term" on a preference):
  1. Create/update `memory/feedback_<topic>.md` (frontmatter + Why + How to apply sections — see existing feedback_*.md as a reference if any exist)
  2. Add an index line in the appropriate `MEMORY.md` section: `- [Title](memory/feedback_<topic>.md) — one-line hook`
  3. **Do not write the rule body directly into `MEMORY.md`** — MEMORY is an index, detail lives in subfiles
- **Long-term project state** write flow is analogous — `memory/<topic>.md` detail + `MEMORY.md` index line.
- If the target section does not exist, **ask the user which section to put it in or whether to create a new one**. Do not silently rewrite the overall structure.
- Only when the current draft is explicitly marked "completed" AND the user confirms, migrate a todo to a milestone.
- Only clean up or archive old files when the user explicitly requests it. Do not proactively offer.

---

## Constraints / 约束

- Manual trigger only. No natural-language auto-trigger.
- Never dump large code blocks directly into `INDEX.md` or `MEMORY.md` — both are index files; detail goes into subfiles.
- **`INDEX.md` is solely a "session log index"** — no rules, no preferences.
- **Long-term preferences/constraints/project state write to `MEMORY.md` index line + `memory/<topic>.md` detail.** Do not write the body to `MEMORY.md`.
- One-off needs go in the session log only, not MEMORY.
- If memory root is ambiguous, ask before writing.
- "Review" / "recap" / "look back" do not imply consent to write — still ask.
- All `memory/` modifications require user confirmation before execution.

---

## Confirmation flow detail / 确认流程说明

After draft generation, **walk through with the user item by item**, showing one item per message, **each with "my recommendation + reasoning for both sides"**, waiting for the reply before the next:

1. **Key decisions**: per-item display + rec + reasoning. User replies "keep / drop / edit".
2. **Personal preferences**: per-item display + rec + reasoning (including "long-term / one-off" reasoning for both). User replies "keep / drop / edit" + picks "long-term" or "one-off".
3. **Gotchas**: per-item display + rec + reasoning. User replies "keep / drop / edit".
4. **Technical implementation / code records**: batch display. User replies "keep all / drop X / edit Y".
5. **Completed milestones**: batch display. Same reply format.
6. **Todo updates**: batch display. Same reply format.
7. **Conflict warnings** (if any): per-item display + rec + reasoning. Ask "override?"
8. **Same-day duplicates** (if any): per-item display + rec + reasoning. Ask "merge / override / skip?"
9. **Long-term promotion offer** (triggered when a preference is marked "long-term"): ask "Also write to MEMORY.md? Draft is `feedback_<topic>.md` titled <X>"

After everything is confirmed, summarize the confirmed result, then execute writes.

### Single-item display format (important) / 单条展示格式（重要）

**Every per-item-confirm entry MUST use this format** — content + my recommendation + reasoning for both sides, so the user has grounds to choose when uncertain:

> **[Category] Item N**
>
> > [content]
> > (If recurrence-flagged: ⚠️ "appeared in X past sessions")
>
> **💡 My recommendation: [keep / long-term / override / ...]**
>
> **Reasoning for [recommended]**: [2-3 sentences, why this option]
>
> **Reasoning for [the opposite]**: [2-3 sentences, why the other option could be right; let the user pick against the recommendation]
>
> Lean which way?

**Special format for personal preferences** (needs "long-term / one-off" reasoning for both):

> **[Personal preference] Item N**
>
> > [content]
>
> **💡 My recommendation: [long-term / one-off]**
>
> **Long-term reasoning**: [why this is worth writing to MEMORY.md, cross-session stable]
>
> **One-off reasoning**: [why this might be context-specific and not worth promoting]
>
> Lean which way?

### How I (Claude) form recommendations

- **Key decisions** → default "keep", unless content is a routine op (like a casual `git commit` that doesn't need recording)
- **Personal preference long-term/one-off** → check three things:
  1. Does it depend on a specific tool/scenario (strong dependency → one-off; general → long-term)
  2. Does MEMORY.md already have a similar principle (yes → lean long-term, can merge; no → judge by generality)
  3. The user's tone ("from now on always X" → long-term; "let's do it this way this time" → one-off)
- **Gotchas** → default "keep", unless low-frequency with a clear fix path (e.g., a one-off environment config)
- **Conflict warnings** → check timestamps and scope of new vs old; if new is more precise, recommend "override"
- **Same-day duplicates** → roughly same content → recommend "merge"; new content updates the conclusion → recommend "override"; entirely different in nature → recommend "skip"

Reasoning should be brief (2-3 sentences). Don't pile on arguments. Final choice belongs to the user.

---

## Portability / 迁移备注

This skill is written for Claude Code. If you port to Codex / Agent Skills:

- Keep `name` and `description` as standard fields.
- Drop `disable-model-invocation` — host-side config controls implicit invocation.
- Rewrite explicit-invocation examples in a neutral way; don't hardcode `/memory-keeper` as universal syntax.

---

## Credits / 致谢

Distilled from a real-world long-running Claude Code workflow. The two non-obvious design choices — **(1) the startup restatement of constraints into the conversation stream, and (2) the pre-output three-question self-check** — were both born from concrete failures of "LLM dumps the full draft in one message". If you find them counter-intuitive, that is the point: they are designed to override an LLM's training-data reflex.

从一份长跑的 Claude Code 实战工作流蒸馏。两个反直觉的设计点——**(1) SKILL 启动时把约束复述到对话流里、(2) 输出前的三问自检**——都来自真实"一次铺开全部草稿"的违规踩坑。如果你觉得这两条设计很啰嗦，那正是它们的价值所在：它们专门对抗 LLM 训练数据的反射。
