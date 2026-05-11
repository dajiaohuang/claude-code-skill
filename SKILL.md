---
name: claude-code
description: Call Claude Code CLI for coding tasks — write, refactor, review, debug, and scaffold new projects. Use when the user asks for code creation, modification, review, or debugging. Fall back to built-in tools only if Claude Code errors out.
---

# Claude Code — 调用指南

## 核心原则

所有代码相关工作（写、改、重构、审查、debug）**一律通过 Claude Code 完成**。不使用内置的 write/edit/exec 直接操作代码文件。仅当 Claude Code 报错或不可用时才 fall back 回来。

## 新项目工作流

1. **建文件夹** — 在 `D:\claw\` 下创建项目目录
2. **`claude init`** — 进入项目目录执行初始化
3. **Plan Mode** — 用 `claude -p "<需求>"` 或 `/plan` 进入规划模式
4. **澄清问题决策** — Claude 提出问题时（功能/UI/数据库等），默认选**功能最大化**的选项，超出本机性能才降级

## 项目更新规则

| 需求状态 | 做法 |
|---|---|
| **无详细需求** | 告诉 Claude 浏览代码找出改进点 → 列清单+方案 → 返回让用户决定 |
| **有详细需求** | 直接给 Claude 执行 |
| **已 `claude init` 的项目** | 禁止用内置工具改文件，所有变更走 Claude Code |

## 调用方式

| 场景 | 命令 | 说明 |
|---|---|---|
| 交互式 TUI | `claude` | 全终端交互，需要 pty:true |
| 单次任务 | `claude -p "prompt"` | 非交互，跑完即退出 |
| 规划模式 | `claude -p "plan: 需求..."` | 先规划再实现 |
| 审查代码 | `claude -p "审查 <file>"` | 代码审查 |
| 批量任务 | spawn subagent | 通过 subagent 调起，避免阻塞主会话 |

## 超时设定

- 所有 `claude` 命令的 exec 超时设为 **3600 秒（1 小时）**
- `timeout` 参数传入 `3600`，`yieldMs` 用默认值
- subagent 的 `runTimeoutSeconds` 也设为 **3600**
