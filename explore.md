# Claude Code Haha — 项目调研报告

> 调研时间：2026-04-17 | 仓库：https://github.com/NanmiCoder/cc-haha

---

## 一、项目概述

**cc-haha** 基于 2026-03-31 泄露的 Claude Code 源码修复而成，是一个功能完整的本地可运行版 Agent 操作系统。

| 类别 | 技术 |
|------|------|
| 运行时 | Bun |
| 语言 | TypeScript（832 文件：482 `.tsx` + 332 `.ts`） |
| 终端 UI | React 19 + Ink 6（100+ 组件、87+ hooks） |
| 协议 | MCP（Model Context Protocol）、LSP |

架构：`bin/claude-haha` → `cli.tsx` → `main.tsx`（800KB 核心） → 工具执行 → API → TUI 渲染

---

## 二、六大独特优势

### 2.1 IM 远程控制（Channel 系统）

**独一无二**——没有任何主流开源框架支持。通过 Telegram / 飞书 / Discord / Slack 远程控制运行中的 Agent。

**原理**：Channel 本质是声明了 `experimental['claude/channel']` 能力的 MCP Server。

```typescript
// 基础声明（推送消息）       // 加上权限中继（远程审批）
{ experimental: {             { experimental: {
    'claude/channel': {}          'claude/channel': {},
  }                               'claude/channel/permission': {}
}                               }
                              }
```

**六层访问控制**：能力声明 → GrowthBook 运行时开关 → OAuth → 组织策略 → 会话白名单 → Marketplace 验证

> GrowthBook 是开源特性开关平台，Anthropic 用它远程控制功能灰度。`tengu_harbor` 是 Channel 总开关（默认 false）。cc-haha 需要绕过这些远程门控才能本地使用。

**权限中继**：敏感操作（如 Bash）的确认提示转发到手机，用户回复 `yes tbxkq` 即可审批。多源竞争（终端/Bridge/Channel/Hook）先到先得。

### 2.2 多 Agent 协作体系

#### 6 种内置 Agent

| Agent | 模式 | 模型 | 用途 |
|-------|------|------|------|
| general-purpose | 读写 | 继承 | 通用任务 |
| Explore | 只读 | Haiku | 快速低成本探索 |
| Plan | 只读 | 继承 | 架构规划 |
| verification | 只读 | 继承 | 对抗性验证 |
| claude-code-guide | 只读 | Haiku | 文档指南 |
| statusline-setup | 读写 | Sonnet | 状态栏配置 |

**verification Agent**："你的工作不是确认有效——而是尝试破坏它。" 后台只读运行，3+ 文件编辑后自动触发。按变更类型选择验证策略（前端启 dev server、后端 curl 端点、基础设施 dry-run 等）。输出必须包含 `Command run` + `Output observed` + `VERDICT: PASS/FAIL/PARTIAL`，没有命令输出的 PASS 会被拒绝。

**statusline-setup Agent**：通过 `/statusline` 触发，读取 PS1 → 转换转义序列 → 写入 `~/.claude/settings.json`。可展示会话、模型、上下文窗口、限额等信息。

#### Teams + 并行 + 隔离

- **Agent Teams**：创建团队 → SendMessage 通信 → 广播 → 协同关停
- **并行执行**：一条消息可同时生成多个 Agent
- **Worktree 隔离**：Agent 在独立 git worktree 中运行
- **自动后台化**：前台任务 > 120 秒自动转后台
- **7 种权限模式**：default / plan / acceptEdits / bypassPermissions / dontAsk / auto / bubble
- **自定义 Agent**：`.claude/agents/*.md` Markdown 定义

### 2.3 AutoDream "做梦"记忆整理

#### 四种记忆类型

| 类型 | 示例 |
|------|------|
| User（用户画像） | "深厚 Go 经验，React 新手" |
| Feedback（行为反馈） | "不要尾部摘要" |
| Project（项目动态） | "2026-03-05 起合并冻结" |
| Reference（外部引用） | "oncall 看 grafana.internal/d/api-latency" |

#### 存储路径

所有记忆保存在本地磁盘，路径解析优先级：
1. 环境变量 `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE`
2. `settings.json` 的 `autoMemoryDirectory`
3. **默认** `~/.claude/projects/<sanitized-git-root>/memory/`

```
memory/
├── MEMORY.md              ← 索引（始终加载，≤200行/25KB）
├── user_role.md           ← 各主题记忆文件（YAML frontmatter + Markdown）
├── feedback_testing.md
├── .consolidate-lock      ← 锁文件（PID + mtime 双重用途）
├── logs/                  ← 日志（Assistant 模式）
└── team/                  ← 团队共享记忆
```

#### AutoDream 工作流

**触发**：每次 AI 回复后 fire-and-forget 检查，通过五重门控：
1. 功能开关启用
2. 距上次整合 ≥ 24h
3. 上次扫描后 ≥ 10 分钟
4. 新会话 ≥ 5 个
5. 获取锁成功

**执行**：启动一个 **forked subagent**（不是程序逻辑，是 AI 自主执行），四阶段：
1. **Orient** — ls 记忆目录，读 MEMORY.md，浏览已有文件
2. **Gather** — 扫描日志、检查漂移记忆、窄范围 grep 转录
3. **Consolidate** — 合并到已有文件，相对日期→绝对日期，删除过时事实
4. **Prune** — MEMORY.md 保持 ≤200行，删旧指针，压缩冗长条目

> 详细 Prompt 和实现细节见 [autodream.md](autodream.md)

### 2.4 Computer Use 桌面控制

24 个 MCP 工具实现截图-分析-操作闭环。官方依赖私有原生模块，本项目用 **Python Bridge**（pyautogui + mss + pyobjc）替代。

| 类别 | 工具 |
|------|------|
| 截屏 | screenshot、zoom |
| 鼠标 | left_click、right_click、double_click、drag、scroll 等 11 种 |
| 键盘 | type、key、hold_key |
| 应用/剪贴板 | open_application、read/write_clipboard |

安全机制：应用白名单、文件锁并发保护、剪贴板自动保存恢复、首次使用自动创建 venv。

### 2.5 Skills 声明式插件系统

用 Markdown 文件定义自动化工作流，6 种来源：Bundled → Managed → User → Project → Plugin → MCP。

```yaml
---
name: 我的技能
context: fork           # inline 或 fork（子代理隔离）
allowed-tools: "Bash, Read"
paths: "src/**/*.ts"    # 条件激活 glob
---
Markdown 格式的提示词...
```

核心特性：条件激活（glob 匹配时才对模型可见）、Fork 执行（独立 token 预算）、运行时发现（自动向上遍历）、12+ 内置 Skills（/verify、/debug、/dream、/batch 等）。

### 2.6 多模型支持（100+）

| 方式 | 说明 |
|------|------|
| LiteLLM 代理 | Anthropic → OpenAI 协议转换，100+ 模型 |
| 直连 | OpenRouter、MiniMax 等兼容服务 |
| 原生集成 | AWS Bedrock、GCP |

限制：Extended Thinking 和 Prompt Caching 为 Anthropic 专有；小模型（<70B）可能无法处理复杂工具调用。

---

## 三、竞品对比

| 维度 | cc-haha | Aider | OpenDevin | SWE-Agent |
|------|---------|-------|-----------|-----------|
| 多 Agent | 6 种 + Teams + 并行 | 单 Agent | 多 Agent（无 Teams） | 单 Agent |
| IM 远程控制 | Telegram/飞书/Discord | 无 | 无 | 无 |
| 记忆系统 | 4 类 + AutoDream | 无 | 无 | 无 |
| 桌面控制 | 24 个工具 | 无 | 仅浏览器 | 无 |
| Skills 插件 | Markdown 声明式 | 无 | 无 | 无 |
| MCP 集成 | 全系统基于 MCP | 无 | 无 | 无 |
| 权限安全 | 7 种模式 + 六层控制 | 基础确认 | Docker 隔离 | Docker 隔离 |

---

## 四、补充资料

| 文件 | 内容 |
|------|------|
| [autodream.md](autodream.md) | AutoDream Prompt 原文、Forked Agent 机制、设计要点 |
| [agent-loop-and-compression.md](agent-loop-and-compression.md) | AsyncGenerator Agent Loop、上下文四级渐进压缩管线 |

---

*本报告基于仓库源码和文档分析，将随调研深入持续更新。*
