# 🦀 Claude Code 技能 — OpenClaw AgentSkill

让 AI 助手通过 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) 执行代码任务，而不是用内置工具直接操作文件。

## 用途

当用户提出代码相关需求（写代码、重构、审查、debug、新项目搭建）时，助手调用 Claude Code 来完成，而非使用内置的 write/edit/exec 直接操作代码文件。

## 安装

### 前置要求

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI (`npm install -g @anthropic-ai/claude-code`)
- [OpenClaw](https://openclaw.ai) 或兼容 AgentSkill 架构的 AI 助手

### 安装方式

```bash
# 克隆到 OpenClaw skills 目录
cd ~/.openclaw/skills
git clone https://github.com/dajiaohuang/claude-code-skill.git

# 或者复制到你的 OpenClaw 安装目录
cp -r claude-code-skill /path/to/openclaw/skills/
```

## 工作流

### 新项目

1. 创建项目目录（`D:\claw\<project-name>`）
2. `claude init` 初始化
3. Plan Mode：用 `claude -p "需求"` 或 `/plan` 进入规划模式
4. 澄清问题时默认选**功能最大化**的选项

### 现有项目更新

| 场景 | 做法 |
|---|---|
| 无详细需求 | Claude 找改进点 → 列清单 → 用户决定 |
| 有详细需求 | 直接让 Claude 执行 |
| 已 init 的项目 | 禁止内置工具改文件 |

## 调用方式速查

| 场景 | 命令 |
|---|---|
| 交互式 TUI | `claude` |
| 单次任务 | `claude -p "prompt"` |
| 审查代码 | `claude -p "审查 <file>"` |
| 批量任务 | 通过 subagent 调起 |

## 许可证

MIT
