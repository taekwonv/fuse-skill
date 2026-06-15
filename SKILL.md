# Fuse — Universal Multi-Agent Worktree Fusion Skill

Use this Markdown instruction as a portable `/fuse` skill for any coding agent, including Codex, Claude Code, Gemini CLI, OpenCode, Qwen Code, or other local CLI agents.

This file is intentionally **not Codex-specific**, **not Claude-specific**, and **not Gemini-specific**. It defines a shared workflow that any orchestrator agent can follow.

---

## 0. Purpose

`/fuse` runs the same user task through multiple independent coding agents, lets each agent solve it in its own private git worktree, then has the orchestrator compare the results, identify the best solution, optionally synthesize improvements from multiple results, and present a final recommendation or final implementation.

The key rule is:

```text
1 selected worker agent = 1 private git worktree
```

Claude, Codex, Gemini, Qwen, OpenCode, or any other worker must **never share one worktree**.

---

## 1. Invocation Syntax

The host tool may expose this as `/fuse`, `$fuse`, or a normal prompt. Treat all of the following as equivalent.

```text
/fuse agents: claude,codex task: Fix the booking status race condition.
```

```text
/fuse agents: claude,gemini tests: ./gradlew test task: Refactor notification dispatch.
```

```text
/fuse agents: claude@sonnet,codex@gpt-5.5,gemini@pro base: main task: Improve auth middleware.
```

```text
/fuse agents: codex,claude,gemini mode: review task: Review current PR for bugs and missing tests.
```

### Supported fields

| Field | Meaning |
|---|---|
| `agents:` | Comma-separated worker agents to run. Examples: `claude,codex`, `claude,gemini`, `codex,claude,gemini,qwen`. |
| `task:` | The actual task to perform. Required. |
| `base:` | Base branch/ref. Default: current branch or `main`, depending on repo state. |
| `tests:` | Test/lint/build command workers should run. Optional. |
| `mode:` | `implement`, `review`, `plan`, or `debug`. Default: `implement`. |
| `copy-docs:` | Optional list of untracked instruction files to copy mechanically into each worktree, such as `AGENTS.md,CLAUDE.md`. Use only when those files are not tracked by git. |
| `final:` | `recommend`, `patch`, or `worktree`. Default: `worktree` for implementation tasks and `recommend` for planning/review tasks. |
| `max-workers:` | Optional cap on concurrently running workers. |

### Model hints

The syntax `agent@model` is a **hint**, not a guarantee.

Examples:

```text
claude@sonnet
codex@gpt-5.5
gemini@pro
qwen@coder
```

If a CLI agent does not support setting that model from the command line, do not claim that the model was used. Use the agent's available default or report that the requested model hint could not be enforced.

---

## 2. Non-Negotiable Rules

1. The current agent receiving `/fuse` is the **orchestrator**.
2. The orchestrator must detect the current repository root from the directory where `/fuse` was invoked.
3. All worker agents must operate on the same source project, but each worker must create and use its **own private git worktree**.
4. Worker agents must not modify the original invocation directory directly.
5. Worker agents must not share a worktree with other worker agents.
6. The orchestrator must not assume agent instruction files live under `.fuse/agents` or any other Fuse-specific folder.
7. Agent instruction files are native to each agent:
   - Codex may read `AGENTS.md` according to its own rules.
   - Claude Code may read `CLAUDE.md` according to its own rules.
   - Gemini CLI may read its own configured instruction files according to its own rules.
   - Other agents should follow their own native discovery/configuration rules.
8. Fuse must not parse, merge, rewrite, or reinterpret native agent instruction files.
9. If instruction files are tracked by git, they appear in worker worktrees naturally.
10. If instruction files are untracked, copy them only when the user explicitly provides `copy-docs:`.
11. The original repo directory is reserved for orchestration, judging, and final user-approved integration.
12. Any final implementation should be produced in a dedicated final worktree unless the user explicitly asks to apply it to the original directory.

---

## 3. Directory Model

Assume the user runs `/fuse` from a project directory like:

```text
/home/user/my_code
```

The orchestrator must resolve:

```text
FUSE_ROOT = git rev-parse --show-toplevel
```

For a run id like:

```text
fuse-20260615-123000
```

workers should create sibling worktrees outside the root working tree:

```text
/home/user/my_code-fuse-worktrees/fuse-20260615-123000/claude
/home/user/my_code-fuse-worktrees/fuse-20260615-123000/codex
/home/user/my_code-fuse-worktrees/fuse-20260615-123000/gemini
```

Recommended branch names:

```text
fuse/fuse-20260615-123000/claude
fuse/fuse-20260615-123000/codex
fuse/fuse-20260615-123000/gemini
fuse/fuse-20260615-123000/final
```

The exact path may vary by operating system, but the isolation rule may not vary.

---

## 4. Orchestrator Workflow

When receiving `/fuse`, the orchestrator must follow this sequence.

### Step 1 — Parse the request

Extract:

```text
agents
model hints
task
base branch/ref
tests
mode
final behavior
copy-docs list
```

If `agents:` is missing, ask for it unless a local default is clearly configured.

### Step 2 — Resolve repo root

Run or conceptually perform:

```bash
git rev-parse --show-toplevel
```

If the current directory is not inside a git repo, stop and tell the user `/fuse` requires a git repo.

### Step 3 — Create run metadata

Inside the original repo, create a lightweight run folder only for orchestration metadata:

```text
.fuse/runs/<run-id>/
  TASK.md
  WORKERS.md
  JUDGE_REPORT.md
  FINAL_REPORT.md
```

This folder is not for agent-specific instruction files. It is only for run artifacts.

### Step 4 — Build a canonical task charter

Create `.fuse/runs/<run-id>/TASK.md` containing:

```markdown
# Fuse Task

## User Task
<original user task>

## Mode
implement | review | plan | debug

## Base Ref
<base ref>

## Worker Agents
<agent list with model hints if any>

## Test Command
<test command or "not specified">

## Acceptance Criteria
- Correctness
- Minimality
- Consistency with existing architecture
- Tests added/updated when relevant
- No unrelated changes
- Clear report

## Mandatory Worker Output
Each worker must create `FUSE_REPORT.md` in its own worktree.
```

### Step 5 — Launch each worker agent independently

For each selected worker, start a separate agent session from `FUSE_ROOT` and give it the worker prompt in section 5.

The worker, not another worker, is responsible for creating its own private worktree and moving into it before modifying files.

The orchestrator may provide the recommended worktree path and branch name, but must not instruct two agents to use the same path.

### Step 6 — Wait for worker reports

Each worker must finish with:

```text
<worker-worktree>/FUSE_REPORT.md
```

and a diff relative to the base ref.

If a worker fails, record the failure and continue with the other workers unless all workers fail.

### Step 7 — Collect diffs and reports

For each worker worktree, collect:

```bash
git diff <base-ref>...HEAD
```

and the worker's `FUSE_REPORT.md`.

Store summaries under:

```text
.fuse/runs/<run-id>/results/<agent>/
  FUSE_REPORT.md
  diff.patch
  status.txt
```

### Step 8 — Judge results

Evaluate each implementation using the rubric in section 7.

Produce:

```text
.fuse/runs/<run-id>/JUDGE_REPORT.md
```

### Step 9 — Produce final result

Depending on `final:`:

- `recommend`: present the ranking, best result, and merge instructions.
- `worktree`: create or ask a final integrator to create a separate final worktree and synthesize the best solution.
- `patch`: produce a final patch for user review without applying it to the original directory.

Default:

```text
mode: implement -> final: worktree
mode: plan/review -> final: recommend
```

### Step 10 — Report to user

The final response must include:

```text
- Which agents ran
- Where each worker worktree is
- Which worker won and why
- What was borrowed from other workers
- Tests run and results
- Final worktree/branch or recommended patch
- Risks and next steps
```

---

## 5. Worker Prompt Template

The orchestrator should send each worker a prompt like this.

```text
You are a worker agent in a Fuse multi-agent workflow.

Important:
- You are currently being launched from the original repository root: <FUSE_ROOT>.
- Do not modify this original directory directly.
- You must create and use your own private git worktree before editing files.
- No other worker agent may use your worktree.
- Do not assume shared state with other workers.
- Read your native instruction files using your own normal rules. For example, if you are Claude Code, follow your CLAUDE.md behavior; if you are Codex, follow your AGENTS.md behavior. Fuse does not provide separate agent-md files.

Your worker identity:
- Agent: <agent-name>
- Model hint: <model-hint-or-none>

Run information:
- Run ID: <run-id>
- Base ref: <base-ref>
- Recommended branch: fuse/<run-id>/<agent-name>
- Recommended worktree path: <worktree-parent>/<run-id>/<agent-name>

First, create your private worktree:

  git worktree add -b fuse/<run-id>/<agent-name> <recommended-worktree-path> <base-ref>

Then move into that worktree and perform the task there.

Task:
<task>

Mode:
<implement | review | plan | debug>

Tests to run:
<tests or "not specified">

Required behavior:
1. Work only inside your private worktree.
2. Keep the solution minimal and consistent with the existing project.
3. Add or update tests if relevant.
4. Run the specified tests if provided. If not provided, infer the smallest relevant test command if safe.
5. Do not commit unless explicitly instructed.
6. Create FUSE_REPORT.md at the root of your private worktree.

FUSE_REPORT.md must include:

# Fuse Worker Report

## Agent
<agent name and model if known>

## Summary
What you changed.

## Design Decisions
Why you chose this approach.

## Changed Files
List changed files and why.

## Tests Run
Exact commands and results.

## Risks
Potential regressions, uncertainties, or unresolved concerns.

## Review Notes
What the orchestrator should inspect carefully.
```

---

## 6. Worker Shell Examples

These examples are illustrative only. The exact command to start each CLI agent depends on the user's local setup.

### Claude worker

```bash
cd <FUSE_ROOT>
claude "<worker prompt>"
```

### Codex worker

```bash
cd <FUSE_ROOT>
codex "<worker prompt>"
```

### Gemini worker

```bash
cd <FUSE_ROOT>
gemini "<worker prompt>"
```

### OpenCode or Qwen worker

```bash
cd <FUSE_ROOT>
opencode "<worker prompt>"
qwen "<worker prompt>"
```

If a local CLI requires a different non-interactive flag, use the user's known command template. Do not invent successful execution if the command fails.

---

## 7. Judge Rubric

Score each worker from 0 to 10 on:

1. Correctness
2. Simplicity
3. Fit with existing architecture
4. Test coverage
5. Maintainability
6. Security/privacy risk
7. Regression risk
8. Diff cleanliness
9. Completeness against acceptance criteria

Then produce:

```markdown
# Fuse Judge Report

## Ranking
1. <agent> — reason
2. <agent> — reason
3. <agent> — reason

## Score Table
| Agent | Correctness | Simplicity | Architecture Fit | Tests | Risk | Maintainability | Total |
|---|---:|---:|---:|---:|---:|---:|---:|

## Consensus
What most agents agreed on.

## Contradictions
Where implementations made different assumptions.

## Unique Strengths
Best idea from each worker.

## Blind Spots
What none of the workers handled well.

## Recommended Final Strategy
Which implementation should be used as the base and what should be borrowed from others.
```

---

## 8. Final Integrator Workflow

For implementation tasks, prefer a final worktree:

```bash
git worktree add -b fuse/<run-id>/final <worktree-parent>/<run-id>/final <base-ref>
```

The integrator may be the orchestrator itself or a selected agent. It must:

1. Use the judge's recommended winner as the base strategy.
2. Avoid blindly merging patches.
3. Reconstruct the final implementation cleanly in the final worktree.
4. Borrow only specific improvements from other workers.
5. Keep the final diff smaller and cleaner than an indiscriminate merge.
6. Add or improve tests using the strongest test coverage found across workers.
7. Run tests.
8. Create `FINAL_FUSE_REPORT.md`.

`FINAL_FUSE_REPORT.md` must include:

```markdown
# Final Fuse Report

## Final Strategy
Which worker was used as the base and why.

## Borrowed Ideas
What was borrowed from other workers.

## Rejected Ideas
What was intentionally not used and why.

## Changed Files
Final changed files.

## Tests Run
Exact commands and results.

## Remaining Risks
Anything the user should verify.

## Merge Recommendation
Whether this final worktree is ready to merge.
```

---

## 9. Native Agent Instruction Files

Fuse must respect native agent conventions and must not replace them.

Examples:

```text
/home/user/my_code/AGENTS.md      -> Codex native guidance
/home/user/my_code/CLAUDE.md      -> Claude Code native guidance
/home/user/my_code/GEMINI.md      -> Gemini/native guidance if supported by the user's setup
```

Rules:

1. Do not move these files into `.fuse`.
2. Do not require `.fuse/agents/*.md`.
3. Do not combine these files into the worker prompt.
4. If they are tracked by git, let git worktree include them naturally.
5. If they are untracked and the user provides `copy-docs:`, copy them mechanically without interpreting them.
6. If an agent fails to read its native files, report that as an agent/runtime issue, not a Fuse issue.

---

## 10. Optional Local Configuration

A project may optionally define command templates, but Fuse must not require them.

Example file:

```text
.fuse/config.json
```

Example content:

```json
{
  "commands": {
    "claude": "claude",
    "codex": "codex",
    "gemini": "gemini",
    "opencode": "opencode",
    "qwen": "qwen"
  },
  "worktree_parent_suffix": "-fuse-worktrees",
  "default_base": "main",
  "default_final": "worktree"
}
```

This configuration is only for launching agents and choosing paths. It must not define or override native agent instruction files.

---

## 11. Failure Handling

### Worker cannot start

Record:

```text
agent name
command attempted
error message
```

Continue with other workers.

### Worker does not create a worktree

Mark the worker failed. Do not accept changes made in the original directory.

### Worker modifies the original directory

Stop that worker, warn the user, and do not use those changes unless the user explicitly approves after review.

### Worker creates a dirty but useful result

Collect the diff and report, but mark risk as high.

### Tests fail

Do not hide failures. Include exact command and failure summary.

### Merge conflicts

Do not auto-resolve by blindly combining patches. Use a final worktree and reconstruct the solution intentionally.

---

## 12. Final User Response Template

Use this structure when reporting the result.

```markdown
## Fuse Result

Run ID: `<run-id>`
Base: `<base-ref>`
Mode: `<mode>`

### Agents Run
| Agent | Worktree | Status | Tests |
|---|---|---|---|

### Winner
`<agent>` won because ...

### What Was Borrowed
- From `<agent>`: ...
- From `<agent>`: ...

### Final Output
Final worktree: `<path>`
Final branch: `<branch>`

### Tests
`<command>` -> pass/fail

### Risks
- ...

### Recommended Next Step
Review the final diff, then merge or open a PR from `<branch>`.
```

---

## 13. Minimal `/fuse` Execution Summary

When in doubt, do the simplest correct version:

```text
1. Parse agents and task.
2. Resolve current git repo root.
3. For each agent, launch it from the repo root with a prompt requiring it to create its own private worktree.
4. Wait for FUSE_REPORT.md and diffs.
5. Compare results with the judge rubric.
6. Create a final integration worktree.
7. Synthesize the best solution.
8. Run tests.
9. Report winner, final worktree, tests, and risks.
```

Never share one worktree across multiple worker agents.
