# Fuse

**Fuse helps coding agents produce better results by making multiple agents work in parallel, then taking the strongest parts of each result.**

Instead of asking one model or agent to solve a task once, Fuse sends the same task to several agents such as Claude Code, Codex, Gemini CLI, OpenCode, Qwen Code, or other local coding agents. Each worker agent solves the task independently in its own isolated git worktree. The orchestrator compares their patches and reports, identifies the best ideas, optionally asks for peer review, and integrates the strongest final result.

This is useful because different agents often have different strengths. One may produce the cleanest architecture, another may write better tests, and another may catch edge cases. Fuse turns those differences into an advantage.

```text
                 User task
                    |
                    v
            Orchestrator agent
                    |
       +------------+------------+
       |            |            |
       v            v            v
   Claude worker  Codex worker  Gemini worker
   private tree   private tree   private tree
       |            |            |
       +------------+------------+
                    |
                    v
          Judge + final integrator
                    |
                    v
             Best final result
```

The most important rule:

```text
1 selected worker agent = 1 private git worktree
```

Workers must never share one worktree, and workers must never directly modify the original directory where `/fuse` was invoked.

---

## Installation

Clone this repository, then copy the agent-specific Fuse files into the project where you want to use Fuse.

```bash
git clone https://github.com/taekwonv/fuse-skill.git
cd fuse-skill
```

### Claude Code

Install the Claude skill from `claude/fuse`.

Project-level:

```bash
mkdir -p /path/to/your-project/.claude/skills
cp -R claude/fuse /path/to/your-project/.claude/skills/fuse
```

User-level:

```bash
mkdir -p ~/.claude/skills
cp -R claude/fuse ~/.claude/skills/fuse
```

### Codex

Install the Codex skill from `codex/fuse`.

Project-level:

```bash
mkdir -p /path/to/your-project/.agents/skills
cp -R codex/fuse /path/to/your-project/.agents/skills/fuse
```

User-level:

```bash
mkdir -p ~/.agents/skills
cp -R codex/fuse ~/.agents/skills/fuse
```

### Gemini CLI

Install the Gemini command and context files from `gemini`.

Project-level:

```bash
mkdir -p /path/to/your-project/.gemini/commands
cp gemini/fuse.toml /path/to/your-project/.gemini/commands/fuse.toml
```

Then add the Fuse instructions to your project's `GEMINI.md`.

If the project does not have a `GEMINI.md` yet:

```bash
cp gemini/GEMINI.md /path/to/your-project/GEMINI.md
```

If the project already has a `GEMINI.md`:

```bash
cat gemini/GEMINI.md >> /path/to/your-project/GEMINI.md
```

Reload Gemini commands if needed:

```text
/commands reload
```

### Other agents

Copy the appropriate Markdown instruction file into the place where that agent stores reusable skills, commands, prompts, or project instructions. The agent must be able to read the Fuse instructions when the user invokes `/fuse`.

---

## Repository Layout

```text
fuse-skill/
  README.md
  LICENSE
  universal_SKILL.md

  claude/
    fuse/
      SKILL.md

  codex/
    fuse/
      SKILL.md

  gemini/
    GEMINI.md
    fuse.toml
```

Use the `claude/`, `codex/`, or `gemini/` directory depending on the agent you want to install Fuse into.

---

## Quick Start

Run Fuse from the root of the repository you want the agents to work on.

```text
/fuse agents: claude,codex task: Fix the booking status race condition.
```

```text
/fuse agents: claude,gemini tests: ./gradlew test task: Refactor notification dispatch.
```

```text
/fuse agents: claude,codex,gemini mode: review task: Review this PR for bugs, missing tests, and privacy risks.
```

```text
/fuse agents: claude,codex,gemini judge: claude integrator: codex peer-review: true tests: npm test task: Improve auth middleware.
```

Some Codex environments may expose skills as `$fuse` instead of `/fuse`:

```text
$fuse agents: claude,codex,gemini task: Fix the auth middleware race condition.
```

---

## How Fuse Works

### 1. The current agent becomes the orchestrator

The agent that receives `/fuse` or `$fuse` is responsible for the run.

```text
/fuse invoked in Claude  -> Claude is the orchestrator
/fuse invoked in Gemini  -> Gemini is the orchestrator
$fuse invoked in Codex   -> Codex is the orchestrator
```

The orchestrator parses the request, identifies the repo root, coordinates workers, judges outputs, and produces the final result.

### 2. Each worker creates its own private worktree

If the user runs Fuse from:

```text
/home/user/my_code
```

then workers should create separate sibling worktrees such as:

```text
/home/user/my_code-fuse-worktrees/<run-id>/claude
/home/user/my_code-fuse-worktrees/<run-id>/codex
/home/user/my_code-fuse-worktrees/<run-id>/gemini
```

A worker must only modify its own worktree.

### 3. Each worker produces a report

Each worker should write:

```text
FUSE_REPORT.md
```

The report should include:

- Summary
- Design decisions
- Changed files
- Tests run
- Risks
- Review notes

### 4. The orchestrator compares the results

The judge evaluates each output using:

- Correctness
- Simplicity
- Fit with existing architecture
- Test coverage
- Regression risk
- Maintainability
- Security and privacy risk

By default, the orchestrator is the judge. You can override this:

```text
/fuse agents: claude,codex,gemini judge: claude task: ...
```

### 5. Optional peer review

With `peer-review: true`, agents can review each other before the final judge decision.

```text
/fuse agents: claude,codex,gemini peer-review: true task: ...
```

Peer review is optional. The final decision should still come from one judge.

### 6. The integrator produces the final result

By default, the orchestrator integrates the final result. You can assign a different integrator:

```text
/fuse agents: claude,codex,gemini judge: claude integrator: codex task: ...
```

The integrator should not blindly merge patches. It should choose the strongest base implementation and selectively borrow the best ideas from the other workers.

---

## Supported Arguments

| Argument | Meaning |
|---|---|
| `agents:` | Comma-separated worker agents. Example: `claude,codex,gemini`. |
| `task:` | The task to perform. Required. |
| `base:` | Base branch or ref. Example: `main`, `origin/main`, `HEAD~1`. |
| `tests:` | Test, lint, or build command workers should run. |
| `mode:` | `implement`, `review`, `plan`, or `debug`. Default: `implement`. |
| `judge:` | Optional final judge agent. Default: orchestrator. |
| `integrator:` | Optional final implementation agent. Default: orchestrator. |
| `peer-review:` | `true` or `false`. Default: `false`. |
| `final:` | `recommend`, `patch`, or `worktree`. |
| `copy-docs:` | Optional list of untracked instruction files to copy into each worktree. |
| `max-workers:` | Optional concurrency cap. |

Model hints are allowed:

```text
/fuse agents: claude@sonnet,codex@gpt-5.5,gemini@pro task: ...
```

They are only hints. Fuse should not claim that a specific model was used unless the selected CLI can actually enforce that model.

---

## Native Agent Instruction Files

Fuse does not replace each agent's normal project instruction system.

Use the native files your agents already understand:

```text
AGENTS.md   # Codex project instructions
CLAUDE.md   # Claude Code project instructions
GEMINI.md   # Gemini CLI project instructions, if your setup uses it
```

Fuse must not require `.fuse/agents/*.md`.

If these files are tracked by git, they naturally appear in each worker worktree. If they are untracked, pass them explicitly:

```text
/fuse agents: claude,codex copy-docs: AGENTS.md,CLAUDE.md task: ...
```

---

## Safety Notes

- Run Fuse only inside a git repository.
- Review final diffs before merging.
- Do not let multiple workers edit the same worktree.
- Do not assume all agent CLIs support the same model-selection flags.
- Do not claim a worker used a model unless the CLI confirmed it.
- Be careful with private code when routing prompts through remote model providers.

---

## License

MIT License. See [LICENSE](./LICENSE) for details.
