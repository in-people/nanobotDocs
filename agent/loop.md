# `loop.py` 代码分析

## 概述

这是整个 nanobot 系统的核心处理引擎，负责消息分发、LLM 调用循环、工具执行和会话管理。

## 核心类

| 类 | 职责 |
|---|---|
| **AgentLoop** | 主循环引擎，管理消息处理、工具执行、会话状态 |
| **_LoopHook** | 核心钩子，处理流式输出、进度更新、工具执行 |
| **_LoopHookChain** | 钩子链，组合核心钩子和扩展钩子 |

## 关键功能

### 1. 消息处理流程 (loop.py:453-541)

```python
run() → _dispatch() → _process_message() → _run_agent_loop()
```

- **run()**: 主循环，从消息总线消费消息，创建异步任务
- **_dispatch()**: 每个会话串行处理，会话间并发执行
- **_process_message()**: 处理单条消息，构建上下文，调用 LLM
- **_run_agent_loop()**: 实际的迭代循环，处理工具调用

### 2. 流式输出支持 (loop.py:74-96)

```python
async def on_stream(self, context: AgentHookContext, delta: str) -> None:
    if self._show_thinking:
        # 显示思考内容：不过滤，直接发送所有内容
        incremental = delta
    else:
        # 不显示思考内容：过滤掉 <thinking> 标签
        prev_clean = strip_think(self._stream_buf)
        self._stream_buf += delta
        new_clean = strip_think(self._stream_buf)
        incremental = new_clean[len(prev_clean):]
```

- `show_thinking` 参数控制是否显示 `<thinking>` 标签内容
- 实时增量传输，支持分段流式会话（stream_segment）
- 流式内容通过消息总线发布为 OutboundMessage

### 3. 工具系统 (loop.py:285-310)

默认注册的工具：

| 工具类型 | 具体工具 | 功能 |
|---|---|---|
| 文件系统 | ReadFileTool, WriteFileTool, EditFileTool, ListDirTool | 文件读写编辑 |
| 搜索 | GlobTool, GrepTool | 模式匹配和内容搜索 |
| Shell | ExecTool | 命令执行 |
| Web | WebSearchTool, WebFetchTool | 网络搜索和抓取 |
| 通信 | MessageTool, SpawnTool | 消息发送和子代理生成 |
| 定时任务 | CronTool | 定时任务管理 |

工具可以配置：
- `allowed_dir`: 限制工作目录
- `restrict_to_workspace`: 仅限工作区访问
- `sandbox`: 沙箱模式

### 4. 运行时检查点 (loop.py:770-840)

中断恢复机制：

```python
def _set_runtime_checkpoint(self, session: Session, payload: dict[str, Any]) -> None:
    """持久化当前进行中的回合状态到会话元数据"""
    session.metadata[self._RUNTIME_CHECKPOINT_KEY] = payload
    self.sessions.save(session)
```

- 保存未完成的工具调用状态
- 重启时去重已保存的消息（通过 `_checkpoint_message_key` 比对）
- 未完成的工具调用标记为错误消息

### 5. 会话管理 (loop.py:737-768)

```python
def _save_turn(self, session: Session, messages: list[dict], skip: int) -> None:
    """保存新回合消息到会话，截断过大的工具结果"""
```

- 自动截断过大的工具结果（max_tool_result_chars）
- 清理易失性多模态内容（base64 图片转为占位符）
- 跳过空的 assistant 消息
- 添加时间戳记录

### 6. 并发控制 (loop.py:252-256)

```python
_MAX = int(os.environ.get("NANOBOT_MAX_CONCURRENT_REQUESTS", "3"))
self._concurrency_gate: asyncio.Semaphore | None = (
    asyncio.Semaphore(_MAX) if _MAX > 0 else None
)
```

- `NANOBOT_MAX_CONCURRENT_REQUESTS` 环境变量控制并发请求数
- 默认值为 3，≤0 表示无限制
- 每个会话内部串行（asyncio.Lock），会话间并发

### 7. 日志系统 (loop.py:276-283, 382-389, 443-449)

```python
session_id = f"agentloop-{uuid.uuid4().hex[:8]}"
config = LoggerConfig(enabled=True)
self.agent_logger = AgentLogger(session_id=session_id, config=config)
```

记录的事件：
- `LOOP_STARTED`: 循环开始
- `STREAM_START`: 流式输出开始
- `MESSAGE_RECEIVED`: 消息接收
- `LOOP_COMPLETED`: 循环完成

最终生成 HTML 报告到 `logs/traces/` 目录。

### 8. 上下文治理机制 (runner.py:98-99, 590-668)

#### 核心代码

```python
# runner.py:98-99
messages = self._apply_tool_result_budget(spec, messages)
messages_for_model = self._snip_history(spec, messages)
```

| 方法 | 作用 | 处理对象 |
|---|---|---|
| `_apply_tool_result_budget` | 截断过大的工具结果 | 工具返回的内容 |
| `_snip_history` | 裁剪历史消息 | 整个消息列表 |

#### 1. 工具结果预算 (_apply_tool_result_budget)

**代码逻辑** (runner.py:590-609):

```python
def _apply_tool_result_budget(self, spec, messages):
    updated = messages
    for idx, message in enumerate(messages):
        if message.get("role") != "tool":  # 只处理工具消息
            continue
        normalized = self._normalize_tool_result(
            spec, tool_call_id, tool_name, message.get("content")
        )
        if normalized != message.get("content"):
            if updated is messages:
                updated = [dict(m) for m in messages]  # 按需复制
            updated[idx]["content"] = normalized
    return updated
```

**处理流程**:

```
遍历消息列表
  ↓
找到 role="tool" 的消息
  ↓
调用 _normalize_tool_result()
  ↓
如果内容超过 max_tool_result_chars
  ↓
截断或持久化到文件
  ↓
返回更新后的消息列表
```

**截断策略** (runner.py:562-588):

```python
def _normalize_tool_result(self, spec, tool_call_id, tool_name, result):
    # 1. 持久化大结果到文件
    content = maybe_persist_tool_result(
        spec.workspace,
        spec.session_key,
        tool_call_id,
        result,
        max_chars=spec.max_tool_result_chars,  # 默认 100000
    )

    # 2. 如果仍然太大，直接截断
    if isinstance(content, str) and len(content) > spec.max_tool_result_chars:
        return truncate_text(content, spec.max_tool_result_chars)

    return content
```

**示例**:
```
原始工具结果: 500KB 的日志文件
     ↓
持久化到: .workspace/tool_results/<tool_call_id>.txt
     ↓
返回给 LLM: "结果已保存到文件，共 500000 字符"
```

#### 2. 历史消息裁剪 (_snip_history)

**代码逻辑** (runner.py:611-668):

```python
def _snip_history(self, spec, messages):
    # 1. 计算可用预算
    max_output = spec.max_tokens or provider_max_tokens
    budget = spec.context_block_limit or (
        spec.context_window_tokens - max_output - 1024  # 安全缓冲
    )

    # 2. 估算当前 token 数
    estimate, _ = estimate_prompt_tokens_chain(
        self.provider, spec.model, messages, spec.tools.get_definitions()
    )

    # 3. 如果未超预算，直接返回
    if estimate <= budget:
        return messages

    # 4. 分离系统消息和普通消息
    system_messages = [msg for msg in messages if msg["role"] == "system"]
    non_system = [msg for msg in messages if msg["role"] != "system"]

    # 5. 从最新消息开始倒序保留
    kept = []
    kept_tokens = 0
    for message in reversed(non_system):
        msg_tokens = estimate_message_tokens(message)
        if kept and kept_tokens + msg_tokens > remaining_budget:
            break
        kept.append(message)
        kept_tokens += msg_tokens

    # 6. 确保从 user 消息开始（完整性）
    kept.reverse()
    for i, message in enumerate(kept):
        if message["role"] == "user":
            kept = kept[i:]  # 从这个 user 消息开始
            break

    return system_messages + kept
```

**裁剪策略**:

```
原始消息: [系统] [用户1] [助手1] [用户2] [助手2] [用户3] [助手3]
                     ↓ 预算不足
第1步: 分离系统消息
       系统: [系统消息]
       其他: [用户1] [助手1] [用户2] [助手2] [用户3] [助手3]
                     ↓
第2步: 倒序保留（保留最新的）
       临时保留: [用户3] [助手3] [用户2]
                     ↓
第3步: 确保从 user 消息开始
       最终: [系统消息] [用户2] [助手2] [用户3] [助手3]
```

**为什么倒序保留？**

- **最新对话最重要**: 用户的当前请求通常依赖最近的上下文
- **保持连贯性**: 从 user 消息开始确保对话完整
- **避免孤立消息**: 不会出现只有助手没有用户的片段

#### 实际运行示例

**场景：长对话 + 大文件读取**

```python
# 初始状态
messages = [
    {"role": "system", "content": "你是助手"},  # 500 tokens
    {"role": "user", "content": "读取日志文件"},  # 50 tokens
    {"role": "assistant", "content": "正在读取...", "tool_calls": [...]},
    {"role": "tool", "content": "300KB 的日志内容"},  # 300,000 tokens ⚠️
    {"role": "user", "content": "分析错误模式"},  # 50 tokens
]

# 第1步: _apply_tool_result_budget()
messages = [
    {"role": "system", "content": "你是助手"},
    {"role": "user", "content": "读取日志文件"},
    {"role": "assistant", "content": "正在读取...", "tool_calls": [...]},
    {"role": "tool", "content": "日志已分析，发现 15 个错误..."},  # 截断到 1000 字符
    {"role": "user", "content": "分析错误模式"},
]

# 第2步: _snip_history() (假设上下文窗口 4K)
messages_for_model = [
    {"role": "system", "content": "你是助手"},
    {"role": "user", "content": "分析错误模式"},  # 只保留最新对话
]
```

#### 关键配置参数

| 参数 | 默认值 | 作用 |
|---|---|---|
| `max_tool_result_chars` | 100000 | 单个工具结果最大字符数 |
| `context_window_tokens` | 200000 | 模型上下文窗口大小 |
| `context_block_limit` | None | 自定义上下文块限制 |
| `max_tokens` | 4096 | 最大输出 token 数 |

#### 设计优势

1. **防止 OOM**: 避免超过模型上下文限制
2. **成本控制**: 减少不必要的 token 消耗
3. **性能优化**: 更小的上下文 = 更快的响应
4. **智能保留**: 优先保留最新、最相关的对话
5. **容错机制**: 裁剪失败时回退到原始消息

这套机制确保了长时间对话和大数据量场景下的稳定性。

### 9. 运行时检查点机制 (loop.py:175, 770-840)

#### 核心概念

**运行时检查点**：在 LLM 返回工具调用但工具还未执行完成时，保存当前状态到会话元数据，用于进程崩溃后的恢复。

#### 核心常量

```python
# loop.py:175
_RUNTIME_CHECKPOINT_KEY = "runtime_checkpoint"
```

#### 检查点的生命周期

##### 1. 设置检查点 (_set_runtime_checkpoint)

**代码** (loop.py:770-773):

```python
def _set_runtime_checkpoint(self, session: Session, payload: dict[str, Any]) -> None:
    """持久化当前进行中的回合状态到会话元数据"""
    session.metadata[self._RUNTIME_CHECKPOINT_KEY] = payload
    self.sessions.save(session)
```

**调用时机**：LLM 返回工具调用后，工具执行前

```python
# loop.py:155-165
await self._emit_checkpoint(
    spec,
    {
        "phase": "awaiting_tools",  # ← 等待工具执行
        "iteration": iteration,
        "assistant_message": assistant_message,
        "completed_tool_results": [],
        "pending_tool_calls": [...],  # ← 未完成的工具调用
    },
)
```

**保存的数据结构**:

```python
session.metadata["runtime_checkpoint"] = {
    "phase": "awaiting_tools",
    "iteration": 1,
    "assistant_message": {
        "role": "assistant",
        "content": "我来读取文件",
        "tool_calls": [{"id": "call_123", "type": "function", ...}]
    },
    "completed_tool_results": [],  # 已完成的工具结果
    "pending_tool_calls": [...]   # 未完成的工具调用
}
```

##### 2. 恢复检查点 (_restore_runtime_checkpoint)

**代码** (loop.py:791-840):

```python
def _restore_runtime_checkpoint(self, session: Session) -> bool:
    """将未完成的回合恢复到会话历史中"""

    checkpoint = session.metadata.get(self._RUNTIME_CHECKPOINT_KEY)
    if not isinstance(checkpoint, dict):
        return False

    # 1. 恢复 assistant ���息
    assistant_message = checkpoint.get("assistant_message")
    if isinstance(assistant_message, dict):
        restored = dict(assistant_message)
        restored.setdefault("timestamp", datetime.now().isoformat())
        restored_messages.append(restored)

    # 2. 恢复已完成的工具结果
    for message in checkpoint.get("completed_tool_results") or []:
        if isinstance(message, dict):
            restored = dict(message)
            restored.setdefault("timestamp", datetime.now().isoformat())
            restored_messages.append(restored)

    # 3. 未完成的工具调用标记为错误
    for tool_call in checkpoint.get("pending_tool_calls") or []:
        restored_messages.append({
            "role": "tool",
            "tool_call_id": tool_call.get("id"),
            "name": tool_call.get("function", {}).get("name"),
            "content": "Error: Task interrupted before this tool finished.",
            "timestamp": datetime.now().isoformat(),
        })

    # 4. 去重：避免重复添加已存在的消息
    overlap = 0
    for size in range(min(len(session.messages), len(restored_messages)), 0, -1):
        existing = session.messages[-size:]
        restored = restored_messages[:size]
        if all(self._checkpoint_message_key(l) == self._checkpoint_message_key(r)
               for l, r in zip(existing, restored)):
            overlap = size
            break

    session.messages.extend(restored_messages[overlap:])
    self._clear_runtime_checkpoint(session)  # ← 恢复后清除检查点
    return True
```

**调用时机**：新请求到达时

```python
# loop.py:624-625
if self._restore_runtime_checkpoint(session):
    self.sessions.save(session)
```

**去重机制**：

通过 `_checkpoint_message_key()` 比对消息是否已存在：

```python
@staticmethod
def _checkpoint_message_key(message: dict[str, Any]) -> tuple[Any, ...]:
    return (
        message.get("role"),
        message.get("content"),
        message.get("tool_call_id"),
        message.get("name"),
        message.get("tool_calls"),
        message.get("reasoning_content"),
        message.get("thinking_blocks"),
    )
```

##### 3. 清除检查点 (_clear_runtime_checkpoint)

**代码** (loop.py:775-777):

```python
def _clear_runtime_checkpoint(self, session: Session) -> None:
    if self._RUNTIME_CHECKPOINT_KEY in session.metadata:
        session.metadata.pop(self._RUNTIME_CHECKPOINT_KEY, None)
```

**调用时机**：

1. **正常完成时** (loop.py:613, 674):

```python
self._save_turn(session, all_msgs, 1 + len(history))
self._clear_runtime_checkpoint(session)  # ← 正常完成，清除检查点
self.sessions.save(session)
```

2. **恢复检查点后** (loop.py:839):

```python
# 在 _restore_runtime_checkpoint() 内部
self._clear_runtime_checkpoint(session)  # ← 恢复后清除
```

#### 完整流程图

```
1. LLM 返回工具调用
   ↓
2. _set_runtime_checkpoint() - 保存检查点
   session.metadata["runtime_checkpoint"] = {
       "assistant_message": {...},
       "pending_tool_calls": [...]
   }
   ↓
3. 开始执行工具...
   ↓
   [如果此时中断/崩溃]
   ↓
4. 系统重启
   ↓
5. 新请求到达
   ↓
6. _restore_runtime_checkpoint() - 恢复检查点
   - 将 assistant 消息添加到历史
   - 将 pending 标记为错误
   - 去重已存在的消息
   - 清除检查点
   ↓
7. LLM 看到完整历史（包括错误提示）
   ↓
   [如果正常完成]
   ↓
8. _save_turn() - 保存完整回合
   ↓
9. _clear_runtime_checkpoint() - 清除检查点
   session.metadata.pop("runtime_checkpoint")
```

#### 实际场景示例

**场景 1：正常完成（无中断）**

```python
# 第 1 轮迭代
LLM: "我来读取文件" + tool_call(read_file)
→ _set_runtime_checkpoint()  # 保存检查点
→ 执行 read_file 工具
→ 工具成功返回
→ _save_turn()  # 保存完整回合到会话
→ _clear_runtime_checkpoint()  # ← 清除检查点

# 第 2 轮迭代
LLM: "文件内容是..."
→ 无需恢复，正常处理
```

**场景 2：中途中断（有检查点）**

```python
# 第 1 轮迭代
LLM: "我来分析代码" + tool_call(grep)
→ _set_runtime_checkpoint()
  保存: {
    "assistant_message": {
      "role": "assistant",
      "content": "我来分析代码",
      "tool_calls": [{"id": "call_123", ...}]
    },
    "pending_tool_calls": [
      {"id": "call_123", "name": "grep"}
    ]
  }
→ 开始执行 grep...
→ [系统崩溃/进程被杀] 💥

# 系统重启后
用户发送新消息: "继续"
→ _restore_runtime_checkpoint()
→ 恢复到会话历史:
  [
    {"role": "user", "content": "分析代码"},
    {"role": "assistant", "content": "我来分析代码", "tool_calls": [...]},
    {"role": "tool", "content": "Error: Task interrupted..."}  ← 错误提示
  ]
→ _clear_runtime_checkpoint()  # ← 清除已使用的检查点

# LLM 看到的完整上下文
[
  {"role": "user", "content": "分析代码"},
  {"role": "assistant", "content": "我来分析代码", "tool_calls": [...]},
  {"role": "tool", "content": "Error: Task interrupted..."},
  {"role": "user", "content": "继续"},  ← 新消息
]

# LLM 响应
"抱歉，刚才的分析被中断了。让我重新分析..."
```

#### 设计优势

1. **容错性**: 进程崩溃后可恢复到中断点，不会丢失 LLM 响应
2. **数据完整性**: 用户能看到完整的对话历史，包括被中断的任务
3. **用户可见性**: 明确告知用户任务被中断，而不是静默失败
4. **自动清理**: 完成后自动清除，避免数据冗余
5. **幂等性**: `pop(key, None)` 多次调用安全，不会抛异常
6. **去重机制**: 避免恢复时重复添加已存在的消息

#### 关键要点

- `_set_runtime_checkpoint()`: **保存**当前进行中的状态
- `_restore_runtime_checkpoint()`: **恢复**未完成的回合（内部会调用清除）
- `_clear_runtime_checkpoint()`: **清除**已完成的检查点

三者配合实现了**完整的检查点生命周期管理**，确保系统在崩溃后能够优雅恢复，不会丢失用户数据和上下文。

## 关键数据流

```
InboundMessage → bus.consume_inbound()
              → _dispatch() [会话锁 + 并发门控]
              → _process_message()
              → _run_agent_loop()
              → AgentRunner (实际 LLM 调用)
              → OutboundMessage → bus.publish_outbound()
```

## MCP (Model Context Protocol) 支持

```python
async def _connect_mcp(self) -> None:
    """连接配置的 MCP 服务器（一次性，懒加载）"""
    if self._mcp_connected or self._mcp_connecting or not self._mcp_servers:
        return
    # ...
    await connect_mcp_servers(self._mcp_servers, self.tools, self._mcp_stack)
```

- 懒加载连接，失败可重试
- 工具自动注册到 ToolRegistry
- 关闭时优雅清理连接

## 命令系统

支持两种命令模式：

1. **优先级命令** (loop.py:475-480): 直接在主循环处理，如 `/stop`
2. **普通命令** (loop.py:627-631): 在会话上下文中处理

### 优先级命令处理详解

#### 核心代码 (loop.py:475-480)

```python
if self.commands.is_priority(raw):
    ctx = CommandContext(msg=msg, session=None, key=msg.session_key, raw=raw, loop=self)
    result = await self.commands.dispatch_priority(ctx)
    if result:
        await self.bus.publish_outbound(result)
    continue
```

#### 逐行分析

| 行 | 作用 | 说明 |
|---|---|---|
| `if self.commands.is_priority(raw)` | 检测优先级命令 | 判断用户输入是否为优先级命令（如 `/stop`） |
| `ctx = CommandContext(...)` | 构建命令上下文 | 封装消息、会话、原始输入等信息 |
| `session=None` | 无会话状态 | 优先级命令不需要会话历史 |
| `result = await self.commands.dispatch_priority(ctx)` | 执行命令 | 调用命令路由器分发处理 |
| `if result:` | 检查返回值 | 如果命令有响应结果 |
| `await self.bus.publish_outbound(result)` | 发布响应 | 将结果发送回用户 |
| `continue` | 跳过正常流程 | 不进入后续的 `_dispatch()` 处理 |

#### 为什么需要优先级命令？

**1. 紧急控制**
```python
# 示例：停止正在运行的 agent
/stop
```
- 不需要等待当前 LLM 调用完成
- 立即中断处理流程

**2. 系统级操作**
```bash
/clear           # 清空会话历史
/config set key=value  # 修改配置
/status          # 查看系统状态
```
- 这些操作不应消耗 LLM token
- 需要即时响应

**3. 绕过会话锁**

正常消息处理流程：
```
message → _dispatch() [会话锁] → _process_message()
```

优先级命令直接在主循环处理：
```
message → is_priority() → dispatch_priority() → 立即响应
```

#### 执行时机对比

```
主循环 (run())
  ├─ 消息到达
  ├─ 【优先级命令检查】← 在这里拦截
  │   └─ 是 → dispatch_priority() → continue（跳过后续）
  │   └─ 否 ↓
  ├─ 创建任务 → _dispatch()
  │   └─ 获取会话锁
  │   └─ _process_message()
  │       └─ 检查普通命令
  │       └─ 调用 LLM
```

#### 实际应用场景

**场景 1: 停止长时间运行的任务**
```python
# 用户发送 /stop
if self.commands.is_priority("/stop"):
    → 立即调用停止逻辑
    → 不需要等待当前工具调用完成
    → 返回 "Agent 已停止"
    → continue（不进入正常处理）
```

**场景 2: 会话管理**
```python
# 用户发送 /clear
if self.commands.is_priority("/clear"):
    → 清空会话历史
    → 返回 "会话已清空"
    → continue
```

#### 设计优势

1. **响应速度快**: 直接在主循环处理，无需等待会话锁
2. **资源效率**: 不消耗 LLM 调用，节省 token 和时间
3. **可控性强**: 可以中断不正常的任务或紧急操作
4. **职责分离**: 系统命令与业务逻辑分开处理

#### 与普通命令的区别

| 特性 | 优先级命令 | 普通命令 |
|---|---|---|
| 处理位置 | 主循环 (run) | 会话处理 (_process_message) |
| 需要会话锁 | 否 | 是 |
| 上下文 | 无会话历史 | 有完整会话历史 |
| 典型用途 | /stop, /clear | /help, /config |

这种设计确保了关键系统命令始终能够得到及时响应，不会因为会话繁忙或 LLM 调用延迟而被阻塞。

## 设计亮点

1. **容错性**: 中断恢复、错误处理、连接重试
2. **可观测性**: 详细日志、使用统计、HTML 报告
3. **资源控制**: 并发限制、超时控制、内容截断
4. **状态持久化**: 会话保存、检查点机制
5. **扩展性**: 钩子系统、工具注册、MCP 服务器
6. **流式体验**: 实时输出、思考控制、分段传输

## 重要配置项

| 配置 | 默认值 | 说明 |
|---|---|---|
| `max_iterations` | 20 | 最大工具调用迭代次数 |
| `max_tool_result_chars` | 100000 | 工具结果最大字符数 |
| `context_window_tokens` | 200000 | 上下文窗口大小 |
| `NANOBOT_MAX_CONCURRENT_REQUESTS` | 3 | 最大并发请求数 |

## 总结

这是一个设计完善的生产级异步处理引擎，具备完整的错误恢复、资源管理和可观测性支持。核心采用了"每会话串行、会话间并发"的设计模式，通过锁和信号量机制保证线程安全和资源控制。
