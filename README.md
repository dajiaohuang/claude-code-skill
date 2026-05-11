# Claude Code Skill — OpenClaw AgentSkill

Route code tasks through the [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI instead of using built-in file manipulation tools.

## Overview

When a user requests code-related work (writing, refactoring, reviewing, debugging, scaffolding new projects), the AI assistant delegates to Claude Code rather than using built-in write/edit/exec tools directly.

## Installation

### Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- [OpenClaw](https://openclaw.ai) or an AI assistant platform compatible with the AgentSkill format

### Install Claude Code

```powershell
# Windows (PowerShell) - requires Node.js >= 18
npm install -g @anthropic-ai/claude-code

# Verify installation
claude --version

# Login to Anthropic
claude login
```

```bash
# macOS / Linux
npm install -g @anthropic-ai/claude-code
claude --version
claude login
```

### Install This Skill

```powershell
# Windows (PowerShell)
git clone https://github.com/dajiaohuang/claude-code-skill.git $env:USERPROFILE\.openclaw\skills\claude-code-skill
```

```bash
# macOS / Linux
git clone https://github.com/dajiaohuang/claude-code-skill.git ~/.openclaw/skills/claude-code-skill
```

Or manually copy the `SKILL.md` and `README.md` files into your OpenClaw skills directory.

## Quick Start

Once the skill is installed and matched, the assistant automatically routes coding tasks to Claude Code:

```
User: "Add a logout button to the navbar"
→ Assistant spawns Claude Code to implement the change

User: "Review my recent changes for security issues"
→ Assistant spawns Claude Code to run a security review

User: "Refactor UserService to be less than 200 lines"
→ Assistant spawns Claude Code for the refactoring plan + execution
```

## Usage Examples

### New Project Setup

```
User: "Create a React + TypeScript todo app"

Assistant:
  1. Creates D:\claw\todo-app\
  2. Runs claude init inside the project
  3. Runs claude -p "plan: Create a React + TypeScript todo app with..."
  4. After plan approval, Claude Code scaffolds the full project
```

### Code Review

```
User: "Review src/auth/login.ts"

Assistant:
  → claude -p "Review src/auth/login.ts for security, correctness, and style"
  → Returns structured feedback with severity levels
```

### Debugging

```
User: "Login returns 500 after the recent session changes"

Assistant:
  → claude -p "debug: POST /api/login returns 500.
     Reproduce: curl -X POST /api/login -d '{\"user\":\"test\",\"pass\":\"test\"}'.
     Recent changes: src/middleware/session.ts (git diff HEAD~3..HEAD).
     Error: [pasted error log]"
```

### Multi-File Refactoring

```
User: "Extract shared validation logic from all controllers"

Assistant:
  1. claude -p "plan: Extract shared validation into a middleware/validator layer"
  2. After plan approval, spawns subagents for each controller refactor
  3. Runs integration verification
```

### Batch Parallel Tasks

```
User: "Add unit tests for all service files"

Assistant:
  → Spawns subagent A: add tests for UserService
  → Spawns subagent B: add tests for OrderService
  → Spawns subagent C: add tests for PaymentService
  → Collects results and reports summary
```

## Subagent Mode

Subagent spawning is the **preferred pattern** for parallel or non-blocking Claude Code invocations.

### When to Use Subagents

| Scenario | Subagents |
|---|---|
| Independent file changes | One subagent per file/set |
| Batch code reviews | One subagent per module |
| Multi-module refactoring | One subagent per module |
| Long-running task | Background subagent, main session stays responsive |

### How It Works

```
Main Session
  ├─ spawn subagent-1 → claude -p "task A"
  ├─ spawn subagent-2 → claude -p "task B"
  ├─ spawn subagent-3 → claude -p "task C"
  │   (all run in parallel)
  └─ await all → aggregate results → verify
```

### Constraints

- Subagent tasks must be **mutually independent** — no two subagents should write to the same file
- Number of concurrent subagents should not exceed CPU core count
- Subagents inherit the same no-timeout policy — they run to natural completion

## Key Design Decisions

1. **Claude Code first, built-in tools as fallback** — Only use built-in file tools when Claude Code errors out
2. **No timeout** — Claude Code runs to completion; progress is reported every 10 minutes
3. **Maximal functionality by default** — When Claude Code asks clarifying questions, choose the fullest-featured option unless it exceeds local machine capacity
4. **Subagent parallelism** — Independent tasks run concurrently via subagents
5. **Windows-first paths** — Project directories live under `D:\claw\` on Windows

## Project Workflow

### New Projects

1. Create directory under `D:\claw\<project-name>` (Windows) or `~/claw/<project-name>` (macOS/Linux)
2. Run `claude init` to generate CLAUDE.md and initialize the project
3. Enter plan mode: `claude -p "plan: <requirements>"`
4. Review and approve the plan
5. Claude Code executes, reporting progress every 10 minutes

### Existing Projects

| Requirement State | Approach |
|---|---|
| No detailed requirements | Claude explores codebase → lists improvements → user decides |
| Detailed requirements | Pass directly to Claude Code for execution |
| Already `claude init`'d | Never use built-in file tools; all changes go through Claude Code |

## Invocation Reference

| Scenario | Command | Notes |
|---|---|---|
| Interactive TUI | `claude` | Full terminal UI, requires `pty: true` |
| One-shot task | `claude -p "prompt"` | Non-interactive, exits on completion |
| Plan mode | `claude -p "plan: ..."` | Generates plan, user approves, then implements |
| Code review | `claude -p "review <target>"` | Add dimensions: security, perf, style |
| Debugging | `claude -p "debug: <issue>"` | Include repro steps + error logs |
| Batch/parallel | spawn subagent × N | Independent tasks only |

## Prompt Writing Tips

### Effective Prompts

- Describe the **goal**, not the steps: "Refactor UserService so each method is under 20 lines"
- Provide **context**: relevant files, past decisions, constraints
- Specify **output format** when needed: "List 5 improvements, one per line"
- For reviews, specify **dimensions**: security, performance, readability, architecture

### What to Avoid

- Vague instructions: "look at my code"
- Over-constraining: "only use X library / Y pattern" unless truly required
- Mixing unrelated tasks in a single prompt
- Omitting error logs or reproduction steps in debug requests

## FAQ

### Q: Why use Claude Code instead of built-in tools?

Claude Code provides a richer execution environment — it can run tests, read the entire codebase, iterate on solutions, and produce more polished results. Built-in tools are a safety net for when Claude Code is unavailable.

### Q: When should I use `claude` (interactive) vs `claude -p` (non-interactive)?

Use `claude -p` for well-defined, single-step tasks. Use `claude` (interactive TUI) for exploratory tasks, multi-step refactors, or when you need to review intermediate results and adjust direction.

### Q: What happens if Claude Code errors?

The assistant falls back to built-in file tools and notifies you. See the [Error Handling](#error-handling--fallback) section in SKILL.md for details.

### Q: How do I handle very large tasks?

Break them into independent sub-tasks and spawn subagents for parallel execution. For dependent tasks, run them sequentially with verification between steps.

### Q: Does this work on Windows?

Yes. The skill is designed with Windows in mind — project paths default to `D:\claw\`, and PowerShell-specific notes are included. See the [Windows Environment](#windows-environment) section in SKILL.md.

### Q: How do I know progress during long runs?

Claude Code reports progress every 10 minutes — files changed, current status, estimated remaining work. You receive the final result when it completes.

## Windows-Specific Notes

- Default project root: `D:\claw\`
- PowerShell 5.1 does not support `&&` chaining — use `; if ($?) { }` instead
- For UTF-8 output in PowerShell: `[Console]::OutputEncoding = [Text.Encoding]::UTF8`
- Windows Terminal is recommended over legacy conhost
- CRLF/LF line-ending issues are handled automatically by `claude init` + `.gitattributes`

## License

MIT
