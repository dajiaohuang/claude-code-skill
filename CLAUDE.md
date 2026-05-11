# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is an [OpenClaw](https://openclaw.ai) AgentSkill — a skill definition that tells AI assistants to delegate code work to the Claude Code CLI (`claude`) instead of using built-in file manipulation tools.

## File roles

- **`SKILL.md`** — The skill definition loaded by OpenClaw at runtime. YAML frontmatter (`---` delimited) contains `name` and `description` used for skill matching. The markdown body is injected into the assistant's system prompt as instructions.
- **`README.md`** — Human-facing documentation (installation, usage, license). Not loaded by OpenClaw at runtime.

## Skill frontmatter

```yaml
---
name: <skill-name>
description: <one-line summary used for skill matching>
---
```

The `description` field is critical — OpenClaw uses it to decide when to trigger the skill. Make it specific about triggering conditions (when to activate and when to skip).

## Development workflow

No build/lint/test tooling — this repo contains only markdown files. To test a skill change: restart the OpenClaw agent or reload skills, then issue a request that should match the skill's `description`.

When editing `SKILL.md`, preserve valid YAML frontmatter between `---` delimiters. The body uses standard Markdown.

## Key design decisions

- All code work is routed through `claude` CLI (TUI via `claude`, non-interactive via `claude -p`). Built-in tools are a fallback only when Claude Code errors.
- For projects already initialized with `claude init`, never use built-in file tools — all changes go through Claude Code.
- On Windows, project directories live under `D:\claw\`.
- Subagent spawning is the preferred pattern for parallel/batch Claude Code invocations to avoid blocking the main session.
- Timeouts are deliberately disabled so Claude Code runs to natural completion, with 10-minute progress check-ins.
