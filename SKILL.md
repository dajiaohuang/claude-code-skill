---
name: claude-code
description: Call Claude Code CLI for coding tasks — write, refactor, review, debug, and scaffold new projects. Use when the user asks for code creation, modification, review, or debugging. Fall back to built-in tools only if Claude Code errors out.
---

# Claude Code — 调用指南

## 目录

- [核心原则](#核心原则)
- [速查表](#速查表)
- [新项目工作流](#新项目工作流)
- [现有项目更新](#现有项目更新)
- [调用方式](#调用方式)
- [Subagent 并发模式](#subagent-并发模式)
- [Prompt 编写指南](#prompt-编写指南)
- [代码审查](#代码审查)
- [调试流程](#调试流程)
- [多文件重构](#多文件重构)
- [超时与进度策略](#超时与进度策略)
- [错误处理与 Fallback](#错误处理与-fallback)
- [性能调优](#性能调优)
- [交互 vs 非交互选择](#交互-vs-非交互选择)
- [Windows 环境说明](#windows-环境说明)

---

## 核心原则

**所有代码相关工作一律通过 Claude Code CLI 完成**，包括：编写、修改、重构、审查、调试、搭建项目。不使用内置工具（write / edit / exec）直接操作代码文件。

仅当 Claude Code 报错或不可用时，才 fall back 到内置工具。

## 速查表

| 场景 | 命令 / 方式 | 备注 |
|---|---|---|
| 交互式 TUI | `claude` | 需要 `pty: true` |
| 单次任务 | `claude -p "prompt"` | 非交互，跑完即退出 |
| 规划模式 | `claude -p "plan: 需求..."` | 先生成计划，用户确认后实现 |
| 代码审查 | `claude -p "审查 <目标>"` | 详见 [代码审查](#代码审查) |
| 调试 | `claude -p "debug: <问题>"` | 详见 [调试流程](#调试流程) |
| 批量并行任务 | spawn subagent × N | 详见 [Subagent 并发模式](#subagent-并发模式) |
| 已 init 项目 | 一律 `claude` / `claude -p` | 禁止内置工具直接改文件 |
| Fallback | 内置工具 | 仅当 Claude Code 报错时 |

## 新项目工作流

1. **建目录** — 在 `D:\claw\` 下创建项目文件夹
2. **`claude init`** — 进入项目目录执行初始化，生成 CLAUDE.md
3. **Plan Mode** — 用 `claude -p "plan: <需求>"` 进入规划模式，Claude 生成方案后用户确认再执行
4. **决策默认值** — Claude 提问时（功能选型 / UI / 数据库 / 架构等），默认选**功能最大化**的方案，仅当超出本机性能才降级
5. **验证** — 完成后要求 Claude 运行测试、构建或启动服务，确保交付物可用

## 现有项目更新

| 需求状态 | 做法 |
|---|---|
| **无详细需求** | 先让 Claude 浏览代码找出可改进点 → 列出清单 + 方案 → 返回让用户决策 |
| **有详细需求** | 直接传给 Claude 执行 |
| **已 `claude init` 的项目** | 禁止用内置工具改文件，所有变更走 Claude Code |

## 调用方式

### 交互式 TUI

```bash
claude
```

- 适用：探索性任务、需要多轮对话的复杂重构、边走边看
- 要求：`pty: true`（终端模拟）
- 优势：用户可中途介入、实时调整方向

### 非交互单次

```bash
claude -p "prompt"
```

- 适用：明确、边界清晰的单次任务
- 优势：跑完即退，不占用终端

### 规划模式

```bash
claude -p "plan: <需求描述>"
```

或交互式下：

```
/plan
```

### 其他常用参数

| 参数 | 用途 |
|---|---|
| `--output-format json` | 结构化输出 |
| `--model` | 指定模型 |
| `--max-turns` | 限制最大轮次 |

## Subagent 并发模式

**首选模式** — 通过 subagent spawn 并行调起多个 Claude Code 实例，避免阻塞主会话。

### 适用场景

- 同时处理多个独立文件
- 批量代码审查
- 并行执行多步独立任务（如前后端同时改动）
- 大型项目的分模块重构

### 调用模式

```
主会话 → spawn subagent A (claude -p "任务A")
       → spawn subagent B (claude -p "任务B")
       → spawn subagent C (claude -p "任务C")
       → 等待所有完成 → 汇总结果
```

### 注意事项

- 各 subagent 之间任务需**互不依赖**（无共享文件写入）
- 完成后由主会话汇总并做集成验证
- subagent 同样不设超时，自然完成

## Prompt 编写指南

### 好的 Prompt

- 明确描述**目标**而非**步骤**："重构 UserService 使方法不超过 20 行" 优于 "拆分 UserService 的 login 方法"
- 提供必要的**上下文**：相关文件、已有设计决策、约束条件
- 指定**输出格式**："列出 5 个优化点，每个一行"
- 对于审查类任务，指明**审查维度**：安全性、性能、可读性、架构一致性

### 避免的做法

- 模糊指令："帮我看看代码"
- 过度约束："只能用 X 库 / Y 模式"，除非确实必要
- 在一个 prompt 里塞太多不相关的任务

### 决策原则

当 Claude Code 在执行中提出问题时，默认策略：
1. **功能最大化** — 选实现最多的选项
2. **超出性能才降级** — 仅当涉及本机资源不足时选轻量方案
3. **优先最新稳定技术** — 依赖选最新稳定版，不锁定旧版本

## 代码审查

### 审查类型

| 类型 | 命令模板 | 关注点 |
|---|---|---|
| 单文件审查 | `claude -p "审查 path/to/file.ts"` | 逻辑、命名、边界处理 |
| PR 审查 | `claude -p "审查当前分支相对于 main 的改动"` | 整体一致性、测试覆盖、破坏性变更 |
| 安全审查 | `claude -p "security-review"` 或 `/security-review` | OWASP Top 10、注入、认证、敏感数据 |
| 架构审查 | `claude -p "审查项目架构，找耦合点和改进机会"` | 模块边界、依赖方向、抽象层次 |

### 审查输出要求

- 按严重程度分级：🔴 阻塞 / 🟡 建议 / 🟢 可选
- 每条问题附带具体代码位置和建议修复方案
- 结尾总结：总体评价 + 是否建议合并

## 调试流程

### 标准调试流程

1. **复述问题** — `claude -p "debug: <问题描述 + 复现步骤 + 错误日志>"`
2. **定位根因** — Claude Code 搜索相关代码、分析调用链
3. **提出修复** — 给出最小化修复方案
4. **验证修复** — 运行相关测试或用例确认

### 高效调试 Prompt

```
claude -p "debug: 用户登录后 token 未写入 cookie。
复现: 访问 /login → 输入凭据 → 跳转 /dashboard → cookie 为空。
相关文件: src/auth/login.ts, src/middleware/session.ts
错误日志: [粘贴日志]"
```

### 调试原则

- 优先怀疑**最近改动**的代码
- 一次只改一个变量，确认后再继续
- 修完后跑完整测试套件，不只跑相关用例

## 多文件重构

### 重构流程

1. **圈定范围** — 列出所有需要改动的文件
2. **Plan 先行** — `claude -p "plan: 重构 <范围>"` 生成方案
3. **分步执行** — 按依赖顺序逐步完成，每步验证
4. **最后统一验证** — 全量测试 + lint

### 重构原则

- 保持每步可回滚（小步提交）
- 不改行为的前提下调整结构
- 先写测试（如果缺失），再重构
- 跨模块重构使用 subagent 并行处理独立模块

### 跨模块并行示例

```
主会话 → plan: 确定重构方案 + 模块边界
       → spawn subagent A: 重构 Module A
       → spawn subagent B: 重构 Module B
       → 汇总 → 集成测试验证
```

## 超时与进度策略

### 核心设定

- **永不超时** — exec / subagent 不设 `timeout` 参数，`runTimeoutSeconds` 设为 0 或省略
- Claude Code 跑到自然完成或报错才结束

### 进度汇报机制

> **每 10 分钟轮询一次进度**

1. 用 `process` poll + cron wake（每 10 分钟弹一次）或 spawn + 推送
2. 检查指标：文件改动数、日志输出、进程存活状态
3. **直接转发进度给用户**："已在改 XX 文件，还差 YY，仍在跑"
4. Claude Code 结束后发送最终结果摘要

### 超时策略细化

| 任务类型 | 预估时长 | 进度汇报 |
|---|---|---|
| 单文件修改 | < 2 分钟 | 通常不需要 |
| 中小重构 | 5–15 分钟 | 每 10 分钟 |
| 大型重构 | 15–60 分钟 | 每 10 分钟 |
| 新项目搭建 | 10–30 分钟 | 每 10 分钟 |

## 错误处理与 Fallback

### 分层策略

```
调用 Claude Code
  ├─ 成功 → 返回结果给用户
  ├─ 可恢复错误（超时除外）
  │   ├─ 重试 1 次（换 prompt 措辞）
  │   └─ 仍失败 → Fallback 到内置工具
  ├─ 网络/认证错误
  │   └─ 提示用户检查配置 → Fallback 到内置工具
  └─ Claude Code 不可用
      └─ 直接用内置工具完成（告知用户当前模式为 fallback）
```

### Fallback 规则

1. **仅当 Claude Code 返回错误时才 fall back**
2. Fallback 时明确告知用户："Claude Code 不可用，使用内置工具替代"
3. Fallback 完成后建议用户排查 Claude Code 配置
4. 不允许静默 fallback — 用户必须知情

### 常见错误处理

| 错误 | 处理 |
|---|---|
| `claude: command not found` | 提示安装 `npm install -g @anthropic-ai/claude-code` |
| 认证失败 | 提示运行 `claude login` |
| 项目未 init | 先运行 `claude init` |
| 输出截断 | 缩小 prompt 范围或拆分为多步 |

## 性能调优

### Claude Code 执行效率优化

- **缩小范围**：单次任务聚焦 1–3 个文件，大任务拆成多次调用
- **预加载上下文**：把关键文件内容贴在 prompt 里，减少 Claude Code 自行探索的时间
- **指定模型**：简单任务用 `--model haiku`，复杂任务用默认（Opus）
- **避免重复 init**：已 init 的项目直接执行任务，不做重复初始化

### 本机资源考量

- 并行 subagent 数量不超过本机 CPU 核心数
- 大型构建（如 `npm install`、`docker build`）在计划阶段评估耗时
- 涉及 GPU 训练的任务确认 CUDA 可用

## 交互 vs 非交互选择

| 维度 | 交互式 `claude` | 非交互 `claude -p` |
|---|---|---|
| 任务清晰度 | 模糊、需要探索 | 明确、边界清晰 |
| 步骤数 | 多步、需要中途决策 | 单步或线性步骤 |
| 用户参与 | 需要实时反馈 | 不需要 |
| 上下文量 | 需要多轮累积 | 一次性给足 |
| 文件数 | 多文件、跨模块 | 少量文件 |
| 决策点 | 需要用户确认方向 | 可自主完成 |

**原则：不确定时先用 `claude -p`，如果 prompt 太大或需要多轮再用交互式。**

## Windows 环境说明

### 路径约定

- 项目目录统一放在 `D:\claw\` 下
- 路径分隔符使用反斜杠 `\` 或正斜杠 `/` 均可（Claude Code 在 Windows 下兼容两者）

### PowerShell 注意事项

- 执行 `claude` 前确保 Node.js 在 PATH 中
- 如遇编码问题，在 PowerShell 中执行 `[Console]::OutputEncoding = [Text.Encoding]::UTF8`
- `claude -p` 的 prompt 中包含中文时，确保 PowerShell 以 UTF-8 编码运行
- 避免使用 `&&` 链式操作符（PowerShell 5.1 不支持），使用 `; if ($?) { }` 代替

### Git 集成

- Windows 下的文件权限和换行符 (CRLF/LF) 问题由 `claude init` 自动处理
- 建议在项目根目录配置 `.gitattributes` 统一换行符

### 已知限制

- TUI 模式下终端宽度可能影响输出格式，建议 ≥ 120 列
- Windows Terminal 优于传统 conhost，推荐使用
