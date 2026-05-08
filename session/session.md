# Session、Channel 与并发控制机制分析

> 源码: `nanobot/agent/loop.py`, `nanobot/bus/events.py`

---

## 一、三个核心概念

### 1. Channel（渠道）

消息来源的平台/通道。当前为 `cli`，表示从命令行终端使用。

nanobot 支持的渠道：

| channel | 说明 |
|---|---|
| `cli` | 命令行终端 |
| `telegram` | Telegram 机器人 |
| `discord` | Discord 机器人 |
| `slack` | Slack 机器人 |
| `wechat` | 微信 |
| `dingtalk` | 钉钉 |
| `feishu` | 飞书 |
| `wecom` | 企业微信 |
| `qq` | QQ |
| `email` | 邮件 |
| `whatsapp` | WhatsApp |

### 2. Chat ID（聊天标识）

同一个渠道内的具体聊天窗口/群组。当前为 `direct`，表示 CLI 的直接对话。

不同渠道的 chat_id 含义不同：

| 渠道 | chat_id 示例 | 含义 |
|---|---|---|
| cli | `direct` | 固定值，直接对话 |
| telegram | `8281248569` | 用户 ID 或群组 ID |
| discord | `1234567890` | 频道 ID |
| slack | `#general` | 频道名 |
| email | `user@example.com` | 邮箱地址 |

### 3. Session Key（会话键）

由 `channel` + `chat_id` 组合而成，是并发控制的唯一标识：

```python
# bus/events.py L22-24
@property
def session_key(self) -> str:
    return self.session_key_override or f"{self.channel}:{self.chat_id}"
```

当前 session_key = `cli:direct`

---

## 二、三者关系

```
Channel（渠道）── 平台级别
  └── Chat ID（聊天窗口）── 同一平台内的不同对话
        └── Session Key = "channel:chat_id" ── 并发控制的最小单位
```

具体例子：

```
nanobot 同时运行
├── telegram:123456789    ← 张三在 Telegram 私聊
├── telegram:987654321    ← 李四在 Telegram 私聊
├── telegram:-10012345    ← 某个 Telegram 群组
├── discord:channel-01    ← Discord 某个频道
├── cli:direct            ← 命令行终端
└── slack:#general        ← Slack 某个频道
```

---

## 三、并发控制机制

> 源码: `loop.py` `_dispatch()` 方法

```python
async def _dispatch(self, msg: InboundMessage) -> None:
    """Process a message: per-session serial, cross-session concurrent."""
    lock = self._session_locks.setdefault(msg.session_key, asyncio.Lock())
    gate = self._concurrency_gate or nullcontext()
    async with lock, gate:
        # ... 处理消息
```

### 两层锁机制

#### 第一层：Session Lock（会话锁）— per-session serial

```python
lock = self._session_locks.setdefault(msg.session_key, asyncio.Lock())
```

- 每个 `session_key` 对应一把 `asyncio.Lock`
- 同一会话的消息必须排队，一条处理完才能处理下一条
- **目的**：保证 LLM 对话上下文的正确性

#### 第二层：Concurrency Gate（全局信号量）— 全局限流

```python
_max = int(os.environ.get("NANOBOT_MAX_CONCURRENT_REQUESTS", "3"))
self._concurrency_gate = asyncio.Semaphore(_max) if _max > 0 else None
```

- 全局最多同时处理 N 个请求（默认 3）
- 防止同时打太多 LLM API 调用
- 可通过环境变量 `NANOBOT_MAX_CONCURRENT_REQUESTS` 配置

#### 组合使用

```python
async with lock, gate:  # 先拿会话锁（串行），再拿全局信号量（限流）
```

---

## 四、并发场景示例

### 场景：3 个会话同时来了消息

```
会话 A (telegram:111): 消息1 → 消息2 → 消息3    ← 串行排队
会话 B (telegram:222): 消息1 → 消息2            ← 串行排队
会话 C (cli:direct):   消息1                    ← 串行排队
```

时间线：

```
时刻1: A消息1、B消息1、C消息1 → 并发执行（占满3个信号量）
时刻2: A消息1 完成 → A消息2 开始（B消息1、C消息1 还在执行）
时刻3: B消息1 完成 → B消息2 开始
时刻4: C消息1 完成 → 释放一个信号量
...
```

### 关键规则

| 规则 | 说明 |
|---|---|
| 同一 session 内 | 严格串行，保证上下文正确 |
| 不同 session 间 | 互不阻塞，并行处理 |
| 全局并发上限 | 默认 3，防止 API 过载 |

---

## 五、数据流

```
InboundMessage
├── channel: str          # 来源渠道
├── sender_id: str        # 发送者 ID
├── chat_id: str          # 聊天窗口 ID
├── content: str          # 消息内容
├── session_key: str      # = f"{channel}:{chat_id}"  ← 并发控制键
└── metadata: dict        # 渠道特定数据

        ↓

_dispatch(msg)
├── lock = session_locks[session_key]   ← 会话锁
├── gate = concurrency_gate             ← 全局信号量
└── async with lock, gate:
        ↓
    _process_message(msg)
        ↓
    OutboundMessage
    ├── channel: str
    ├── chat_id: str
    ├── content: str
    └── media: list
```
