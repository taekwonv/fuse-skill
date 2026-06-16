# Fuse

**Fuse helps coding agents produce better results by making multiple agents work in parallel, then taking the strongest parts of each result.**

Instead of asking one model or agent to solve a task once, Fuse sends the same task to several agents such as Claude Code, Codex, Gemini CLI, OpenCode, Qwen Code, or other local coding agents. Each worker agent solves the task independently in its own isolated git worktree. The orchestrator compares their patches and reports, identifies the best ideas, optionally asks for peer review, and integrates the strongest final result.

This is useful because different agents often have different strengths. One may produce the cleanest architecture, another may write better tests, and another may catch edge cases. Fuse turns those differences into an advantage.

```text
User
  |
  |  /fuse agents: claude,codex,gemini task: ...
  v
Orchestrator agent
  |
  +-- Claude worker  -> private git worktree -> patch + FUSE_REPORT.md
  +-- Codex worker   -> private git worktree -> patch + FUSE_REPORT.md
  +-- Gemini worker  -> private git worktree -> patch + FUSE_REPORT.md
  |
  v
Judge / optional peer review
  |
  v
Integrator
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

The source skill file in this repo is:

```text
skill.md
```

Install it where your agent expects reusable skills, commands, or prompts.

### Claude Code

Project-level:

```bash
mkdir -p .claude/skills/fuse
cp skill.md .claude/skills/fuse/SKILL.md
```

User-level:

```bash
mkdir -p ~/.claude/skills/fuse
cp skill.md ~/.claude/skills/fuse/SKILL.md
```

### Codex

Project-level:

```bash
mkdir -p .agents/skills/fuse
cp skill.md .agents/skills/fuse/SKILL.md
```

User-level:

```bash
mkdir -p ~/.agents/skills/fuse
cp skill.md ~/.agents/skills/fuse/SKILL.md
```

### Gemini CLI

Create a Gemini custom command that reads `skill.md`:

```bash
mkdir -p .gemini/commands
cat > .gemini/commands/fuse.toml <<'EOF2'
description = "Run the Fuse multi-agent worktree workflow."
prompt = """
Read and follow ./skill.md.

User arguments:
{{args}}

You are the orchestrator for this /fuse run. Parse the arguments, resolve the git repository root, ensure each selected worker agent creates its own private git worktree, collect reports and diffs, judge the results, and produce the requested final output.
"""
EOF2
```

Reload commands if needed:

```text
/commands reload
```

### Other agents

Put `skill.md` wherever that agent stores reusable skills, commands, prompts, or project instructions. The agent must be able to read the file and follow it when the user invokes `/fuse`.

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

## Copy/Paste Adaptation Prompts

Use these prompts if you want Claude, Codex, or Gemini to convert `skill.md` into the exact native format for that agent.

### Adapt for Claude Code

Paste this into Claude Code from the repository root:

```text
You are adapting ./skill.md into a Claude Code skill.

Source file:
- ./skill.md

Target file:
- ./.claude/skills/fuse/SKILL.md

Requirements:
1. Create a valid Claude Code skill named "fuse".
2. Add any Claude Code skill metadata/frontmatter required by the current Claude Code skill format.
3. Preserve the Fuse workflow exactly:
   - the current Claude session is the orchestrator when /fuse is invoked;
   - each selected worker agent must create its own private git worktree;
   - workers must never share a worktree;
   - workers must never directly modify the original invocation directory;
   - native files such as CLAUDE.md, AGENTS.md, and GEMINI.md remain native to each agent;
   - do not introduce a .fuse/agents convention.
4. Keep the source filename references as skill.md.
5. Keep the skill instruction-only unless a script is absolutely required.
6. Do not claim support for agent CLIs that are not installed.
7. After writing the file, summarize exactly what changed.
```

### Adapt for Codex

Paste this into Codex from the repository root:

```text
You are adapting ./skill.md into a Codex skill.

Source file:
- ./skill.md

Target file:
- ./.agents/skills/fuse/SKILL.md

Requirements:
1. Create a valid Codex skill named "fuse".
2. Add Codex-compatible skill metadata/frontmatter, including name and description if required by the current Codex skill format.
3. Preserve the Fuse workflow exactly:
   - the current Codex session is the orchestrator when $fuse or /fuse is invoked;
   - each selected worker agent must create its own private git worktree;
   - workers must never share a worktree;
   - workers must never directly modify the original invocation directory;
   - native files such as AGENTS.md, CLAUDE.md, and GEMINI.md remain native to each agent;
   - do not introduce a .fuse/agents convention.
4. Use $fuse examples for Codex CLI/IDE, while noting that some environments may expose it as /fuse.
5. Keep the source filename references as skill.md.
6. Keep the skill instruction-only unless a script is absolutely required.
7. Do not claim that model hints are enforceable unless the selected CLI actually supports them.
8. After writing the file, summarize exactly what changed.
```

### Adapt for Gemini CLI

Paste this into Gemini CLI from the repository root:

```text
You are adapting ./skill.md into a Gemini CLI custom command.

Source file:
- ./skill.md

Target file:
- ./.gemini/commands/fuse.toml

Requirements:
1. Create a valid Gemini CLI custom command named /fuse.
2. Use TOML format.
3. Include a concise description.
4. The command prompt must instruct Gemini to read and follow ./skill.md.
5. The command prompt must pass user-provided command arguments with {{args}}.
6. Preserve the Fuse workflow exactly:
   - the current Gemini session is the orchestrator when /fuse is invoked;
   - each selected worker agent must create its own private git worktree;
   - workers must never share a worktree;
   - workers must never directly modify the original invocation directory;
   - native files such as GEMINI.md, AGENTS.md, and CLAUDE.md remain native to each agent;
   - do not introduce a .fuse/agents convention.
7. Keep the source filename references as skill.md.
8. Do not claim that Gemini can enforce another agent's model selection unless that external CLI supports it.
9. After writing the file, summarize exactly what changed.
```

---

## Recommended Repository Layout

```text
fuse/
  README.md
  skill.md
  LICENSE
```

Optional examples:

```text
examples/
  claude/SKILL.md
  codex/SKILL.md
  gemini/fuse.toml
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
