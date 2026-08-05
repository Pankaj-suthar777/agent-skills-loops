# loopV1

First version of the multi-agent engineering loop. Agent-agnostic — usable with opencode, Claude Code, Codex, or any agent CLI that supports markdown agent definitions.

## Contents

| Path | Purpose |
|---|---|
| `agents/orchestrator.md` | Main agent. Classifies tasks, delegates, runs verification/review gates, manages repair budget, maintains loop state. Never edits application code. |
| `agents/worker.md` | Subagent. Implements code. Edits allowed; destructive git commands denied. |
| `agents/tester.md` | Subagent. Objective verification only, never edits. `TARGETED` (fast checks) and `FINAL_GATE` (regression gate) modes. |
| `agents/reviewer.md` | Subagent. Independent read-only senior review. CRITICAL / IMPORTANT / MINOR findings. |
| `opencode.json` | Reference config example (default agent `orchestrator`). Adapt to your agent CLI. |
| `package.json` | Optional `@opencode-ai/plugin` dependency (opencode only). |
| `loop-state/current.json` | Live loop state (task, tier, repair budget, phases). |
| `loop-history.md` | Append-only log of completed loops. |

## Setup

```sh
cp -r agents/ .agent/          # or .opencode/ for opencode, .claude/ for Claude Code
# keep a loop-state/ + loop-history.md next to the agents
```

## Quick start

1. Start your agent CLI in the target project.
2. Give the orchestrator a plain-language task.
3. It routes the task: TRIVIAL (direct), STANDARD (budget 5, reserve 1), COMPLEX (budget 7, reserve 2).
4. Loop: `@worker` implements → `@tester` TARGETED → `@reviewer` reviews → `@tester` FINAL_GATE → done.

## Customization

- Models: frontmatter of each agent file (currently `opencode-go/deepseek-v4-flash`).
- Toolchains: bash allow-lists in each file (Flutter, npm, pytest pre-allowed).
- Budgets/steps: `orchestrator.md` tier table and `steps:` limits.

See the root `README.md` for the full workflow details.
