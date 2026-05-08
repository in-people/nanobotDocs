# Nanobot 调用链分析

## 从对话开始到 Anthropic 流式输出的完整调用链

### 1. 用户输入阶段
**文件**: `commands.py:1044-1050`
```python
await bus.publish_inbound(InboundMessage(
    channel=cli_channel,
    sender_id="user",
    chat_id=cli_chat_id,
    content=user_input,
    metadata={"_wants_stream": True},
))
```

### 2. 消息分发处理
**文件**: `loop.py:433-490`
- `AgentLoop._dispatch()` 接收消息
- 检查 `_wants_stream` 标志
- 创建流式回调函数 `on_stream` 和 `on_stream_end`

### 3. 流式回调设置
**文件**: `loop.py:448-469`
```python
async def on_stream(delta: str) -> None:
    meta = dict(msg.metadata or {})
    meta["_stream_delta"] = True
    await self.bus.publish_outbound(OutboundMessage(
        channel=msg.channel, chat_id=msg.chat_id,
        content=delta,
        metadata=meta,
    ))
```

### 4. 消息处理核心
**文件**: `loop.py:514-619`
- `_process_message()` 构建上下文
- 调用 `_run_agent_loop()` 执行代理循环

### 5. 代理循环执行
**文件**: `loop.py:338-399`
- 创建 `_LoopHook` 包装流式回调
- 调用 `AgentRunner.run()` 执行实际的 LLM 调用

### 6. 流式输出钩子
**文件**: `loop.py:70-83`
```python
async def on_stream(self, context: AgentHookContext, delta: str) -> None:
    from nanobot.utils.helpers import strip_think

    prev_clean = strip_think(self._stream_buf)
    self._stream_buf += delta
    new_clean = strip_think(self._stream_buf)
    incremental = new_clean[len(prev_clean):]
    if incremental and self._on_stream:
        await self._on_stream(incremental)
```

### 7. AgentRunner 调用 Provider
**文件**: `agent/runner.py` 调用 provider 的流式 API

### 8. Anthropic Provider 流式调用
**文件**: `anthropic_provider.py:483-527`
```python
async def chat_stream(
    self,
    messages: list[dict[str, Any]],
    on_content_delta: Callable[[str], Awaitable[None]] | None = None,
) -> LLMResponse:
    kwargs = self._build_kwargs(
        messages, tools, model, max_tokens, temperature,
        reasoning_effort, tool_choice,
    )
    idle_timeout_s = int(os.environ.get("NANOBOT_STREAM_IDLE_TIMEOUT_S", "90"))
    try:
        async with self._client.messages.stream(**kwargs) as stream:
            if on_content_delta:
                stream_iter = stream.text_stream.__aiter__()
                while True:
                    try:
                        text = await asyncio.wait_for(
                            stream_iter.__anext__(),
                            timeout=idle_timeout_s,
                        )
                    except StopAsyncIteration:
                        break
                    await on_content_delta(text)
            response = await asyncio.wait_for(
                stream.get_final_message(),
                timeout=idle_timeout_s,
            )
        return self._parse_response(response)
```

### 9. 流式渲染器
**文件**: `stream.py:92-108`
```python
async def on_delta(self, delta: str) -> None:
    self.streamed = True
    self._buf += delta
    if self._live is None:
        if not self._buf.strip():
            return
        self._stop_spinner()
        c = _make_console()
        c.print()
        c.print(f"[cyan]{__logo__} nanobot[/cyan]")
        self._live = Live(self._render(), console=c, auto_refresh=False)
        self._live.start()
    now = time.monotonic()
    if "\n" in delta or (now - self._t) > 0.05:
        self._live.update(self._render())
        self._live.refresh()
        self._t = now
```

### 10. 消息总线消费
**文件**: `commands.py:975-1020`
```python
async def _consume_outbound():
    while True:
        try:
            msg = await asyncio.wait_for(bus.consume_outbound(), timeout=1.0)
            
            if msg.metadata.get("_stream_delta"):
                if renderer:
                    await renderer.on_delta(msg.content)
                continue
            if msg.metadata.get("_stream_end"):
                if renderer:
                    await renderer.on_end(
                        resuming=msg.metadata.get("_resuming", False),
                    )
                continue
```

## 完整调用链总结

```
用户输入 
  → MessageBus.publish_inbound() 
  → AgentLoop._dispatch() 
  → AgentLoop._process_message() 
  → AgentLoop._run_agent_loop() 
  → AgentRunner.run() 
  → AnthropicProvider.chat_stream() 
  → Anthropic API流式响应 
  → on_content_delta回调 
  → LoopHook.on_stream() 
  → MessageBus.publish_outbound() 
  → _consume_outbound() 
  → StreamRenderer.on_delta() 
  → Rich Live实时渲染
```

## 关键组件说明

### 核心组件
1. **MessageBus**: 消息总线，负责组件间的异步通信
2. **AgentLoop**: 代理循环，核心处理引擎
3. **AgentRunner**: 代理运行器，执行实际的 LLM 调用
4. **AnthropicProvider**: Anthropic API 提供者，处理与 Claude 的交互
5. **StreamRenderer**: 流式渲染器，使用 Rich 实时渲染输出

### 流式处理特点
- **异步管道**: 所有组件都是异步的，支持高并发
- **消息总线解耦**: 各组件通过消息总线通信，易于扩展
- **增量渲染**: 只渲染变化的部分，提高性能
- **多渠道支持**: 支持 CLI、Telegram 等多种输出渠道
- **错误处理**: 完善的超时和错误处理机制

### 设计优势
1. **可扩展性**: 通过消息总线可以轻松添加新的处理器
2. **模块化**: 每个组件职责单一，便于维护
3. **性能优化**: 异步处理和增量渲染提高响应速度
4. **容错性**: 完善的错误处理和重试机制
5. **多租户支持**: 支持多个会话并发处理