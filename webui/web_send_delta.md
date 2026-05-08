# WebSocket流式输出机制分析

## 概述

本文档详细分析了nanobot系统中WebSocket流式输出的实现机制，重点关注`send_delta`方法如何被调用。

## 消息流程

```
WebSocket客户端 → WebSocketChannel → MessageBus → Agent处理 → MessageBus → WebSocketChannel → WebSocket客户端
```

## 关键调用点

### 1. WebSocket接收消息
```python
# websocket.py:734-756
async for raw in connection:
    envelope = _parse_envelope(raw)
    if envelope is not None:
        await self._dispatch_envelope(connection, client_id, envelope)
        continue
    
    content = _parse_inbound_payload(raw)
    if content is None:
        continue
    await self._handle_message(
        sender_id=client_id,
        chat_id=default_chat_id,
        content=content,
        metadata={"remote": getattr(connection, "remote_address", None)},
    )
```

### 2. 消息发送到总线
```python
# websocket.py:140-184 (BaseChannel._handle_message)
async def _handle_message(self, sender_id, chat_id, content, ...):
    if not self.is_allowed(sender_id):
        return
    
    msg = InboundMessage(
        channel=self.name,
        sender_id=str(sender_id),
        chat_id=str(chat_id),
        content=content,
        metadata=meta,
    )
    
    await self.bus.publish_inbound(msg)  # 发送到消息总线
```

### 3. ChannelManager监听总线
```python
# manager.py:196-240 (ChannelManager._dispatch_outbound)
async def _dispatch_outbound(self) -> None:
    while True:
        try:
            msg = await asyncio.wait_for(self.bus.consume_outbound(), timeout=1.0)
            
            # 关键：根据channel字段选择调用哪个send_delta
            if msg.metadata.get("_stream_delta") or msg.metadata.get("_stream_end"):
                await channel.send_delta(msg.chat_id, msg.content, msg.metadata)
            elif not msg.metadata.get("_streamed"):
                await channel.send(msg)
                
        except asyncio.TimeoutError:
            continue
```

## Channel选择机制

### 关键发现
```python
await self.bus.publish_outbound(OutboundMessage(
    channel=msg.channel,  # ← 关键！这里的channel决定了调用哪个send_delta
    chat_id=msg.chat_id,
    content="", metadata=msg.metadata or {},
))
```

### Channel路由原理
```python
# manager.py:54
self.channels: dict[str, BaseChannel] = {}  # key: channel名称

# manager.py:230-232
channel = self.channels.get(msg.channel)  # 根据channel名称查找
if channel:
    await self._send_with_retry(channel, msg)  # 调用对应channel的方法
```

## 多态行为分析

### 1. 继承体系
```python
# 基类：BaseChannel
class BaseChannel(ABC):
    @abstractmethod
    async def send_delta(self, chat_id: str, delta: str, metadata: dict[str, Any] | None = None) -> None:
        pass
    
    @property
    def supports_streaming(self) -> bool:
        return bool(streaming) and type(self).send_delta is not BaseChannel.send_delta

# 具体子类：WebSocketChannel
class WebSocketChannel(BaseChannel):
    async def send_delta(self, chat_id: str, delta: str, metadata: dict[str, Any] | None = None) -> None:
        # WebSocket实现：通过WebSocket发送delta事件
        conns = list(self._subs.get(chat_id, ()))
        for connection in conns:
            body = {"event": "delta", "chat_id": chat_id, "text": delta}
            await connection.send(json.dumps(body))
```

### 2. 多态的体现
- **接口统一**：所有Channel都实现`send_delta`方法
- **运行时绑定**：根据`msg.channel`选择具体实现
- **解耦的设计**：调用方不需要知道具体实现细节

### 3. 多态的优势
- **a) 接口统一**：ChannelManager只认识`BaseChannel`接口
- **b) 开闭原则**：新增Channel只需继承BaseChannel
- **c) 运行时绑定**：动态决定调用哪个实现

## WebSocket的send_delta实现

```python
# websocket.py:857-879
async def send_delta(self, chat_id: str, delta: str, metadata: dict[str, Any] | None = None) -> None:
    conns = list(self._subs.get(chat_id, ()))
    if not conns:
        return
    
    meta = metadata or {}
    if meta.get("_stream_end"):
        body = {"event": "stream_end", "chat_id": chat_id}
    else:
        body = {
            "event": "delta",
            "chat_id": chat_id,
            "text": delta,
        }
    
    if meta.get("_stream_id") is not None:
        body["stream_id"] = meta["_stream_id"]
    
    raw = json.dumps(body, ensure_ascii=False)
    for connection in conns:
        await self._safe_send_to(connection, raw, label=" stream ")
```

## 总结

1. **Channel名称**：在OutboundMessage中通过`channel`字段指定
2. **Channel查找**：ChannelManager根据channel名称在字典中查找对应的Channel实例
3. **方法调用**：找到实例后，调用其实现的send_delta方法
4. **多态行为**：这是面向对象编程中典型的多态实现，通过继承和虚函数实现运行时绑定

关键结论：`channel`字段是决定调用哪个Channel的send_delta方法的关键！这体现了面向对象编程中的多态特性。
