---
name: claude-code
description: Call Claude Code CLI for coding tasks — write, refactor, review, debug, and scaffold new projects. Use when the user asks for code creation, modification, review, or debugging. Fall back to built-in tools only if Claude Code errors out.
---

# Claude Code — Invocation Guide

## Table of Contents

- [Core Principles](#core-principles)
- [Quick Reference](#quick-reference)
- [New Project Workflow](#new-project-workflow)
- [Updating Existing Projects](#updating-existing-projects)
- [Invocation Methods](#invocation-methods)
- [Subagent Concurrency](#subagent-concurrency)
- [Prompt Writing Guide](#prompt-writing-guide)
- [Code Review](#code-review)
- [Debugging Workflow](#debugging-workflow)
- [Multi-File Refactoring](#multi-file-refactoring)
- [Timeout & Progress Strategy](#timeout--progress-strategy)
- [Error Handling & Fallback](#error-handling--fallback)
- [Performance Tuning](#performance-tuning)
- [Interactive vs Non-Interactive](#interactive-vs-non-interactive)
- [Windows Environment](#windows-environment)

---

## Core Principles

**All code-related work goes through the Claude Code CLI**, including: writing, modifying, refactoring, reviewing, debugging, and scaffolding projects. Do not use built-in tools (write / edit / exec) to directly manipulate code files.

Fall back to built-in tools **only when Claude Code errors or is unavailable**.

## Quick Reference

| Scenario | Command / Method | Notes |
|---|---|---|
| Interactive TUI | `claude` | Requires `pty: true` |
| One-shot task | `claude -p "prompt"` | Non-interactive, exits on completion |
| Plan mode | `claude -p "plan: requirements..."` | Generates plan, user approves, then implements |
| Code review | `claude -p "review <target>"` | See [Code Review](#code-review) |
| Debugging | `claude -p "debug: <issue>"` | See [Debugging Workflow](#debugging-workflow) |
| Batch parallel tasks | spawn subagent × N | See [Subagent Concurrency](#subagent-concurrency) |
| Already-init'd project | Always use `claude` / `claude -p` | Never use built-in tools on initialized projects |
| Fallback | Built-in tools | Only when Claude Code errors |

## New Project Workflow

1. **Create directory** — under `D:\claw\` on Windows (or `~/claw/` on macOS/Linux)
2. **`claude init`** — enter the project directory and run init to generate CLAUDE.md
3. **Plan Mode** — use `claude -p "plan: <requirements>"` to let Claude generate a plan for approval before implementation
4. **Decision defaults** — when Claude asks clarifying questions (feature selection / UI / database / architecture), default to the **most feature-complete** option; downgrade only if it exceeds local machine capacity
5. **Verification** — after completion, ask Claude to run tests, builds, or start services to ensure deliverables are functional

## Updating Existing Projects

| Requirement State | Approach |
|---|---|
| **No detailed requirements** | Let Claude explore the codebase → list improvements + plan → return for user decision |
| **Detailed requirements** | Pass directly to Claude for execution |
| **Already `claude init`'d project** | Never use built-in file tools; all changes go through Claude Code |

## Invocation Methods

### Interactive TUI

```bash
claude
```

- Use: exploratory tasks, complex multi-turn refactoring, situations requiring mid-course adjustments
- Requires: `pty: true` (terminal emulation)
- Advantage: user can intervene and adjust direction in real time

### Non-Interactive One-Shot

```bash
claude -p "prompt"
```

- Use: clearly defined, bounded single tasks
- Advantage: runs to completion and exits, no terminal held

### Plan Mode

```bash
claude -p "plan: <requirement description>"
```

Or in interactive mode:

```
/plan
```

### Other Common Flags

| Flag | Purpose |
|---|---|
| `--output-format json` | Structured output |
| `--model` | Specify model (e.g., `haiku`, `sonnet`, `opus`) |
| `--max-turns` | Cap the number of turns |

## Subagent Concurrency

**Preferred pattern** — spawn multiple Claude Code instances in parallel via subagents to avoid blocking the main session.

### When to Use

- Multiple independent files to modify simultaneously
- Batch code reviews
- Parallel execution of independent tasks (e.g., frontend + backend changes at once)
- Module-by-module refactoring in large projects

### Pattern

```
Main session → spawn subagent A (claude -p "task A")
             → spawn subagent B (claude -p "task B")
             → spawn subagent C (claude -p "task C")
             → await all → aggregate results
```

### Constraints

- Subagent tasks must be **mutually independent** (no shared file writes)
- Main session aggregates and runs integration verification after all complete
- Subagents inherit the same no-timeout policy — they run to natural completion

## Prompt Writing Guide

### Effective Prompts

- Describe the **goal**, not the steps: "Refactor UserService so no method exceeds 20 lines" beats "Split UserService's login method"
- Provide necessary **context**: relevant files, existing design decisions, constraints
- Specify **output format** when appropriate: "List 5 optimization points, one per line"
- For reviews, name the **dimensions**: security, performance, readability, architectural consistency

### What to Avoid

- Vague instructions: "look at my code"
- Over-constraining: "only use X library / Y pattern" unless truly necessary
- Cramming too many unrelated tasks into one prompt

### Decision Principles

When Claude Code raises clarifying questions during execution:
1. **Maximal functionality** — choose the option that delivers the most
2. **Downgrade only on resource constraint** — switch to a lighter option only when local resources are insufficient
3. **Prefer latest stable tech** — default to latest stable versions of dependencies

## Code Review

### Review Types

| Type | Command Template | Focus Areas |
|---|---|---|
| Single file | `claude -p "review path/to/file.ts"` | Logic, naming, edge cases |
| PR review | `claude -p "review changes on current branch vs main"` | Overall consistency, test coverage, breaking changes |
| Security review | `claude -p "security-review"` or `/security-review` | OWASP Top 10, injection, auth, sensitive data |
| Architecture review | `claude -p "review project architecture for coupling and improvement opportunities"` | Module boundaries, dependency direction, abstraction levels |

### Review Output Requirements

- Severity levels: 🔴 Blocking / 🟡 Suggestion / 🟢 Optional
- Each issue includes specific code location and suggested fix
- Summary: overall assessment + merge recommendation

## Debugging Workflow

### Standard Debugging Flow

1. **Reproduce the issue** — `claude -p "debug: <problem description + reproduction steps + error logs>"`
2. **Root cause analysis** — Claude Code searches relevant code, analyzes call chains
3. **Propose fix** — minimal fix that addresses the root cause
4. **Verify fix** — run relevant tests or manual verification

### Effective Debug Prompt

```
claude -p "debug: Token not written to cookie after login.
Reproduce: visit /login → enter credentials → redirect to /dashboard → cookie is empty.
Relevant files: src/auth/login.ts, src/middleware/session.ts
Error log: [paste log]"
```

### Debugging Principles

- Suspect **recently changed** code first
- Change one variable at a time, verify before proceeding
- After fixing, run the full test suite, not just the affected tests

## Multi-File Refactoring

### Refactoring Flow

1. **Scope definition** — list all files that need changes
2. **Plan first** — `claude -p "plan: refactor <scope>"` generates the approach
3. **Execute incrementally** — proceed in dependency order, verifying each step
4. **Final verification** — full test suite + lint

### Refactoring Principles

- Keep each step rollback-safe (small commits)
- Restructure without changing behavior
- Write tests first if they are missing, then refactor
- For cross-module refactoring, use subagents for independent modules

### Cross-Module Parallel Example

```
Main session → plan: define refactoring strategy + module boundaries
             → spawn subagent A: refactor Module A
             → spawn subagent B: refactor Module B
             → aggregate → integration test verification
```

## Timeout & Progress Strategy

### Core Settings

- **No timeout** — exec / subagent invocations do not set `timeout`; `runTimeoutSeconds` is 0 or omitted
- Claude Code runs until natural completion or error

### Progress Reporting

> **Poll progress every 10 minutes**

1. Use `process` poll + cron wake (every 10 minutes) or spawn + push notifications
2. Check indicators: file changes, log output, process liveness
3. **Forward progress directly to the user**: "Modified X files, Y remaining, still running"
4. Send final summary when Claude Code completes

### Timeout Strategy by Task Type

| Task Type | Estimated Duration | Progress Reporting |
|---|---|---|
| Single-file edit | < 2 minutes | Usually not needed |
| Small refactor | 5–15 minutes | Every 10 minutes |
| Large refactor | 15–60 minutes | Every 10 minutes |
| New project scaffold | 10–30 minutes | Every 10 minutes |

## Error Handling & Fallback

### Layered Strategy

```
Invoke Claude Code
  ├─ Success → return results to user
  ├─ Recoverable error (excluding timeout)
  │   ├─ Retry once (rephrase prompt)
  │   └─ Still failing → Fallback to built-in tools
  ├─ Network / auth error
  │   └─ Prompt user to check config → Fallback to built-in tools
  └─ Claude Code unavailable
      └─ Complete with built-in tools directly (inform user of fallback mode)
```

### Fallback Rules

1. **Only fall back when Claude Code returns an error**
2. Explicitly inform the user: "Claude Code unavailable, using built-in tools instead"
3. After fallback completion, suggest the user check their Claude Code configuration
4. Never silently fall back — the user must be informed

### Common Errors

| Error | Action |
|---|---|
| `claude: command not found` | Prompt to install: `npm install -g @anthropic-ai/claude-code` |
| Auth failure | Prompt to run `claude login` |
| Project not initialized | Run `claude init` first |
| Truncated output | Narrow prompt scope or split into multiple steps |

## Performance Tuning

### Claude Code Execution Efficiency

- **Narrow scope**: focus on 1–3 files per invocation; split large tasks into multiple calls
- **Preload context**: paste key file contents in the prompt to reduce Claude Code's exploration time
- **Model selection**: use `--model haiku` for simple tasks, default (Opus) for complex ones
- **Avoid redundant init**: once a project is initialized, go straight to task execution

### Local Resource Considerations

- Parallel subagent count should not exceed local CPU core count
- Large builds (e.g., `npm install`, `docker build`) — evaluate time cost during planning
- For GPU training tasks, verify CUDA availability beforehand

## Interactive vs Non-Interactive

| Dimension | Interactive `claude` | Non-Interactive `claude -p` |
|---|---|---|
| Task clarity | Fuzzy, requires exploration | Clear, well-bounded |
| Step count | Multiple steps, mid-course decisions | Single step or linear sequence |
| User involvement | Real-time feedback needed | Not needed |
| Context volume | Accumulates over multiple turns | One-shot, pre-loaded |
| File count | Many files, cross-module | Few files |
| Decision points | Requires user confirmation on direction | Can complete autonomously |

**Rule of thumb: when in doubt, start with `claude -p`. Switch to interactive if the prompt is too large or multi-turn interaction is needed.**

## Windows Environment

### Path Conventions

- Project directories live under `D:\claw\`
- Both backslash `\` and forward slash `/` path separators are accepted (Claude Code handles both on Windows)

### PowerShell Notes

- Ensure Node.js is in PATH before running `claude`
- For encoding issues, run: `[Console]::OutputEncoding = [Text.Encoding]::UTF8`
- When `claude -p` prompts contain non-ASCII characters, ensure PowerShell runs with UTF-8 encoding
- Avoid `&&` chaining in PowerShell 5.1 — use `; if ($?) { }` instead

### Git Integration

- File permissions and line-ending (CRLF/LF) issues are handled automatically by `claude init`
- Recommend configuring `.gitattributes` in the project root for consistent line endings

### Known Limitations

- Terminal width in TUI mode may affect output formatting; ≥ 120 columns recommended
- Windows Terminal is preferred over legacy conhost
