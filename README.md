# agent-delegation

Claude Code plugin: delegate implementation/research tasks from a **top-level** Claude Code session to locally installed CLI agents, using non-interactive, stateless calls (no PTY simulation, no MCP wrapper) — the cheapest-token integration pattern. Background/rationale: [`docs/多平台代理協作方法論.md`](docs/多平台代理協作方法論.md).

## Supported agent frameworks

Currently two — support depends on what's installed on the machine (check both before assuming either is available):

| Executor | CLI | Invocation | Notes |
|---|---|---|---|
| `agy` | Google Antigravity CLI, via the `agy-bridge` wrapper | `agy-bridge --type search\|code\|analysis\|review\|implement -- "<prompt>"` | `search` defaults to a fast Gemini Flash model; `code`/`implement`/`analysis`/`review` default to a Gemini Pro model |
| `copilot` | GitHub Copilot CLI | `copilot -p "<prompt>" --allow-all-tools` | Model selectable via `--model`; see `copilot help config` for the current list |

Adding a new executor (e.g. `aider`) means: confirm it has a non-interactive/scriptable mode, add its invocation pattern to `skills/agent-delegation/SKILL.md`, and it inherits the same log convention below — no other changes needed.

## Log convention

**Folder** (fixed formula, no per-project setup needed):
```
~/.claude/delegation-logs/<sanitized-cwd>/
```
`<sanitized-cwd>` = the delegating project's working directory with every `/` replaced by `-` (same convention Claude Code's own per-project memory folder uses under `~/.claude/projects/`). Created on first use with `mkdir -p`.

**Filename**:
```
<YYYYMMDD-HHMMSS>_<session-id-first-8-chars>_<executor>-<model>_<task-slug>.md
```
Timestamp first so the folder lists chronologically; session id ties the log back to which Claude Code session issued it; `executor` is `agy` or `copilot`; `model` is the actual model used; `task-slug` is a short kebab-case description. On same-second collisions (parallel/background dispatch), prefix the slug with an incrementing index (`01-`, `02-`).

**Content**: a fixed template (session/executor/model/command/status header, then Task / Steps taken / Files read / Files modified / Verification / Follow-ups). Full template and the read/write rules (when the top-level session should read a log, what must go in the delegation prompt) are in [`skills/agent-delegation/SKILL.md`](skills/agent-delegation/SKILL.md) — that's the file Claude Code actually loads, kept as the single source of truth; this README summarizes it.

## Install

```bash
claude plugin marketplace add LawrenceCHH/agent-delegation
claude plugin install agent-delegation@agent-delegation
```

Update later with:
```bash
claude plugin update agent-delegation
```

## Hard restriction

This is for the top-level session directly conversing with the user only. Subagents spawned via the Task/Agent tool must not invoke this delegation themselves — they should report back recommending it, and let the parent session decide. See the skill's `description` frontmatter and body for the enforced wording.
