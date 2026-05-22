# memory-keeper

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
