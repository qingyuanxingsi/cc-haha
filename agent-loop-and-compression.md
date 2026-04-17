# Agent Loop（AsyncGenerator）与上下文压缩管线

> 源码：`src/query.ts`（1730行）、`src/QueryEngine.ts`（1296行）、`src/services/compact/`

---

## 一、Agent Loop — AsyncGenerator 三层链

### 为什么用 AsyncGenerator

| 能力 | 实现方式 |
|------|----------|
| **流式** | 每个 SSE 事件、消息、工具结果通过 `yield` 逐个产出 |
| **中断** | 检查 `abortController.signal.aborted` 后 `return`，生成器链自动终止 |
| **背压** | 消费者不调 `.next()` 时生成器暂停在 `yield`，不积压 |

### 三层嵌套结构

```
QueryEngine.submitMessage()  ←  query()          ←  queryLoop()
   for await 消费                yield* 委托          while(true) { yield 事件 }
```

**`yield*` 的核心作用**：
- `yield` 产出的值**向上穿透**所有 `yield*` 层，到达最终消费者
- `return` 的值只被直接上层的 `yield*` 捕获，消费者看不到
- 形成两条并行通道：`yield` 给消费者看事件流，`return` 给上层做控制流决策

```typescript
// query.ts — 第 1 层
async function* query(params): AsyncGenerator<StreamEvent, Terminal> {
  const terminal = yield* queryLoop(params)  // 透传所有事件，捕获 return
  return terminal
}

// query.ts — 第 2 层（核心）
async function* queryLoop(params): AsyncGenerator<StreamEvent, Terminal> {
  while (true) {
    // 压缩管线 → API 流式调用 → 中断检查 → 停止钩子 → 工具执行
    for await (const msg of deps.callModel(...)) { yield msg }
    if (aborted) return { reason: 'aborted' }
    const stopResult = yield* handleStopHooks(...)  // yield* 再次委托
    for await (const update of toolUpdates) { yield update.message }
  }
}
```

### 事件流转路径

```
API SSE → queryModelWithStreaming() yield → queryLoop() yield → query() yield* 透传
→ submitMessage() for await → 调用者（CLI / SDK / Desktop）
```

### 并发控制

`generators.ts` 的 `all()` 函数：带并发上限的 `Promise.race` 调度多个 AsyncGenerator。

---

## 二、上下文压缩管线

### 设计原则

**能不花钱就不花钱，能少丢就少丢**——从零成本到 API 调用渐进降级。

### 触发时机

**每轮都跑**。`queryLoop` 的 `while(true)` 中，每次 AI 回复前按顺序检查四关。大部分轮次空跑（几次内存读取 + 一次 token 估算）即跳过。

```
用户发消息 → 跑四关检查 → 调 API → AI 回复
                                     ↓ 有工具调用
              执行工具 → continue → 再跑四关检查 → 调 API → ...
```

### 四关检查 + 反应式恢复

```
┌─ 第 1 关：Snip ──────────────────────────────────────────────┐
│  触发：消息数量超过阈值                                        │
│  操作：直接删掉最老的几条消息，不做总结                          │
│  成本：0 API  信息：完全丢失                                   │
│  类比：撕掉笔记本最前面几页                                     │
└──────────────────────────────────────────────────────────────┘
         ↓
┌─ 第 2 关：MicroCompact ─────────────────────────────────────┐
│  触发：距上次回复 > N 分钟（缓存冷）或 可压缩工具数 > 阈值       │
│  操作：保留最近 N 个工具输出，其余替换为占位文字                  │
│        缓存热时走 cache_edits 不动本地消息                      │
│  成本：0 API  信息：工具输出细节丢失，调用记录保留                │
│  类比：把查过的 10 个文件内容擦掉，只留标题                      │
└──────────────────────────────────────────────────────────────┘
         ↓
┌─ 第 3 关：Context Collapse ──────────────────────────────────┐
│  触发：上下文 ≥ ~90% 且有可折叠的连续对话段                      │
│  操作：折叠为摘要，原始消息保留在内存（读时投影）                 │
│        折叠后若 < 93% → AutoCompact 不触发                     │
│  成本：0 API*  信息：可恢复                                    │
│  类比：聊天记录折叠成 "[讨论了认证模块重构]"，点开能看原文         │
└──────────────────────────────────────────────────────────────┘
         ↓
┌─ 第 4 关：AutoCompact ───────────────────────────────────────┐
│  触发：token ≥ 93%（有效窗口 - 13000）                         │
│  限制：压缩源跳过（防死锁）、Collapse 开启时跳过、失败 3 次熔断    │
│  操作：                                                       │
│    4a. Session Memory — 用已有记忆裁剪旧消息（0 API）           │
│    4b. Full Compact — LLM 摘要替换全部（1 API），恢复最近 5 文件  │
│  类比：让 AI 花 10 秒写摘要，撕掉旧对话，贴在第一页               │
└──────────────────────────────────────────────────────────────┘
         ↓
      正常调 API
         ↓ 如果 413
┌─ 反应式恢复（安全网）────────────────────────────────────────┐
│  1. 排空 Collapse 暂存折叠 → 重试                             │
│  2. Reactive Compact 紧急全量压缩 → 重试                       │
│  3. 报错给用户                                                │
│  另：max_output_tokens 错误 → 升级 8K→64K → 注入恢复消息（×3）  │
└──────────────────────────────────────────────────────────────┘
```

### 总结表

| 级别 | 名称 | 触发条件 | API 成本 | 信息损失 |
|------|------|---------|---------|---------|
| 1 | Snip | 消息数超限 | 0 | 完全丢失 |
| 2 | MicroCompact | 时间 > N 分钟 或 工具数 > 阈值 | 0 | 工具输出丢失 |
| 3 | Context Collapse | ≥ 90% 且有可折叠段 | 0* | 可恢复 |
| 4a | Session Memory | ≥ 93% 且记忆非空 | 0 | 旧消息丢失 |
| 4b | Full Compact | ≥ 93% 且 4a 失败 | 1 次 | 全部丢失 |
| 5 | Reactive | API 413 | 1 次 | 全部丢失 |

---

*本文档作为 explore.md 的补充资料。*
