---
name: agent-delegation
description: "TOP-LEVEL SESSION ONLY — do not invoke from a subagent spawned via the Task/Agent tool; if you are such a subagent, suggest this in your report instead and let the parent session decide. Use when the top-level session (talking directly with the user) is about to hand an implementation or research task to a locally installed CLI agent — GitHub Copilot CLI (`copilot`) or Google Antigravity CLI (`agy`, via `agy-bridge`) — in non-interactive mode, or needs to read/write the delegation log for a task already handed off. Defines the exact command patterns, the log folder path formula, filename convention, and log content template so this stays consistent across machines and projects."
---

# agent-delegation

Playbook for delegating implementation/research work from Claude Code to locally installed CLI agents (`copilot`, `agy`) using non-interactive, stateless calls — the token-cheapest pattern (see the "why" writeup in project `多平台代理協作方法論.md` if present; not required to use this skill).

## Hard restriction

This skill is for the **top-level session directly conversing with the user only**. If you were spawned as a subagent via the Task/Agent tool, do **not** call `copilot`/`agy` yourself even after reading this file — report back to the parent session recommending the delegation instead. This prevents uncontrolled fan-out (subagent → executor → more subagents) and runaway API cost.

## Prerequisites

Check both are installed before relying on this skill; not every machine will have both:
```bash
command -v copilot   # GitHub Copilot CLI
command -v agy-bridge # Google Antigravity CLI wrapper
```
If only one is present, use that one. If neither is present, this skill doesn't apply — do the work directly or tell the user.

## Delegation commands

```bash
copilot -p "<task spec>" --allow-all-tools
agy-bridge --type search|code|analysis|review|implement -- "<task spec>"
```

`agy-bridge --type search` defaults to a fast Gemini Flash model (good for web research); `code`/`implement`/`analysis`/`review` default to a Gemini Pro model (good for larger implementation/analysis work). See `agy-bridge --types` and `copilot help config` on the local machine for the current model list — do not assume model names are stable across installs or time.

## Log folder path

```
~/.claude/delegation-logs/<sanitized-cwd>/
```

`<sanitized-cwd>` = the current project's working directory with every `/` replaced by `-` (same formula Claude Code's own per-project memory folder already uses under `~/.claude/projects/`). Compute it, don't guess:
```bash
slug=$(pwd | tr '/' '-')
mkdir -p ~/.claude/delegation-logs/"$slug"
```
This lives under `~/.claude/`, outside any git repo, so it needs no per-project `.gitignore` entry and survives `/tmp` cleanup / reboots.

## Log filename

```
<YYYYMMDD-HHMMSS>_<session-id-first-8-chars>_<executor>-<model>_<task-slug>.md
```

- Timestamp first so the folder lists chronologically.
- Session id: first 8 chars of the current Claude Code session UUID — visible in the scratchpad path handed to you at session start (e.g. `/tmp/claude-1000/<slug>/c1ad8a37-.../scratchpad` → `c1ad8a37`).
- `executor`: `copilot` or `agy`.
- `model`: the actual model name used for that call.
- `task-slug`: short kebab-case description.

Example: `20260812-161200_c1ad8a37_copilot-gpt-5.4_fix-login-race-condition.md`

If dispatching multiple tasks to the same executor+model within the same second (parallel/background calls), prefix the task-slug with an incrementing index (`01-fix-x`, `02-fix-y`) to avoid collisions.

## What to put in the delegation prompt

Before invoking the executor, compute the full log path and hand it the path plus this exact template — do not let the executor invent its own filename or format:

```markdown
# Delegation Log

- session: <full Claude Code session id>
- delegated_at: <ISO timestamp>
- executor: copilot | agy
- model: <model name>
- command: `<exact non-interactive command invoked>`
- status: success | partial | failed

## Task
<one paragraph restating what was asked>

## Steps taken
1. ...

## Files read
- path/to/file (why)

## Files modified/created
- path/to/file (what changed)

## Verification
<what the executor itself checked — tests run, output. If nothing was verified, say so explicitly rather than omitting the section.>

## Follow-ups / open issues
- anything the executor flagged as needing a human/main-session decision
```

Tell the executor to write **only the summary** to that log path on completion — not raw stdout, not its chain-of-thought. Log files that echo raw output defeat the point (the next read-back would re-inject the same noise into the top-level session's context).

Also include in the prompt: the concrete acceptance criteria (test command or observable behavior) and what NOT to do, so the executor doesn't need to ask clarifying questions back.

## When the top-level session should read a log

1. **Right after delegating** — verify before trusting; cross-check the log against `git diff` / test output rather than taking the executor's in-conversation summary at face value.
2. **Before delegating a follow-up on the same task** — read prior log(s), fold the relevant parts into the next prompt. This substitutes for the executor having no session memory of its own; do not switch to a stateful/interactive delegation mode just to preserve context.
3. **When post-delegation verification fails** — read the log to see what the executor believed it did, to locate the gap.
4. **When the user asks about past delegated work, or a new session resumes prior work** — `ls`/glob the project's log folder (`~/.claude/delegation-logs/<sanitized-cwd>/`) for relevant entries.

Do **not** proactively scan the whole log folder every turn, and do not read logs unrelated to the current task — that reintroduces the context cost this whole pattern exists to avoid.
