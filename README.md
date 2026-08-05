# agent-skills-loops

Agent-agnostic multi-agent engineering loop. The loop coordinates specialized agents — orchestrator, worker, tester, reviewer — to implement, verify, and independently review engineering tasks until completion is objectively proven. It works with any agent CLI: opencode, Claude Code, Codex, or anything else that supports subagents and markdown agent definitions.

## Directory layout

```
loops-arch/
└── loopV1/                  # Loop configuration v1
    ├── opencode.json        # example agent-CLI config (adapt to your tool)
    ├── package.json         # plugin dependency (@opencode-ai/plugin)
    ├── agents/
    │   ├── orchestrator.md  # main agent: runs the loop, delegates, gates completion
    │   ├── worker.md        # subagent: implements code
    │   ├── tester.md        # subagent: objective verification (TARGETED / FINAL_GATE)
    │   └── reviewer.md      # subagent: independent read-only senior review
    ├── loop-state/
    │   └── current.json     # live state of the current loop run
    └── loop-history.md      # append-only log of completed loops
```

## How to use this loop

1. **Install into a project.** Copy `loops-arch/loopV1/agents/` into your project's agent directory, conventionally `.agent/` (or your tool's native location: `.opencode/` for opencode, `.claude/` for Claude Code, `.codex/` for Codex, etc.). The agent prompts and the orchestrator's permissions reference `.agent/loop-state/*.json` and `.agent/loop-history.md`, so keep that folder layout.
2. **Platform config.** Use your agent CLI's own config file — the included `opencode.json` is only a reference example (default agent = `orchestrator`, subagent depth 1). For Claude Code use `CLAUDE.md`/subagent entries; for Codex use `AGENTS.md`; for opencode copy `opencode.json`. The `package.json` dependency (`@opencode-ai/plugin`) is optional and only relevant to opencode.
3. **Start the agent in that project** and give the orchestrator a task in plain language. It classifies the task (TRIVIAL / STANDARD / COMPLEX), creates loop state, delegates implementation to `@worker`, verifies with `@tester`, reviews with `@reviewer`, and runs the final regression gate.

## Portability notes

- The `agents/*.md` files use opencode-style frontmatter (YAML header with `description`, `mode`, `model`, `permission` rules). The prompts and workflow are tool-agnostic — only the frontmatter needs adapting to your agent CLI's subagent format (e.g. Claude Code uses `CLAUDE.md` subagent slots, Codex uses `AGENTS.md` sections).
- Permission rules in the frontmatter map to your tool's permission system: worker may edit code and run tests but must be denied destructive git commands (`git push`, `git reset --hard`, `git clean`, `git checkout --`); tester and reviewer are read-only; orchestrator may only write loop state/history, never application code.
- Adjust bash allow-lists to your project's toolchain (Flutter, npm/pnpm/yarn, pytest, ... are pre-allowed as examples).

## Agents

### orchestrator (main agent)

Runs the whole loop. Responsibilities: classify task tier, investigate (via `@explore` for complex work), delegate implementation, run objective verification, obtain independent review, detect regressions, manage the repair budget, prevent repeated failed strategies, and maintain loop state. It can only edit `.agent/loop-state/*.json` and `.agent/loop-history.md` — it never edits application code itself, and has no bash access.

### worker (subagent)

Implements the smallest complete solution for the task or repair given by the orchestrator. Can edit code and run tests/lint, but is denied destructive git commands. Returns a structured report (`IMPLEMENTATION_STATUS`, `FILES_CHANGED`, `RISKS_OR_UNCERTAINTIES`, ...) instead of deciding whether the overall task is done.

### tester (subagent)

Independent verification engineer. Never edits source. Runs in two modes:

- `MODE: TARGETED` — fast, relevant checks on the changed code after implementation/repair.
- `MODE: FINAL_GATE` — broadest reasonable CI-equivalent regression gate ("did this break anything else?") before completion.

Every failure must be classified: `IMPLEMENTATION_REGRESSION`, `PRE_EXISTING`, `ENVIRONMENT`, `UNCERTAIN`, or `NONE`.

### reviewer (subagent)

Independent senior reviewer, completely read-only. Inspects the actual diff and surrounding code and reports findings by severity:

- **CRITICAL** — must be fixed (security, data loss, broken behavior, severe regression).
- **IMPORTANT** — normally must be fixed (likely real bug, meaningful edge case, architecture problem).
- **MINOR** — does not block approval; deferred unless trivially safe to fix.

Returns `STATUS: APPROVED` or `STATUS: CHANGES_REQUIRED`. Reviewer never fixes its own findings — the worker repairs, then tester and reviewer re-run.

## Task tiers & repair budget

| Tier | Examples | Workflow | Repair budget | FINAL_GATE reserve |
|---|---|---|---|---|
| TRIVIAL | copy changes, one-line fix, formatting | WORKER → lightweight verification → DONE | — | — |
| STANDARD | normal bug fixes, multi-file features | WORKER → TARGETED → REVIEWER → FINAL_GATE | 5 | 1 |
| COMPLEX | auth, payments, migrations, concurrency, unknown root cause | EXPLORE → WORKER → TARGETED → REVIEWER → FINAL_GATE | 7 | 2 |

A **repair round** is consumed whenever code must change due to a TARGETED failure, a CRITICAL/IMPORTANT reviewer finding, or a FINAL_GATE regression. The FINAL_GATE reserve guarantees the first final gate is never entered with zero repair capacity — if pre-final repairs would consume the reserve, the orchestrator stops and reports to the user.

TRIVIAL disqualifiers: anything touching auth, payments, schemas/migrations, security-sensitive or shared/CI/build config, concurrency, or exported interfaces is at least STANDARD.

## The loop at a glance

1. **Classify** the task tier and create/update `.agent/loop-state/current.json`.
2. **Understand** — inspect the repo; invoke `@explore` for complex or unclear root causes.
3. **Decompose** — prefer a single `@worker`; parallel workers only for genuinely independent work.
4. **Implement** — `@worker` makes the smallest complete change.
5. **TARGETED test** — `@tester` runs fast relevant checks; failures consume a repair round.
6. **Review** — `@reviewer` approves or sends findings back (findings → worker repair → retest → re-review).
7. **FINAL_GATE** — `@tester` runs the broadest reasonable regression gate.
8. **Completion** — only when TARGETED = PASS AND REVIEWER = APPROVED AND FINAL_GATE = PASS (or gate failures are clearly PRE_EXISTING / ENVIRONMENT).
9. **Close out** — append a concise entry to `.agent/loop-history.md`, then clear `current.json`.

## Failed-hypothesis rule

The orchestrator tracks failed hypotheses in loop state. If essentially the same hypothesis fails twice, it is never tried a third time — instead `@explore` is invoked again for fresh root-cause investigation requiring new evidence.

## Loop state & history

- `.agent/loop-state/current.json` — small JSON snapshot: task, tier, phase, `repair_round`, `repair_budget`, `final_gate_reserve`, `final_gate_attempted`, `failed_hypotheses`, `phase_history`. Workflow metadata only — never store secrets, credentials, .env contents, or large outputs.
- `.agent/loop-history.md` — append-only log; each entry records task, tier, repair rounds used, failed hypotheses, TARGETED/REVIEWER/FINAL_GATE results, skipped checks, pre-existing failures, and outcome.

## Customization notes

- **Models** — each agent declares its own model in frontmatter (e.g. `model: opencode-go/deepseek-v4-flash`). Replace with your provider's model IDs, or let the host tool pick the model.
- **Permissions** — agent permissions are defined in each file's frontmatter (read-only for tester/reviewer, code-edit for worker, state-only for orchestrator). Adjust to your tool's permission syntax.
- **Budgets** — tier repair budgets and step limits (`steps:`) are tunable in `orchestrator.md` and the subagent files.
