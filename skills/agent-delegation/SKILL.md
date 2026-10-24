---
name: agent-delegation
description: "TOP-LEVEL SESSION ONLY — do not invoke from a subagent spawned via the Task/Agent tool; if you are such a subagent, suggest this in your report instead and let the parent session decide. Use when the top-level session (talking directly with the user) is about to hand an implementation or research task to a locally installed CLI agent — GitHub Copilot CLI (`copilot`) or Google Antigravity CLI (`agy`, directly or via the optional `agy-bridge` wrapper) — in non-interactive mode, or needs to read/write the delegation log for a task already handed off. Defines the exact command patterns, the log folder path formula, filename convention, and log content template so this stays consistent across machines and projects."
---

# agent-delegation

Playbook for delegating implementation/research work from Claude Code to locally installed CLI agents (`copilot`, `agy`) using non-interactive, stateless calls — the token-cheapest pattern (see the "why" writeup in project `多平台代理協作方法論.md` if present; not required to use this skill).

## Hard restriction

This skill is for the **top-level session directly conversing with the user only**. If you were spawned as a subagent via the Task/Agent tool, do **not** call `copilot`/`agy` yourself even after reading this file — report back to the parent session recommending the delegation instead. This prevents uncontrolled fan-out (subagent → executor → more subagents) and runaway API cost.

## Prerequisites

Check which are installed before relying on this skill; not every machine will have all:
```bash
command -v copilot    # GitHub Copilot CLI
command -v agy        # Google Antigravity CLI (required for agy delegation)
command -v agy-bridge # optional wrapper; if absent, call `agy -p` directly
```
If only one executor is present, use that one. If none is present, this skill doesn't apply — do the work directly or tell the user.

## Two modes: Quick vs Full

Pick the mode by task type — the log machinery below is only worth its cost when work changes files or must be resumed later.

| Mode | Use for | Log? | Verification |
|---|---|---|---|
| **Quick** | Web search, read-only analysis, Q&A | No — stdout comes straight back into context | Require a source URL per conclusion in the prompt; spot-check the key ones yourself (WebFetch/WebSearch). A conclusion with no source is unverified. |
| **Full** | Implementation, refactors, anything that modifies files or spans sessions | Yes — template, path and filename rules below | `git diff` / test output, then read the log |

### Quick mode command (agy)

```bash
agy -p "<question>. Cite a source URL for every conclusion; say so explicitly if unsure." \
    --model <newest Gemini Flash> --output-format text
```
No `--dangerously-skip-permissions`: in print mode agy denies tool calls lacking an allow rule and exits nonzero, which is the safe default for read-only work. Use `--output-format json` when you need to parse the result. Check the exit code and that stdout is non-empty.

Known issue: agy 1.0.x `-p` could hang or print nothing when stdout was redirected/piped from a subprocess ([antigravity-cli#318](https://github.com/google-antigravity/antigravity-cli/issues/318)). Verified working through a pipe on 1.4.6 (2026-10-24); on older versions, run `agy update` first.

## Delegation commands (Full mode)

```bash
copilot -p "<task spec>" --allow-all-tools --add-dir ~/.claude/delegation-logs --model <default: newest Claude Sonnet>
agy -p "<task spec>" --add-dir ~/.claude/delegation-logs --model <default: newest Gemini Flash>   # or: agy-bridge --type code|analysis|review|implement --model <...> -- "<task spec>"
```
Full-mode agy needs write access (log file, edits): grant it with an allow rule, or `--dangerously-skip-permissions` only inside a sandbox/disposable environment.

**`copilot` has its own path sandbox, separate from `--allow-all-tools`.** `--allow-all-tools` only waives per-tool confirmation prompts; it does not grant filesystem access outside the current project directory. Since the log folder (`~/.claude/delegation-logs/`) is deliberately outside the project by design, writes there fail with "Permission denied" unless the directory is explicitly added with `--add-dir ~/.claude/delegation-logs` (as in the command above). Confirmed by direct test on 2026-08-12: the first attempt without `--add-dir` failed to create the log file; adding it fixed it.

### Default model selection

Unless the task clearly needs something else (see carve-outs below), default to:

| Executor | Default | Why |
|---|---|---|
| `agy` / `agy-bridge` | Newest available Gemini **Flash** model | Cheap/fast; matches this pattern's whole point of keeping delegated calls low-cost |
| `copilot` | Newest available Claude **Sonnet** model | Mid-tier cost/quality balance — not the cheapest (Haiku), not the priciest (Opus) |

Model names and version numbers are **not stable** — they change per machine, per org policy, and over time (already confirmed: `gpt-5-mini` was rejected as unauthorized on one org's Copilot while `claude-haiku-4.5` worked; agy's own model list also updates independently). Resolve "newest" at delegation time, don't hardcode a version string from this doc:

```bash
# newest Gemini Flash for agy — pick the highest-numbered gemini-*-flash-* entry
# (Quick mode: -medium or -low is fine; e.g. gemini-3.8-flash-medium)
agy models | grep flash

# newest Claude Sonnet for copilot — pick the highest-numbered claude-sonnet-* entry
copilot help config | grep -A30 '`model`:' | grep claude-sonnet
```
(As of 2026-08-12 on this machine these resolved to `gemini-3.6-flash-high` and `claude-sonnet-5` — treat as an example, not a value to copy forward.)

`agy-bridge --type search` already resolves to a Flash-tier model by its own internal default (see `agy-bridge --types`), so an explicit `--model` is usually redundant there; `code`/`implement`/`analysis`/`review` internally default to a Pro-tier model instead — pass `--model <newest-flash>` explicitly to keep those on the cheap default too, unless the task is large/complex enough that the extra reasoning of Pro is worth the cost (e.g. a sprawling multi-file implementation, a genuinely hard bug) — in that case use Pro deliberately and note why in the delegation log's Task section.

Also: `--model <name>` must be a model actually authorized under the local Copilot org policy — the full model list in `copilot help config` is not a guarantee of availability. If an explicit `--model` is rejected, fall back to `--model auto` or check what a bare test call resolves to before retrying.

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
