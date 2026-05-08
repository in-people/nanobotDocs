# `memory.py` 代码分析

## 概述

Memory 系统负责会话历史的持久化、归档和合并管理，确保长时间对话不会导致 token 消耗爆炸。

## 核心组件

| 组件 | 职责 |
|---|---|
| **MemoryStore** | 纯文件 I/O 层，管理 MEMORY.md、history.jsonl、SOUL.md 等文件 |
| **Consolidator** | 轻量级合并器，基于 token 预算触发会话历史归档 |
| **Dream** | 重量级记忆处理器，定时分析历史并更新记忆文件 |

## Consolidator - 会话合并机制

### 核心概念

**会话合并（Consolidation）**：当会话历史超过 token 预算时，将旧消息总结并归档到 history.jsonl，从活跃会话中移除，以控制 token 使用量。

### 调用方式

```python
# loop.py:615, 676
self._schedule_background(self.consolidator.maybe_consolidate_by_tokens(session))
```

**含义**：在后台异步检查并合并会话历史，以控制 token 使用量。

### 1. 后台任务调度器 (_schedule_background)

**代码** (loop.py:555-559):

```python
def _schedule_background(self, coro) -> None:
    """Schedule a coroutine as a tracked background task (drained on shutdown)."""
    task = asyncio.create_task(coro)  # 创建异步任务
    self._background_tasks.append(task)  # 追踪任务
    task.add_done_callback(self._background_tasks.remove)  # 完成后自动移除
```

**作用**:
- 将协程调度为后台任务，不阻塞主流程
- 追踪所有后台任务，优雅关闭时等待完成
- 任务完成后自动清理，避免内存泄漏

### 2. Token 预算合并器 (maybe_consolidate_by_tokens)

**代码** (memory.py:451-512):

```python
async def maybe_consolidate_by_tokens(self, session: Session) -> None:
    """循环归档旧消息，直到提示词符合安全预算"""

    # 1. 计算可用预算
    budget = self.context_window_tokens - self.max_completion_tokens - 1024  # 安全缓冲
    target = budget // 2  # 目标是预算的一半

    # 2. 估算当前 token 数
    estimated, source = self.estimate_session_prompt_tokens(session)

    # 3. 如果未超预算，直接返回
    if estimated < budget:
        logger.debug("Token consolidation idle: {}/{}", estimated, budget)
        return

    # 4. 循环归档，直到满足目标
    for round_num in range(self._MAX_CONSOLIDATION_ROUNDS):
        if estimated <= target:
            return

        # 找到归档边界（用户回合边界）
        boundary = self.pick_consolidation_boundary(session, estimated - target)
        if boundary is None:
            return

        # 提取要归档的消息块
        end_idx = boundary[0]
        chunk = session.messages[session.last_consolidated:end_idx]

        # 归档到 history.jsonl
        if not await self.archive(chunk):
            return

        # 更新已合并的位置
        session.last_consolidated = end_idx
        self.sessions.save(session)

        # 重新估算 token 数
        estimated, source = self.estimate_session_prompt_tokens(session)
```

### 3. 归档边界选择 (pick_consolidation_boundary)

**代码** (memory.py:380-400):

```python
def pick_consolidation_boundary(
    self,
    session: Session,
    tokens_to_remove: int,
) -> tuple[int, int] | None:
    """Pick a user-turn boundary that removes enough old prompt tokens."""
    start = session.last_consolidated
    if start >= len(session.messages) or tokens_to_remove <= 0:
        return None

    removed_tokens = 0
    last_boundary: tuple[int, int] | None = None
    for idx in range(start, len(session.messages)):
        message = session.messages[idx]
        # 在用户回合边界切分
        if idx > start and message.get("role") == "user":
            last_boundary = (idx, removed_tokens)
            if removed_tokens >= tokens_to_remove:
                return last_boundary
        removed_tokens += estimate_message_tokens(message)

    return last_boundary
```

**为什么选择用户回合边界？**

- 保持对话完整性：不会将 user-assistant 对拆开
- 避免孤立消息：确保每个回合都是完整的
- 易于理解：归档单元是完整的对话回合

### 4. 归档执行 (archive)

**代码** (memory.py:419-449):

```python
async def archive(self, messages: list[dict]) -> bool:
    """Summarize messages via LLM and append to history.jsonl."""
    if not messages:
        return False
    try:
        # 格式化消息
        formatted = MemoryStore._format_messages(messages)

        # 调用 LLM 总结
        response = await self.provider.chat_with_retry(
            model=self.model,
            messages=[
                {
                    "role": "system",
                    "content": render_template("agent/consolidator_archive.md", strip=True),
                },
                {"role": "user", "content": formatted},
            ],
            tools=None,
            tool_choice=None,
        )

        summary = response.content or "[no summary]"

        # 追加到 history.jsonl
        self.store.append_history(summary)
        return True
    except Exception:
        # 降级：直接转储原始消息
        logger.warning("Consolidation LLM call failed, raw-dumping to history")
        self.store.raw_archive(messages)
        return True
```

**降级策略**：如果 LLM 调用失败，直接将原始消息格式化后写入 history.jsonl，确保不丢失数据。

## 完整流程

```
1. 用户消息处理完成
   ↓
2. _save_turn() - 保存新回合到会话
   ↓
3. _schedule_background() - 调度后台任务
   ↓
4. maybe_consolidate_by_tokens() 在后台执行
   ↓
5. 估算当前会话的 token 数
   ↓
   [如果超过预算]
   ↓
6. 找到安全的归档边界（用户回合开始）
   ↓
7. 提取旧消息块
   ↓
8. 调用 LLM 总结消息块
   ↓
9. 将总结写入 history.jsonl
   ↓
10. 从会话中移除已归档的消息
   ↓
11. 更新 last_consolidated 标记
   ↓
12. 重新检查是否还需要合并
   ↓
   [如果仍超预算，重复 6-12]
   ↓
13. 完成
```

## 实际场景示例

### 场景：长对话的自动合并

**初始状态**:

```python
session.messages = [
    {"role": "user", "content": "介绍 Python"},  # 50 tokens
    {"role": "assistant", "content": "Python 是..."},  # 500 tokens
    {"role": "user", "content": "怎么安装"},  # 30 tokens
    {"role": "assistant", "content": "可以通过..."},  # 600 tokens
    {"role": "user", "content": "有什么特性"},  # 40 tokens
    {"role": "assistant", "content": "主要特性有..."},  # 800 tokens
    # ... 总共 50,000 tokens
]
session.last_consolidated = 0
```

**第 1 轮对话完成**:

```python
_schedule_background(consolidator.maybe_consolidate_by_tokens(session))
```

**后台执行**:

```python
# 1. 估算：50,000 tokens
# 2. 预算：200,000 - 4,096 - 1,024 = 194,880
# 3. 目标：97,440 (预算的一半)
# 4. 超预算！需要合并

# 第 1 轮合并
# 找到边界：第 3 个 user 消息（索引 4）
# 归档消息：[0:4] = 前两个回合
chunk = [
    {"role": "user", "content": "介绍 Python"},
    {"role": "assistant", "content": "Python 是..."},
    {"role": "user", "content": "怎么安装"},
    {"role": "assistant", "content": "可以通过..."},
]

# 调用 LLM 总结
await self.archive(chunk)
# LLM 返回："用户询问了 Python 的介绍和安装方法。Assistant 解释了 Python 的基本特性和安装步骤。"

# 写入 history.jsonl
# [2025-01-15 10:00] USER: 介绍 Python
# [2025-01-15 10:00] ASSISTANT: Python 是...
# [2025-01-15 10:01] USER: 怎么安装
# [2025-01-15 10:01] ASSISTANT: 可以通过...
# ↓ 归档总结
# 用户询问了 Python 的介绍和安装方法。Assistant 解释了 Python 的基本特性和安装步骤。

# 更新会话
session.last_consolidated = 4  # 标记前 4 条已归档
session.messages = session.messages[4:]  # 移除已归档的消息

# 重新估算：25,000 tokens ✓ 符合目标
```

**合并后的效果**:

```python
# 合并前
# 会话历史：50,000 tokens
[
  用户1, 助手1, 用户2, 助手2, 用户3, 助手3, ..., 用户50, 助手50
]
# 每次请求都要发送 50,000 tokens 给 LLM

# 合并后
# 会话历史：25,000 tokens
[
  用户3, 助手3, ..., 用户50, 助手50
]

# history.jsonl（归档文件）：
[
  "用户询问了 Python 的介绍和安装方法。Assistant 解释了 Python 的基本特性和安装步骤。",
  "用户询问了 Python 的数据类型和变量。Assistant 介绍了整数、浮点数、字符串等基本类型...",
  "用户询问了 Python 的控制流和函数。Assistant 讲解了 if 语句、for 循环和函数定义...",
  ...
]
```

## 为什么是后台任务？

### 1. 不阻塞响应

```python
# 主流程
return OutboundMessage(content="回答内容")  # 立即返回给用户

# 后台流程
consolidator.maybe_consolidate_by_tokens(session)  # 异步执行
```

用户不需要等待合并完成，立即收到响应。

### 2. 避免重复合并

```python
# 使用会话锁
lock = self.get_lock(session.key)
async with lock:
    # 同一会话的合并请求串行执行
    await self.consolidator.maybe_consolidate_by_tokens(session)
```

### 3. 优雅关闭

```python
# loop.py:543-547
async def close_mcp(self):
    if self._background_tasks:
        await asyncio.gather(*self._background_tasks, return_exceptions=True)
        self._background_tasks.clear()
```

关闭时等待所有后台任务完成。

## MemoryStore - 文件存储层

### 文件结构

```
workspace/
├── memory/
│   ├── MEMORY.md          # 长期记忆文件
│   ├── history.jsonl      # 归档的会话历史
│   ├── .cursor            # 最后处理的位置
│   └── .dream_cursor      # Dream 处理的游标
├── SOUL.md                # Agent 个性/灵魂
└── USER.md                # 用户偏好/信息
```

### 关键方法

| 方法 | 作用 |
|---|---|
| `append_history()` | 追加归档条目到 history.jsonl |
| `read_file()` | 读取文件内容 |
| `raw_archive()` | 降级归档：直接转储原始消息 |

## 关键配置参数

| 参数 | 默认值 | 说明 |
|---|---|---|
| `context_window_tokens` | 200,000 | 模型上下文窗口大小 |
| `max_completion_tokens` | 4,096 | 最大输出 token 数 |
| `_SAFETY_BUFFER` | 1,024 | 安全缓冲（tokenizer 估算偏差） |
| `_MAX_CONSOLIDATION_ROUNDS` | 5 | 最大合并轮次 |
| `target` | budget // 2 | 目标是预算的一半（留出增长空间） |
| `max_history_entries` | 1,000 | 历史记录最大条目数 |

## 设计优势

1. **自动管理**: 无需手动清理会话历史
2. **非阻塞**: 后台执行，不影响用户体验
3. **智能归档**: 只在超预算时才合并，避免不必要的 LLM 调用
4. **安全边界**: 在用户回合边界合并，保持对话完整性
5. **持久化**: 归档内容保存到 history.jsonl，可追溯
6. **渐进式**: 分轮合并，避免一次性处理大量数据
7. **容错降级**: LLM 失败时自动降级为原始转储
8. **会话锁**: 避免同会话并发合并冲突

## 与其他组件的协作

```
AgentLoop
  ↓ 使用
Consolidator
  ↓ 使用
MemoryStore
  ↓ 写入
history.jsonl (归档文件)
```

**流程**:
1. `AgentLoop` 在每次对话完成后调度合并任务
2. `Consolidator` 检查 token 预算，决定是否需要合并
3. `MemoryStore` 提供文件 I/O 接口，写入归档内容
4. 归档内容持久化到 `history.jsonl`，可被 `Dream` 进一步处理

## 总结

`Consolidator` 是一个**智能的会话历史管理机制**，它：

- **自动监控**会话 token 数量
- **后台异步**执行合并，不阻塞用户
- **智能归档**旧对话到 history.jsonl
- **保持会话**在合理的大小范围内
- **完整保留**历史对话可追溯性

这确保了长时间对话不会导致 token 消耗爆炸，同时保留了完整的对话历史可追溯性。配合 `Dream` 组件，可以将归档的历史进一步提炼为长期记忆（MEMORY.md）。
