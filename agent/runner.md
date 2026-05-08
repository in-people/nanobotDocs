# AgentRunner 代码分析文档

> **文件路径**: `nanobot/agent/runner.py`  
> **主要功能**: 工具使用型 LLM 代理的共享执行循环  
> **核心类**: `AgentRunner`, `AgentRunSpec`, `AgentRunResult`

---

## 📋 目录

1. [整体架构](#整体架构)
2. [核心数据结构](#核心数据结构)
3. [执行流程分析](#执行流程分析)
4. [关键特性](#关键特性)
5. [错误处理机制](#错误处理机制)
6. [性能优化策略](#性能优化策略)
7. [设计模式与最佳实践](#设计模式与最佳实践)
8. [使用场景](#使用场景)
9. [代码质量评估](#代码质量评估)

---

## 🏗️ 整体架构

### 设计理念
`AgentRunner` 是一个**产品层无关的纯执行引擎**，专注于：
- 🔧 **工具执行循环**: 管理 LLM 与工具的交互循环
- 🔄 **状态管理**: 追踪对话历史、工具使用、token 消耗
- 🎯 **生命周期管理**: 通过 **Hook 系统提供扩展点**
- 📊 **可观测性**: 完整的日志和事件追踪

### 架构层次

```
┌─────────────────────────────────────────┐
│         产品层 (Product Layer)           │
│    (业务逻辑、用户界面、特定领域逻辑)      │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│         AgentRunner (执行层)             │
│  ┌─────────────┐  ┌─────────────────┐  │
│  │ AgentRunSpec│  │AgentRunResult   │  │
│  └─────────────┘  └─────────────────┘  │
│  ┌──────────────────────────────────┐  │
│  │   主执行循环 (run 方法)           │  │
│  │  - 模型请求                       │  │
│  │  - 工具执行                       │  │
│  │  - 错误处理                       │  │
│  │  - 状态管理                       │  │
│  └──────────────────────────────────┘  │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│         基础设施层 (Infrastructure)      │
│  - LLM Provider                         │
│  - Tool Registry                        │
│  - Hook System                          │
│  - Logger                               │
└─────────────────────────────────────────┘
```

---

## 📊 核心数据结构

### 1. AgentRunSpec - 运行配置

```python
@dataclass(slots=True)
class AgentRunSpec:
    """单次 Agent 执行的配置"""
    
    # 核心配置
    initial_messages: list[dict[str, Any]]  # 初始对话历史
    tools: ToolRegistry                     # 工具注册表
    model: str                              # 模型名称
    max_iterations: int                     # 最大迭代次数
    
    # Token 管理
    max_tool_result_chars: int              # 工具结果最大字符数
    context_window_tokens: int | None       # 上下文窗口大小
    context_block_limit: int | None         # 上下文块限制
    
    # 模型参数
    temperature: float | None               # 温度参数
    max_tokens: int | None                  # 最大生成 tokens
    reasoning_effort: str | None            # 推理强度 (Claude 特有)
    
    # 执行控制
    concurrent_tools: bool                  # 是否并发执行工具
    fail_on_tool_error: bool               # 工具错误时是否失败
    provider_retry_mode: str               # 提供者重试模式
    
    # 扩展机制
    hook: AgentHook | None                 # 生命周期钩子
    progress_callback: Any | None          # 进度回调
    checkpoint_callback: Any | None        # 检查点回调
    agent_logger: Any                      # Agent 日志记录器
    
    # 错误处理
    error_message: str | None              # 自定义错误消息
    max_iterations_message: str | None     # 最大迭代消息
    
    # 工作区
    workspace: Path | None                 # 工作区路径
    session_key: str | None                # 会话标识
```

**设计要点**:
- 使用 `slots=True` 优化内存使用
- 大量可选参数支持灵活配置
- 清晰的参数分组（核心、Token、模型、控制等）

### 2. AgentRunResult - 运行结果

```python
@dataclass(slots=True)
class AgentRunResult:
    """Agent 执行的结果"""
    
    # 输出
    final_content: str | None              # 最终响应内容
    messages: list[dict[str, Any]]         # 完整对话历史
    
    # 统计
    tools_used: list[str]                  # 使用的工具列表
    usage: dict[str, int]                  # Token 使用统计
    stop_reason: str                       # 停止原因
    
    # 错误追踪
    error: str | None                      # 错误信息
    tool_events: list[dict[str, str]]     # 工具执行事件
```

**停止原因类型**:
- `completed`: 正常完成
- `error`: 模型错误
- `tool_error`: 工具执行错误
- `empty_final_response`: 空响应
- `max_iterations`: 达到最大迭代次数

---

## 🔄 执行流程分析

### 主循环 (run 方法)

```python
async def run(self, spec: AgentRunSpec) -> AgentRunResult:
    """执行 Agent 主循环"""
    
    # 初始化
    hook = spec.hook or AgentHook()
    messages = list(spec.initial_messages)
    usage = {"prompt_tokens": 0, "completion_tokens": 0}
    external_lookup_counts = {}
    
    # 主循环
    for iteration in range(spec.max_iterations):
        # 1. 上下文管理
        messages = self._apply_tool_result_budget(spec, messages)
        messages_for_model = self._snip_history(spec, messages)
        
        # 2. 日志记录
        if spec.agent_logger:
            spec.agent_logger.log_event("loop_iteration_start", {
                "iteration": iteration,
                "messages_count": len(messages_for_model),
                "messages": truncate_messages_for_log(messages_for_model),
            }, step=iteration + 1)
        
        # 3. Hook: 迭代前
        context = AgentHookContext(iteration=iteration, messages=messages)
        await hook.before_iteration(context)
        
        # 4. 请求模型
        response = await self._request_model(spec, messages_for_model, hook, context)
        self._accumulate_usage(usage, response.usage)
        
        # 5. 记录模型输出
        if spec.agent_logger:
            log_model_output(spec.agent_logger, response, iteration + 1)
        
        # 6. 处理工具调用
        if response.has_tool_calls:
            # 6.1 构建助手消息
            assistant_message = build_assistant_message(
                response.content or "",
                tool_calls=[tc.to_openai_tool_call() for tc in response.tool_calls],
                reasoning_content=response.reasoning_content,
                thinking_blocks=response.thinking_blocks,
            )
            messages.append(assistant_message)
            
            # 6.2 执行工具
            await hook.before_execute_tools(context)
            results, new_events, fatal_error = await self._execute_tools(
                spec, response.tool_calls, external_lookup_counts
            )
            
            # 6.3 处理工具错误
            if fatal_error:
                error = f"Error: {type(fatal_error).__name__}: {fatal_error}"
                final_content = error
                stop_reason = "tool_error"
                break
            
            # 6.4 添加工具结果消息
            for tool_call, result in zip(response.tool_calls, results):
                tool_message = {
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "name": tool_call.name,
                    "content": self._normalize_tool_result(spec, tool_call.id, tool_call.name, result),
                }
                messages.append(tool_message)
            
            # 6.5 Hook: 迭代后
            await hook.after_iteration(context)
            continue  # 继续下一轮迭代
        
        # 7. 处理最终响应
        final_content = hook.finalize_content(context, response.content)
        
        # 7.1 空响应重试
        if is_blank_text(final_content):
            response = await self._request_finalization_retry(spec, messages_for_model)
            final_content = hook.finalize_content(context, response.content)
        
        # 7.2 错误处理
        if response.finish_reason == "error":
            final_content = final_content or spec.error_message
            stop_reason = "error"
            break
        
        # 7.3 正常完成
        messages.append(build_assistant_message(final_content))
        break
    
    # 8. 处理最大迭代次数
    else:
        stop_reason = "max_iterations"
        final_content = spec.max_iterations_message or render_template("agent/max_iterations_message.md")
    
    return AgentRunResult(
        final_content=final_content,
        messages=messages,
        tools_used=tools_used,
        usage=usage,
        stop_reason=stop_reason,
        error=error,
        tool_events=tool_events,
    )
```

### 流程图

```
┌─────────────────────────────────────────────────────┐
│                    开始                              │
└──────────────────┬──────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────┐
│  初始化: hook, messages, usage, counters             │
└──────────────────┬──────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────┐
│  主循环 (iteration < max_iterations)                 │
└──────────────────┬──────────────────────────────────┘
                   │
       ┌───────────┴───────────┐
       │                       │
       ▼                       ▼
┌─────────────────┐   ┌─────────────────┐
│ 上下文管理       │   │ 日志记录        │
│ - Token 预算    │   │ - 迭代开始      │
│ - 历史裁剪      │   │ - 消息记录      │
└────────┬────────┘   └────────┬────────┘
         │                     │
         └──────────┬──────────┘
                    │
                    ▼
         ┌──────────────────────┐
         │ Hook: before_iteration│
         └──────────┬───────────┘
                    │
                    ▼
         ┌──────────────────────┐
         │   请求 LLM           │
         │   - 流式/非流式       │
         └──────────┬───────────┘
                    │
         ┌──────────┴──────────┐
         │                     │
         ▼                     ▼
  ┌─────────────┐      ┌─────────────┐
  │ 有工具调用?  │      │  无工具调用  │
  └──────┬──────┘      └──────┬──────┘
         │                    │
         ▼                    ▼
  ┌─────────────┐      ┌─────────────┐
  │ 执行工具     │      │ 处理最终响应 │
  │ - 并发/串行  │      │ - 空响应重试 │
  │ - 错误处理   │      │ - 错误处理  │
  │ - 结果归一化 │      │ - 完成循环  │
  └──────┬──────┘      └──────┬──────┘
         │                    │
         │                    ▼
         │            ┌─────────────┐
         │            │ Hook:       │
         │            │ on_stream_end│
         │            └──────┬──────┘
         │                   │
         └──────────┬────────┘
                    │
                    ▼
         ┌──────────────────────┐
         │ Hook: after_iteration│
         └──────────┬───────────┘
                    │
                    ▼
         ┌──────────────────────┐
         │  继续下一轮 / 完成    │
         └──────────────────────┘
```

---

## 🎯 关键特性

### 1. 智能上下文管理

#### _snip_history - 历史消息裁剪

```python
def _snip_history(self, spec: AgentRunSpec, messages: list[dict[str, Any]]) -> list[dict[str, Any]]:
    """智能裁剪历史消息以适应上下文窗口"""
    
    # 1. 计算预算
    provider_max_tokens = getattr(getattr(self.provider, "generation", None), "max_tokens", 4096)
    max_output = spec.max_tokens or provider_max_tokens or 4096
    budget = spec.context_block_limit or (spec.context_window_tokens - max_output - _SNIP_SAFETY_BUFFER)
    
    # 2. 估算当前 tokens
    estimate, _ = estimate_prompt_tokens_chain(
        self.provider, spec.model, messages, spec.tools.get_definitions()
    )
    
    # 3. 如果在预算内，不裁剪
    if estimate <= budget:
        return messages
    
    # 4. 分离系统消息和非系统消息
    system_messages = [dict(msg) for msg in messages if msg.get("role") == "system"]
    non_system = [dict(msg) for msg in messages if msg.get("role") != "system"]
    
    # 5. 计算系统消息 tokens
    system_tokens = sum(estimate_message_tokens(msg) for msg in system_messages)
    remaining_budget = max(128, budget - system_tokens)
    
    # 6. 从最新消息开始保留
    kept: list[dict[str, Any]] = []
    kept_tokens = 0
    for message in reversed(non_system):
        msg_tokens = estimate_message_tokens(message)
        if kept and kept_tokens + msg_tokens > remaining_budget:
            break
        kept.append(message)
        kept_tokens += msg_tokens
    kept.reverse()
    
    # 7. 确保从用户消息开始
    if kept:
        for i, message in enumerate(kept):
            if message.get("role") == "user":
                kept = kept[i:]
                break
        start = find_legal_message_start(kept)
        if start:
            kept = kept[start:]
    
    # 8. 保留最少 4 条消息
    if not kept:
        kept = non_system[-min(len(non_system), 4):]
        start = find_legal_message_start(kept)
        if start:
            kept = kept[start:]
    
    return system_messages + kept
```

**裁剪策略**:
- ✅ 保留所有系统消息
- ✅ 从最新消息开始保留
- ✅ 确保对话连贯性（从用户消息开始）
- ✅ 最少保留 4 条消息
- ✅ 预留 1024 tokens 安全缓冲

#### _apply_tool_result_budget - 工具结果预算

```python
def _apply_tool_result_budget(self, spec: AgentRunSpec, messages: list[dict[str, Any]]) -> list[dict[str, Any]]:
    """应用工具结果大小限制"""
    
    updated = messages
    for idx, message in enumerate(messages):
        if message.get("role") != "tool":
            continue
        
        normalized = self._normalize_tool_result(
            spec,
            str(message.get("tool_call_id") or f"tool_{idx}"),
            str(message.get("name") or "tool"),
            message.get("content"),
        )
        
        if normalized != message.get("content"):
            if updated is messages:
                updated = [dict(m) for m in messages]
            updated[idx]["content"] = normalized
    
    return updated
```

### 2. 工具执行管理

#### _execute_tools - 工具执行协调

```python
async def _execute_tools(
    self,
    spec: AgentRunSpec,
    tool_calls: list[ToolCallRequest],
    external_lookup_counts: dict[str, int],
) -> tuple[list[Any], list[dict[str, str]], BaseException | None]:
    """执行工具调用，支持并发和错误处理"""
    
    # 1. 分批处理
    batches = self._partition_tool_batches(spec, tool_calls)
    tool_results: list[tuple[Any, dict[str, str], BaseException | None]] = []
    
    # 2. 执行每批工具
    for batch in batches:
        if spec.concurrent_tools and len(batch) > 1:
            # 并发执行
            tool_results.extend(await asyncio.gather(*(
                self._run_tool(spec, tool_call, external_lookup_counts)
                for tool_call in batch
            )))
        else:
            # 串行执行
            for tool_call in batch:
                tool_results.append(await self._run_tool(spec, tool_call, external_lookup_counts))
    
    # 3. 汇总结果
    results: list[Any] = []
    events: list[dict[str, str]] = []
    fatal_error: BaseException | None = None
    
    for result, event, error in tool_results:
        results.append(result)
        events.append(event)
        if error is not None and fatal_error is None:
            fatal_error = error
    
    return results, events, fatal_error
```

#### _partition_tool_batches - 工具分批

  ┌──────────┬─────────────────────────────┬───────────────────────────────────┬──────────────────────┬───────────┐
  │   场景   │          工具列表           │             分批结果              │       执行方式       │ 性能提升  │
  ├──────────┼─────────────────────────────┼───────────────────────────────────┼──────────────────────┼───────────┤
  │ 混合工具 │ [search, grep, write, read] │ [[search, grep], [write], [read]] │ safe并发，unsafe串行 │ ⚡ 高     │
  ├──────────┼─────────────────────────────┼───────────────────────────────────┼──────────────────────┼───────────┤
  │ 全safe   │ [search, grep, read]        │ [[search, grep, read]]            │ 全部并发             │ 🚀 最高   │
  ├──────────┼─────────────────────────────┼───────────────────────────────────┼──────────────────────┼───────────┤
  │ 全unsafe │ [write, delete]             │ [[write], [delete]]               │ 全部串行             │ ➡️  无提升 │
  ├──────────┼─────────────────────────────┼───────────────────────────────────┼──────────────────────┼───────────┤
  │ 禁用并发 │ [search, grep]              │ [[search], [grep]]                │ 全部串行             │ ➡️  无提升 │
  └──────────┴─────────────────────────────┴───────────────────────────────────┴──────────────────────┴───────────┘

```python
def _partition_tool_batches(
    self,
    spec: AgentRunSpec,
    tool_calls: list[ToolCallRequest],
) -> list[list[ToolCallRequest]]:
    """将工具调用分批，支持并发安全的工具"""
    
    if not spec.concurrent_tools:
        return [[tool_call] for tool_call in tool_calls]
    
    batches: list[list[ToolCallRequest]] = []
    current: list[ToolCallRequest] = []
    
    for tool_call in tool_calls:
        get_tool = getattr(spec.tools, "get", None)
        tool = get_tool(tool_call.name) if callable(get_tool) else None
        can_batch = bool(tool and tool.concurrency_safe)
        
        if can_batch:
            current.append(tool_call)
            continue
        
        # 不能并发的工具单独一批
        if current:
            batches.append(current)
            current = []
        batches.append([tool_call])
    
    if current:
        batches.append(current)
    
    return batches
```

**分批策略**:
- ✅ 并发安全的工具可以批量执行
- ✅ 不安全的工具单独执行
- ✅ 保持工具调用顺序

#### _run_tool - 单个工具执行

```python
async def _run_tool(
    self,
    spec: AgentRunSpec,
    tool_call: ToolCallRequest,
    external_lookup_counts: dict[str, int],
) -> tuple[Any, dict[str, str], BaseException | None]:
    """执行单个工具调用"""
    
    _HINT = "\n\n[Analyze the error above and try a different approach.]"
    
    # 1. 检查重复外部查找
    lookup_error = repeated_external_lookup_error(
        tool_call.name, tool_call.arguments, external_lookup_counts
    )
    if lookup_error:
        event = {
            "name": tool_call.name,
            "status": "error",
            "detail": "repeated external lookup blocked",
        }
        if spec.fail_on_tool_error:
            return lookup_error + _HINT, event, RuntimeError(lookup_error)
        return lookup_error + _HINT, event, None
    
    # 2. 准备工具调用
    # 安全获取prepare_call 方法
    prepare_call = getattr(spec.tools, "prepare_call", None)
    tool, params, prep_error = None, tool_call.arguments, None
    
    if callable(prepare_call):
        try:
            prepared = prepare_call(tool_call.name, tool_call.arguments)
            if isinstance(prepared, tuple) and len(prepared) == 3:
                tool, params, prep_error = prepared
        except Exception:
            pass
    
    # 3. 处理准备错误
    if prep_error:
        event = {
            "name": tool_call.name,
            "status": "error",
            "detail": prep_error.split(": ", 1)[-1][:120],
        }
        return prep_error + _HINT, event, RuntimeError(prep_error) if spec.fail_on_tool_error else None
    
    # 4. 执行工具
    try:
        if tool is not None:
            result = await tool.execute(**params)
        else:
            result = await spec.tools.execute(tool_call.name, params)
    except asyncio.CancelledError:
        raise
    except BaseException as exc:
        event = {
            "name": tool_call.name,
            "status": "error",
            "detail": str(exc),
        }
        if spec.fail_on_tool_error:
            return f"Error: {type(exc).__name__}: {exc}", event, exc
        return f"Error: {type(exc).__name__}: {exc}", event, None
    
    # 5. 处理字符串错误
    if isinstance(result, str) and result.startswith("Error"):
        event = {
            "name": tool_call.name,
            "status": "error",
            "detail": result.replace("\n", " ").strip()[:120],
        }
        if spec.fail_on_tool_error:
            return result + _HINT, event, RuntimeError(result)
        return result + _HINT, event, None
    
    # 6. 成功返回
    detail = "" if result is None else str(result)
    detail = detail.replace("\n", " ").strip()
    if not detail:
        detail = "(empty)"
    elif len(detail) > 120:
        detail = detail[:120] + "..."
    
    return result, {"name": tool_call.name, "status": "ok", "detail": detail}, None
```

### 3. Hook 系统

#### Hook 生命周期

```python
# 1. 迭代前
await hook.before_iteration(context)

# 2. 流式输出中
await hook.on_stream(context, delta)

# 3. 流式结束（工具调用前）
await hook.on_stream_end(context, resuming=True)

# 4. 工具执行前
await hook.before_execute_tools(context)

# 5. 迭代后
await hook.after_iteration(context)

# 6. 流式结束（最终响应）
await hook.on_stream_end(context, resuming=False)
```

#### AgentHookContext

```python
class AgentHookContext:
    """Hook 上下文，传递完整的执行状态"""
    
    iteration: int                              # 当前迭代
    messages: list[dict[str, Any]]             # 消息历史
    response: ToolCallRequest | None = None    # 模型响应
    usage: dict[str, int] = field(default_factory=dict)  # Token 使用
    tool_calls: list[ToolCallRequest] = field(default_factory=list)  # 工具调用
    tool_results: list[Any] = field(default_factory=list)  # 工具结果
    tool_events: list[dict[str, str]] = field(default_factory=list)  # 工具事件
    final_content: str | None = None           # 最终内容
    error: str | None = None                   # 错误信息
    stop_reason: str = "completed"            # 停止原因
```

### 4. 日志系统

#### 日志事件类型

```python
# 1. 迭代开始
spec.agent_logger.log_event("loop_iteration_start", {
    "iteration": iteration,
    "max_iterations": spec.max_iterations,
    "messages_count": len(messages_for_model),
    "messages": truncate_messages_for_log(messages_for_model, 400),
    "session_key": spec.session_key,
}, step=iteration + 1)

# 2. 迭代结束
spec.agent_logger.log_event("loop_iteration_end", {
    "iteration": iteration,
    "duration_ms": duration_ms,
    "tool_calls_count": len(response.tool_calls),
    "stop_reason": stop_reason,
    "error": error,
}, step=iteration + 1)

# 3. 模型输出
log_model_output(spec.agent_logger, response, iteration + 1)
```

#### 日志特性

- **自动截断**: 防止日志过大
- **结构化**: JSON 格式，便于分析
- **分步骤**: 每个迭代都有明确标识
- **性能监控**: 记录执行时间
- **错误追踪**: 详细的错误信息

---

## ⚠️ 错误处理机制

### 1. 工具错误处理

```python
# 重复外部查找
lookup_error = repeated_external_lookup_error(
    tool_call.name, tool_call.arguments, external_lookup_counts
)
if lookup_error:
    if spec.fail_on_tool_error:
        return lookup_error + _HINT, event, RuntimeError(lookup_error)
    return lookup_error + _HINT, event, None

# 工具执行异常
except BaseException as exc:
    event = {"name": tool_call.name, "status": "error", "detail": str(exc)}
    if spec.fail_on_tool_error:
        return f"Error: {type(exc).__name__}: {exc}", event, exc
    return f"Error: {type(exc).__name__}: {exc}", event, None

# 字符串错误结果
if isinstance(result, str) and result.startswith("Error"):
    if spec.fail_on_tool_error:
        return result + _HINT, event, RuntimeError(result)
    return result + _HINT, event, None
```

**策略**:
- **宽容模式** (`fail_on_tool_error=False`): 记录错误但继续执行
- **严格模式** (`fail_on_tool_error=True`): 立即终止并返回错误
- **重复防护**: 防止重复的外部查找

### 2. 模型错误处理

```python
if response.finish_reason == "error":
    final_content = clean or spec.error_message or _DEFAULT_ERROR_MESSAGE
    stop_reason = "error"
    error = final_content
    self._append_final_message(messages, final_content)
    break
```

### 3. 空响应处理

```python
if is_blank_text(clean):
    logger.warning("Empty final response on turn {}; retrying with explicit finalization prompt")
    response = await self._request_finalization_retry(spec, messages_for_model)
    retry_usage = self._usage_dict(response.usage)
    self._accumulate_usage(usage, retry_usage)
    clean = hook.finalize_content(context, response.content)

if is_blank_text(clean):
    final_content = EMPTY_FINAL_RESPONSE_MESSAGE
    stop_reason = "empty_final_response"
    break
```

### 4. 最大迭代次数处理

```python
for iteration in range(spec.max_iterations):
    # ... 主循环逻辑
else:
    # 循环正常结束（未 break）
    stop_reason = "max_iterations"
    if spec.max_iterations_message:
        final_content = spec.max_iterations_message.format(max_iterations=spec.max_iterations)
    else:
        final_content = render_template("agent/max_iterations_message.md", strip=True, max_iterations=spec.max_iterations)
    self._append_final_message(messages, final_content)
```

---

## 🚀 性能优化策略

### 1. 并发工具执行

```python
if spec.concurrent_tools and len(batch) > 1:
    tool_results.extend(await asyncio.gather(*(
        self._run_tool(spec, tool_call, external_lookup_counts)
        for tool_call in batch
    )))
```

**优势**:
- ⚡ 减少等待时间
- 📈 提高吞吐量
- 🔄 适合独立的工具调用

**限制**:
- 🔒 只并发安全的工具
- 📊 保持工具调用顺序

### 2. Token 管理

#### 上下文窗口优化
```python
budget = spec.context_block_limit or (spec.context_window_tokens - max_output - _SNIP_SAFETY_BUFFER)
```

#### 工具结果限制
```python
if isinstance(content, str) and len(content) > spec.max_tool_result_chars:
    return truncate_text(content, spec.max_tool_result_chars)
```

#### 消息裁剪
```python
# 从最新消息开始保留
for message in reversed(non_system):
    msg_tokens = estimate_message_tokens(message)
    if kept and kept_tokens + msg_tokens > remaining_budget:
        break
    kept.append(message)
    kept_tokens += msg_tokens
```

### 3. 内存优化

#### 使用 slots
```python
@dataclass(slots=True)
class AgentRunSpec:
    ...
```

#### 懒拷贝
```python
if normalized != message.get("content"):
    if updated is messages:
        updated = [dict(m) for m in messages]  # 只在需要时拷贝
    updated[idx]["content"] = normalized
```

### 4. 资源管理

#### 工具结果持久化
```python
content = maybe_persist_tool_result(
    spec.workspace,
    spec.session_key,
    tool_call_id,
    result,
    max_chars=spec.max_tool_result_chars,
)
```

#### Token 使用统计
```python
def _accumulate_usage(target: dict[str, int], addition: dict[str, int]) -> None:
    for key, value in addition.items():
        target[key] = target.get(key, 0) + value
```

---

## 🎨 设计模式与最佳实践

### 1. 关注点分离

```python
# 产品层无关的纯执行逻辑
class AgentRunner:
    """Run a tool-capable LLM loop without product-layer concerns."""
```

**优势**:
- 🔄 可复用于不同产品
- 🧪 易于测试
- 📦 清晰的职责边界

### 2. 依赖注入

```python
def __init__(self, provider: LLMProvider):
    self.provider = provider
```

**优势**:
- 🔌 灵活替换提供者
- 🧪 易于 mock 测试
- 📦 降低耦合

### 3. Hook 模式

```python
hook = spec.hook or AgentHook()
await hook.before_iteration(context)
await hook.after_iteration(context)
```

**优势**:
- 🔧 扩展点清晰
- 🎭 生命周期管理
- 📊 可观测性

### 4. 策略模式

```python
# 错误处理策略
if spec.fail_on_tool_error:
    return result + _HINT, event, RuntimeError(result)
return result + _HINT, event, None

# 执行策略
if spec.concurrent_tools and len(batch) > 1:
    tool_results.extend(await asyncio.gather(...))
else:
    for tool_call in batch:
        tool_results.append(await self._run_tool(...))
```

### 5. 建造者模式

```python
@dataclass
class AgentRunSpec:
    """配置对象，支持灵活构建"""
    initial_messages: list[dict[str, Any]]
    tools: ToolRegistry
    model: str
    max_iterations: int
    # ... 大量可选配置
```

### 6. 防御性编程

```python
# 检查所有可能的错误
try:
    messages = self._apply_tool_result_budget(spec, messages)
    messages_for_model = self._snip_history(spec, messages)
except Exception as exc:
    logger.warning("Context governance failed: {}; using raw messages", exc)
    messages_for_model = messages

# 安全的默认值
provider_max_tokens = getattr(getattr(self.provider, "generation", None), "max_tokens", 4096)

# 类型检查
if isinstance(prepared, tuple) and len(prepared) == 3:
    tool, params, prep_error = prepared
```

---

## 💼 使用场景

### 1. 代码审查 Agent

```python
spec = AgentRunSpec(
    initial_messages=[
        {"role": "system", "content": "You are a code reviewer."},
        {"role": "user", "content": "Review this PR: ..."}
    ],
    tools=ToolRegistry([
        FileReadTool(),
        GitDiffTool(),
        CodeAnalysisTool(),
    ]),
    model="claude-sonnet-4-6",
    max_iterations=10,
    concurrent_tools=True,
    fail_on_tool_error=False,
    agent_logger=AgentLogger("code_review_session"),
)

runner = AgentRunner(provider)
result = await runner.run(spec)
```

### 2. 研究助手

```python
spec = AgentRunSpec(
    initial_messages=[
        {"role": "system", "content": "You are a research assistant."},
        {"role": "user", "content": "Research the latest AI trends."}
    ],
    tools=ToolRegistry([
        WebSearchTool(),
        WebFetchTool(),
        ArxivSearchTool(),
    ]),
    model="claude-sonnet-4-6",
    max_iterations=15,
    concurrent_tools=True,
    context_window_tokens=200000,
    agent_logger=AgentLogger("research_session"),
)
```

### 3. 自动化工作流

```python
spec = AgentRunSpec(
    initial_messages=[
        {"role": "system", "content": "You automate DevOps tasks."},
        {"role": "user", "content": "Deploy the application."}
    ],
    tools=ToolRegistry([
        DockerBuildTool(),
        K8sDeployTool(),
        HealthCheckTool(),
    ]),
    model="claude-sonnet-4-6",
    max_iterations=20,
    fail_on_tool_error=True,  # 严格模式
    checkpoint_callback=save_checkpoint,
)
```

---

## 📈 代码质量评估

### 优点

✅ **架构设计**
- 清晰的层次结构
- 良好的关注点分离
- 高度可扩展

✅ **错误处理**
- 多层次的错误恢复
- 详细的错误信息
- 优雅的降级策略

✅ **性能优化**
- 智能的并发控制
- 有效的资源管理
- 内存使用优化

✅ **可观测性**
- 完整的日志系统
- 详细的事件追踪
- 性能监控

✅ **代码质量**
- 类型注解完整
- 文档字符串清晰
- 遵循 Python 最佳实践

### 改进建议

🔧 **潜在优化**

1. **类型安全**
```python
# 当前
agent_logger: Any = None

# 建议
from typing import Protocol
class AgentLogger(Protocol):
    def log_event(self, event_type: str, data: dict[str, Any], step: int) -> None: ...
agent_logger: AgentLogger | None = None
```

2. **常量管理**
```python
# 当前
detail[:120]

# 建议
MAX_DETAIL_LENGTH = 120
detail[:MAX_DETAIL_LENGTH]
```

3. **错误分类**
```python
# 当前
BaseException

# 建议
定义更具体的异常层次
class AgentError(Exception): ...
class ToolExecutionError(AgentError): ...
class ModelRequestError(AgentError): ...
```

4. **配置验证**
```python
# 添加配置验证
def _validate_spec(spec: AgentRunSpec) -> None:
    if spec.max_iterations <= 0:
        raise ValueError("max_iterations must be positive")
    if spec.max_tool_result_chars <= 0:
        raise ValueError("max_tool_result_chars must be positive")
```

---

## 📚 总结

`AgentRunner` 是一个**生产级别的工具使用型 LLM 代理执行框架**，具有以下特点：

### 核心优势

1. **🏗️ 架构优秀**: 清晰的层次结构，关注点分离
2. **🔧 高度可扩展**: Hook 系统支持灵活扩展
3. **📊 可观测性强**: 完整的日志和事件追踪
4. **⚡ 性能优化**: 智能的并发控制和资源管理
5. **🛡️ 错误处理**: 多层次的错误恢复机制
6. **🧪 易于测试**: 依赖注入和清晰的接口

### 适用场景

- ✅ 需要多轮工具调用的复杂任务
- ✅ 需要外部数据查找和整合
- ✅ 需要可靠的重试和错误恢复
- ✅ 需要实时响应的流式对话
- ✅ 需要详细日志的生产环境

### 技术亮点

- 🎯 **智能上下文管理**: Token 预算控制和历史裁剪
- 🔄 **并发工具执行**: 提高执行效率
- 🎭 **Hook 系统**: 完整的生命周期管理
- 📊 **结构化日志**: 便于分析和调试
- 🛡️ **防御性编程**: 健壮的错误处理

这个框架展示了如何构建一个**企业级的 AI 代理执行引擎**，是学习和参考的优秀范例。