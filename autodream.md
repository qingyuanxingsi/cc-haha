# AutoDream "做梦"记忆整合

> 源码：`src/services/autoDream/`

---

## 一、核心机制

AutoDream 的整合**由 forked subagent 自主执行**，不是预编程算法：

```typescript
const result = await runForkedAgent({
  promptMessages: [createUserMessage({ content: prompt })],
  canUseTool: createAutoMemCanUseTool(memoryRoot),  // 限制只能写 memory 目录
  querySource: 'auto_dream',
  forkLabel: 'auto_dream',
  skipTranscript: true,
  overrides: { abortController },
  onMessage: makeDreamProgressWatcher(taskId, setAppState),
})
```

子智能体共享父对话的 prompt cache（省 token），可调用 Read/Grep/Glob/只读 Bash/Edit/Write 工具，自主决定怎么读、合并、修剪。每次"做梦"本质上是一次完整的 LLM 对话。

---

## 二、Consolidation Prompt 原文

```markdown
# Dream: Memory Consolidation

You are performing a dream — a reflective pass over your memory files.
Synthesize what you've learned recently into durable, well-organized
memories so that future sessions can orient quickly.

Memory directory: `<memoryRoot>`
This directory already exists — write to it directly with the Write tool.

Session transcripts: `<transcriptDir>` (large JSONL files — grep narrowly,
don't read whole files)

## Phase 1 — Orient
- `ls` the memory directory to see what already exists
- Read `MEMORY.md` to understand the current index
- Skim existing topic files so you improve them rather than creating duplicates

## Phase 2 — Gather recent signal
Sources in rough priority order:
1. **Daily logs** (`logs/YYYY/MM/YYYY-MM-DD.md`) if present
2. **Existing memories that drifted** — facts that contradict the codebase now
3. **Transcript search** — grep JSONL transcripts for narrow terms:
   `grep -rn "<narrow term>" <transcriptDir>/ --include="*.jsonl" | tail -50`

Don't exhaustively read transcripts. Look only for things you already suspect matter.

## Phase 3 — Consolidate
- Merge new signal into existing topic files rather than creating near-duplicates
- Convert relative dates ("yesterday", "last week") to absolute dates
- Delete contradicted facts

## Phase 4 — Prune and index
Update `MEMORY.md` so it stays under 200 lines AND under ~25KB.
Each entry one line under ~150 chars: `- [Title](file.md) — one-line hook`.
Never write memory content directly into it.
- Remove pointers to stale/wrong/superseded memories
- Demote verbose entries (>200 chars → move detail into topic file)
- Resolve contradictions

Return a brief summary of what you consolidated, updated, or pruned.
```

额外上下文（autoDream 模式独有，`/dream` 手动模式不注入）：

```markdown
**Tool constraints for this run:** Bash is restricted to read-only commands
(`ls`, `find`, `grep`, `cat`, `stat`, `wc`, `head`, `tail`, and similar).

Sessions since last consolidation (5):
- session-abc123
- ...
```

---

## 三、Prompt 设计要点

| 要点 | 原因 |
|------|------|
| 工具约束不在主 prompt 中 | `/dream` 手动模式复用同一 prompt 但拥有完整权限，约束通过 `extra` 单独注入 |
| 不预加载记忆内容 | 记忆可能很大，让 agent 自己按需 `ls` + `Read` |
| Transcript 窄范围 grep | JSONL 很大，穷尽读取爆 token，限制 `grep + tail -50` |
| 相对日期→绝对日期 | "yesterday" 过几天就失效 |
| MEMORY.md 是索引不是内容 | 始终加载到上下文，必须精简（≤200行/25KB） |
| Merge, don't create | 防止记忆膨胀，强制先检查已有文件 |

---

## 四、记忆文件格式

```markdown
---
name: 测试策略偏好
description: 集成测试必须使用真实数据库，不要 mock
type: feedback
---
集成测试必须使用真实数据库，不要 mock。
**Why:** mock/生产差异掩盖了上季度迁移失败的问题。
**How to apply:** 确保数据库操作使用真实连接。
```

四种类型：`user`、`feedback`、`project`、`reference`

---

## 五、关键常量

| 常量 | 值 |
|------|-----|
| MEMORY.md 行数上限 | 200 行 |
| MEMORY.md 大小上限 | 25 KB |
| 最小间隔 | 24 小时 |
| 最小会话数 | 5 个 |
| 扫描节流 | 10 分钟 |
| 锁过期 | 1 小时 |

---

*本文档作为 explore.md 的补充资料。*
