# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

**Claude Code Haha** 是基于 Anthropic npm 仓库泄露源码（2026-03-31）修复并增强的本地可运行版 Claude Code。它是一个功能完整的终端 UI（TUI）应用，通过 Anthropic 兼容接口支持多种 AI 模型和 API。

## 常用命令

```bash
# 安装依赖
bun install

# 复制并配置环境变量
cp .env.example .env

# 启动交互式 TUI 模式
./bin/claude-haha

# 无头/打印模式
./bin/claude-haha -p "你的提示词"

# 降级恢复 CLI 模式（无 Ink TUI）
CLAUDE_CODE_FORCE_RECOVERY_CLI=1 ./bin/claude-haha

# Windows（PowerShell 或 cmd）
bun --env-file=.env ./src/entrypoints/cli.tsx

# 文档开发服务器
npm run docs:dev

# 文档构建
npm run docs:build
```

`package.json` 中没有定义测试或 lint 脚本。

## 架构

应用采用分层架构：**入口脚本 → CLI 引导 → 核心应用 → 工具执行 → API 客户端 → TUI 渲染**。

### 入口点

- **`bin/claude-haha`** — Bash 包装脚本，检测平台、设置工作目录，委托给 Bun 运行
- **`src/entrypoints/cli.tsx`** — CLI 引导层，处理快速路径（`--version`、`--dump-system-prompt`），启动守护进程，初始化 MCP/Computer Use，最终加载 `main.tsx`
- **`src/main.tsx`** — 800KB+ 的单体核心文件：Commander.js CLI 定义、启动性能分析、React/Ink TUI 渲染及工具注册
- **`src/localRecoveryCli.ts`** — 基于 readline 的降级 REPL，在 Ink TUI 不可用时启用
- **`preload.ts`** — Bun 预加载脚本（通过 `bunfig.toml` 配置），设置全局 `MACRO` 对象（版本/构建信息），并切换到调用方的工作目录

### 核心子系统

| 子系统 | 位置 | 用途 |
|--------|------|------|
| 查询引擎 | `src/QueryEngine.ts`、`src/query.ts` | 查询处理与 LLM 消息编排 |
| 工具框架 | `src/Tool.ts`、`src/tools.ts`、`src/tools/` | 带权限检查的抽象工具系统（Bash、Edit、Grep 等） |
| 斜杠命令 | `src/commands.ts`、`src/commands/` | 100+ 个斜杠命令实现 |
| 状态存储 | `src/state/` | 类 Redux 的集中式应用状态（`AppState`/`AppStateStore`） |
| 记忆系统 | `src/memdir/` | 跨会话持久化的文件系统记忆 |
| Skills 系统 | `src/skills/` | 可插拔的能力扩展与自定义工作流 |
| 多 Agent | `src/tasks/`、`src/coordinator/` | Agent 编排与并行任务执行 |
| 桥接/远程 | `src/bridge/`、`src/remote/` | CCR（Claude Code Remote）：基于 WebSocket 的远程会话，使用 JWT 认证 |
| 频道系统 | `src/services/` | 通过 Telegram/飞书/Discord 远程控制 Agent |
| MCP | `src/entrypoints/mcp.ts`、`src/services/` | Model Context Protocol 服务端与客户端 |
| Computer Use | `src/vendor/` | 截图、鼠标/键盘模拟 |
| TUI 组件 | `src/components/`、`src/screens/`、`src/hooks/` | 100+ React/Ink 组件、87+ hooks、多个屏幕（REPL、Doctor、ResumeConversation） |

### 多模型 API 支持

API 层对多个服务商进行了统一抽象，通过 `.env` 配置：
- **Anthropic**（默认）— `ANTHROPIC_API_KEY`
- **MiniMax** — 直接使用 Anthropic 兼容端点
- **OpenRouter** — 直接集成
- **DeepSeek / OpenAI** — 通过 LiteLLM 代理
- **AWS Bedrock** — 通过 `@aws-sdk/client-bedrock-runtime`
- **GCP** — 通过 `google-auth-library`

详见 `docs/en/guide/env-vars.md` 和 `docs/en/guide/third-party-models.md`。

### 构建系统

- **运行时**：Bun（必需，正常使用不支持 Node.js 回退）
- **语言**：TypeScript，通过 Bun 的打包器编译，`moduleResolution: "bundler"`
- **JSX**：React 17+ transform（`react-jsx`），通过 Ink 在终端中渲染
- **模块桩**：`stubs/` 目录包含与 Bun 不兼容的包的 shim（`@ant/claude-for-chrome-mcp`、`color-diff-napi`）
- **路径别名**：`tsconfig.json` 中 `src/*` 映射到 `./src/*`
