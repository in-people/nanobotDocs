# OpenAICompatProvider 提供商详解

## 核心定位
重点关注 缓存控制 (157-189行)、响应解析、流式响应解析、错误处理
成本节省、性能提升（缓存的内容无需重新处理、响应更快）

```python 
    # 修复可能损坏的JSON
    if isinstance(args, str):
        args = json_repair.loads(args)

    arguments=json_repair.loads(b["arguments"]) if b["arguments"] else {},
```

`OpenAICompatProvider` 是一个**统一的 OpenAI 兼容 API 提供商**，支持所有遵循 OpenAI API 格式的 LLM 服务，包括：

- 🥇 **OpenAI** (GPT-4, GPT-3.5)
- 🤖 **DeepSeek** (deepseek-chat, deepseek-coder)
- 🧠 **Zhipu AI** (GLM-4, GLM-5)
- 💎 **Gemini** (Gemini Pro, Gemini Flash)
- 🌙 **Moonshot** (Kimi 系列)
- 🔥 **DashScope** (通义千问)
- 📊 **MiniMax**
- 🌐 **Mistral**
- 🎯 **StepFun** (阶跃星辰)
- 🔮 **OpenRouter** (聚合网关)
- 等数十个提供商...

---

## 文件结构概览

```
openai_compat_provider.py
├── 工具函数 (37-103行)
│   ├── _short_tool_id() - 生成工具调用ID
│   ├── _get() - 安全获取值
│   ├── _coerce_dict() - 转换为字典
│   ├── _extract_tc_extras() - 提取工具调用额外字段
│   └── _uses_openrouter_attribution() - OpenRouter归属检测
│
├── OpenAICompatProvider 类 (105-768行)
│   ├── 初始化 (112-142行)
│   ├── 环境设置 (143-156行)
│   ├── 缓存控制 (157-189行)
│   ├── 消息清理 (191-223行)
│   ├── 参数构建 (225-312行)
│   ├── 响应解析 (314-526行)
│   ├── 流式响应解析 (528-625行)
│   ├── 错误处理 (627-693行)
│   └── 公共API (699-767行)
```

---

## 1. 工具函数详解

### **`_short_tool_id()`** - 生成工具调用ID

```python
def _short_tool_id() -> str:
    """9-char alphanumeric ID compatible with all providers (incl. Mistral)."""
    return "".join(secrets.choice(_ALNUM) for _ in range(9))
```

**作用：** 生成9位字母数字ID，兼容所有提供商（包括 Mistral）

**为什么需要：**
- 某些提供商（如 Mistral）对工具调用ID格式有严格要求
- UUID 太长，可能导致问题
- 使用加密安全的随机生成器确保唯一性

---

### **`_get()`** - 安全获取值

```python
def _get(obj: Any, key: str) -> Any:
    """Get a value from dict or object attribute, returning None if absent."""
    if isinstance(obj, dict):
        return obj.get(key)
    return getattr(obj, key, None)
```

**作用：** 统一处理字典和对象的属性访问

**使用场景：**
```python
# 从字典获取
value = _get({"name": "test"}, "name")  # "test"

# 从对象获取
value = _get(SomeObject(), "name")     # 对象的 name 属性

# 不存在的键返回 None
value = _get({"name": "test"}, "age")  # None
```

---

### **`_coerce_dict()`** - 转换为字典

```python
def _coerce_dict(value: Any) -> dict[str, Any] | None:
    """Try to coerce *value* to a dict; return None if not possible or empty."""
    if value is None:
        return None
    if isinstance(value, dict):
        return value if value else None
    model_dump = getattr(value, "model_dump", None)
    if callable(model_dump):
        dumped = model_dump()
        if isinstance(dumped, dict) and dumped:
            return dumped
    return None
```

**支持的转换：**
1. 字典 → 直接返回
2. Pydantic 模型 → 调用 `model_dump()`
3. 其他 → 返回 `None`

---

### **`_extract_tc_extras()`** - 提取工具调用额外字段

```python
def _extract_tc_extras(tc: Any) -> tuple[
    dict[str, Any] | None,
    dict[str, Any] | None,
    dict[str, Any] | None,
]:
    """Extract (extra_content, provider_specific_fields, fn_provider_specific_fields)."""
```

**返回：** `(extra_content, provider_specific_fields, fn_provider_specific_fields)`

**作用：** 提取工具调用的非标准字段，保持提供商特定信息

**提取的字段：**
- `extra_content`: Gemini 等提供商的额外内容
- `provider_specific_fields`: 工具调用级别的提供商特定字段
- `fn_provider_specific_fields`: 函数级别的提供商特定字段

---

## 2. OpenAICompatProvider 类详解

### **A. 初始化方法**

```python
def __init__(
    self,
    api_key: str | None = None,
    api_base: str | None = None,
    default_model: str = "gpt-4o",
    extra_headers: dict[str, str] | None = None,
    spec: "ProviderSpec | None" = None,
):
```

**参数说明：**

| 参数 | 类型 | 说明 |
|------|------|------|
| `api_key` | `str \| None` | API 密钥 |
| `api_base` | `str \| None` | API 基础URL |
| `default_model` | `str` | 默认模型名称 |
| `extra_headers` | `dict \| None` | 额外的HTTP头部 |
| `spec` | `ProviderSpec \| None` | 提供商规格（从注册表） |

**核心功能：**

1. **创建 OpenAI 客户端**
   ```python
   self._client = AsyncOpenAI(
       api_key=api_key or "no-key",
       base_url=effective_base,
       default_headers=default_headers,
       max_retries=0,  # 禁用自动重试（使用我们自己的重试机制）
       timeout=120.0,  # 120秒超时
   )
   ```

2. **设置默认头部**
   ```python
   default_headers = {"x-session-affinity": uuid.uuid4().hex}
   
   # OpenRouter 归属信息
   if _uses_openrouter_attribution(spec, effective_base):
       default_headers.update(_DEFAULT_OPENROUTER_HEADERS)
   
   # 用户自定义头部
   if extra_headers:
       default_headers.update(extra_headers)
   ```

3. **环境变量设置**
   ```python
   def _setup_env(self, api_key: str, api_base: str | None) -> None:
       """Set environment variables based on provider spec."""
       spec = self._spec
       if not spec or not spec.env_key:
           return
       
       # 网关：设置环境变量
       if spec.is_gateway:
           os.environ[spec.env_key] = api_key
       # 普通提供商：设置默认值
       else:
           os.environ.setdefault(spec.env_key, api_key)
       
       # 设置额外的环境变量
       for env_name, env_val in spec.env_extras:
           resolved = env_val.replace("{api_key}", api_key).replace("{api_base}", effective_base)
           os.environ.setdefault(env_name, resolved)
   ```

---

### **B. 缓存控制**
成本节省、性能提升（缓存的内容无需重新处理、响应更快）
```python
@classmethod
def _apply_cache_control(
    cls,
    messages: list[dict[str, Any]],
    tools: list[dict[str, Any]] | None,
) -> tuple[list[dict[str, Any]], list[dict[str, Any]] | None]:
    """Inject cache_control markers for prompt caching."""
```

**功能：**
- 为支持缓存的提供商（如 Anthropic）注入缓存标记
- 在系统消息和倒数第二条消息中添加 `cache_control`
- 在工具定义的边界位置添加缓存标记

**缓存策略：**

```python
cache_marker = {"type": "ephemeral"}

# 1. 标记系统消息
if new_messages and new_messages[0].get("role") == "system":
    new_messages[0] = _mark(new_messages[0])

# 2. 标记倒数第二条消息
if len(new_messages) >= 3:
    new_messages[-2] = _mark(new_messages[-2])

# 3. 标记工具边界
if tools:
    for idx in cls._tool_cache_marker_indices(new_tools):
        new_tools[idx] = {**new_tools[idx], "cache_control": cache_marker}
```

**示例：**

```python
# 原始消息
messages = [
    {"role": "system", "content": "You are a helpful assistant"},
    {"role": "user", "content": "Hello"},
    {"role": "assistant", "content": "Hi"},
    {"role": "user", "content": "How are you?"}
]

# 应用缓存控制后
messages = [
    {"role": "system", "content": [
        {"type": "text", "text": "You are a helpful assistant", "cache_control": {"type": "ephemeral"}}
    ]},
    {"role": "user", "content": "Hello"},
    {"role": "assistant", "content": [
        {"type": "text", "text": "Hi", "cache_control": {"type": "ephemeral"}}
    ]},
    {"role": "user", "content": "How are you?"}
]
```

---

### **C. 消息清理**

```python
def _sanitize_messages(self, messages: list[dict[str, Any]]) -> list[dict[str, Any]]:
    """Strip non-standard keys, normalize tool_call IDs."""
```

这是多提供商适配器的关键安全机制，确保发送给不同 LLM 提供商的消息格式统一、安全、兼容，避免因字段差异导致的 API 调用失败！  
**功能：**
1. 移除非标准的消息键
2. 规范化工具调用ID（确保9位字母数字）

**允许的消息键：**

```python
_ALLOWED_MSG_KEYS = frozenset({
    "role", "content", "tool_calls", "tool_call_id", "name",
    "reasoning_content", "extra_content",
})
```

**工具调用ID规范化：**

```python
@staticmethod
def _normalize_tool_call_id(tool_call_id: Any) -> Any:
    """Normalize to a provider-safe 9-char alphanumeric form."""
    if not isinstance(tool_call_id, str):
        return tool_call_id
    if len(tool_call_id) == 9 and tool_call_id.isalnum():
        return tool_call_id
    # 使用 SHA1 哈希缩短长ID
    return hashlib.sha1(tool_call_id.encode()).hexdigest()[:9]
```

**清理流程：**

```python
# 1. 移除非标准键
sanitized = LLMProvider._sanitize_request_messages(messages, _ALLOWED_MSG_KEYS)

# 2. 规范化工具调用ID
for clean in sanitized:
    if isinstance(clean.get("tool_calls"), list):
        for tc in clean["tool_calls"]:
            tc["id"] = self._normalize_tool_call_id(tc.get("id"))
```

---

### **D. 参数构建**

```python
def _build_kwargs(
    self,
    messages: list[dict[str, Any]],
    tools: list[dict[str, Any]] | None,
    model: str | None,
    max_tokens: int,
    temperature: float,
    reasoning_effort: str | None,
    tool_choice: str | dict[str, Any] | None,
) -> dict[str, Any]:
```

**1. 温度参数支持检测**

```python
@staticmethod
def _supports_temperature(
    model_name: str,
    reasoning_effort: str | None = None,
) -> bool:
    """Return True when the model accepts a temperature parameter.
    
    GPT-5 family and reasoning models (o1/o3/o4) reject temperature
    when reasoning_effort is set to anything other than ``"none"``.
    """
    if reasoning_effort and reasoning_effort.lower() != "none":
        return False
    name = model_name.lower()
    return not any(token in name for token in ("gpt-5", "o1", "o3", "o4"))
```

**不支持的模型：**
- GPT-5 系列（启用推理时）
- o1, o3, o4（推理模型）

**2. 模型名称处理**

```python
# 移除网关前缀
if spec and spec.strip_model_prefix:
    model_name = model_name.split("/")[-1]

# 示例：
# "openai/gpt-4o" → "gpt-4o"
# "anthropic/claude-3-5-sonnet" → "claude-3-5-sonnet"
```

**3. Token 参数**

```python
if spec and getattr(spec, "supports_max_completion_tokens", False):
    kwargs["max_completion_tokens"] = max_tokens  # OpenAI 格式
else:
    kwargs["max_tokens"] = max_tokens  # 其他提供商
```

**4. 推理参数**
  推理努力程度（reasoning_effort）是什么？  
  推理努力程度是控制 AI 模型在生成响应前进行思考的时间和深度的参数，特别适用于推理模型（Reasoning Models）。  
  - 简单问题 → reasoning_effort="low" 或不设置  
  - 复杂问题 → reasoning_effort="high"  

```python
# 推理努力程度
if reasoning_effort:
    kwargs["reasoning_effort"] = reasoning_effort

# 提供商特定的思考参数
if spec and reasoning_effort is not None:
    thinking_enabled = reasoning_effort.lower() != "minimal"
    extra: dict[str, Any] | None = None
    
    if spec.name == "dashscope":
        extra = {"enable_thinking": thinking_enabled}
    elif spec.name in ("volcengine", "volcengine_coding_plan",
                      "byteplus", "byteplus_coding_plan"):
        extra = {
            "thinking": {"type": "enabled" if thinking_enabled else "disabled"}
        }
    
    if extra:
        kwargs.setdefault("extra_body", {}).update(extra)
```

**5. 模型覆盖**

```python
# 应用提供商特定的模型覆盖
if spec:
    model_lower = model_name.lower()
    for pattern, overrides in spec.model_overrides:
        if pattern in model_lower:
            kwargs.update(overrides)
            break

# 示例：Moonshot Kimi K2.5 强制 temperature >= 1.0
# ("kimi-k2.5", {"temperature": 1.0})
```

---

### **E. 响应解析**

```python
def _parse(self, response: Any) -> LLMResponse:
```

**支持的响应格式：**
1. **字符串** → 直接作为内容
2. **字典（原始JSON）** → 提取字段
3. **Pydantic 模型（SDK对象）** → 使用属性访问

**1. 内容提取**

```python
@classmethod
def _extract_text_content(cls, value: Any) -> str | None:
    """支持多种内容格式：
    - 字符串
    - 列表（包含文本块）
    - 混合格式
    """
    if value is None:
        return None
    if isinstance(value, str):
        return value
    if isinstance(value, list):
        parts: list[str] = []
        for item in value:
            # 处理字典项
            item_map = cls._maybe_mapping(item)
            if item_map:
                text = item_map.get("text")
                if isinstance(text, str):
                    parts.append(text)
                    continue
            # 处理对象属性
            text = getattr(item, "text", None)
            if isinstance(text, str):
                parts.append(text)
                continue
            # 处理字符串项
            if isinstance(item, str):
                parts.append(item)
        return "".join(parts) or None
    return str(value)
```

**2. 使用量提取**

```python
@classmethod
def _extract_usage(cls, response: Any) -> dict[str, int]:
    """提取token使用量，支持多种提供商格式：
    
    优先级顺序：
    1. OpenAI/Zhipu/MiniMax/Qwen/Mistral/xAI: prompt_tokens_details.cached_tokens
    2. StepFun/Moonshot: cached_tokens (顶层)
    3. DeepSeek/SiliconFlow: prompt_cache_hit_tokens
    """
    # 解析 usage 对象
    usage_obj = None
    response_map = cls._maybe_mapping(response)
    if response_map is not None:
        usage_obj = response_map.get("usage")
    elif hasattr(response, "usage") and response.usage:
        usage_obj = response.usage
    
    # 提取基本使用量
    usage_map = cls._maybe_mapping(usage_obj)
    if usage_map is not None:
        result = {
            "prompt_tokens": int(usage_map.get("prompt_tokens") or 0),
            "completion_tokens": int(usage_map.get("completion_tokens") or 0),
            "total_tokens": int(usage_map.get("total_tokens") or 0),
        }
    
    # 提取缓存token
    for path in (
        ("prompt_tokens_details", "cached_tokens"),  # OpenAI/Zhipu/MiniMax/Qwen/Mistral/xAI
        ("cached_tokens",),                          # StepFun/Moonshot (top-level)
        ("prompt_cache_hit_tokens",),                # DeepSeek/SiliconFlow
    ):
        cached = cls._get_nested_int(usage_map, path)
        if cached:
            result["cached_tokens"] = cached
            break
    
    return result
```

**3. 工具调用解析**

```python
# 从响应中提取工具调用
raw_tool_calls: list[Any] = []
for ch in choices:
    ch_map = cls._maybe_mapping(ch) or {}
    m = cls._maybe_mapping(ch_map.get("message")) or {}
    tool_calls = m.get("tool_calls")
    if isinstance(tool_calls, list) and tool_calls:
        raw_tool_calls.extend(tool_calls)

# 解析每个工具调用
parsed_tool_calls = []
for tc in raw_tool_calls:
    tc_map = cls._maybe_mapping(tc) or {}
    fn = cls._maybe_mapping(tc_map.get("function")) or {}
    args = fn.get("arguments", {})
    
    # 修复可能损坏的JSON
    if isinstance(args, str):
        args = json_repair.loads(args)
    
    # 提取额外字段
    ec, prov, fn_prov = _extract_tc_extras(tc)
    
    parsed_tool_calls.append(ToolCallRequest(
        id=_short_tool_id(),  # 生成新的9位ID
        name=str(fn.get("name") or ""),
        arguments=args if isinstance(args, dict) else {},
        extra_content=ec,
        provider_specific_fields=prov,
        function_provider_specific_fields=fn_prov,
    ))
```

---

### **F. 流式响应解析**

```python
@classmethod
def _parse_chunks(cls, chunks: list[Any]) -> LLMResponse:
```

**核心逻辑：**

**1. 累积文本内容**

```python
content_parts: list[str] = []

for chunk in chunks:
    chunk_map = cls._maybe_mapping(chunk)
    if chunk_map is not None:
        choice = cls._maybe_mapping(choices[0]) or {}
        delta = cls._maybe_mapping(choice.get("delta")) or {}
        
        text = cls._extract_text_content(delta.get("content"))
        if text:
            content_parts.append(text)

# 合并所有文本片段
content = "".join(content_parts) or None
```

**2. 累积工具调用**

```python
tc_bufs: dict[int, dict[str, Any]] = {}

def _accum_tc(tc: Any, idx_hint: int) -> None:
    """按索引累积工具调用的增量更新"""
    tc_index: int = _get(tc, "index") if _get(tc, "index") is not None else idx_hint
    buf = tc_bufs.setdefault(tc_index, {
        "id": "", "name": "", "arguments": "",
        "extra_content": None, "prov": None, "fn_prov": None,
    })
    
    # 累积 ID
    tc_id = _get(tc, "id")
    if tc_id:
        buf["id"] = str(tc_id)
    
    # 累积名称
    fn = _get(tc, "function")
    if fn is not None:
        fn_name = _get(fn, "name")
        if fn_name:
            buf["name"] = str(fn_name)
    
    # 累积参数（增量追加）
    fn_args = _get(fn, "arguments")
    if fn_args:
        buf["arguments"] += str(fn_args)
    
    # 提取额外字段
    ec, prov, fn_prov = _extract_tc_extras(tc)
    if ec:
        buf["extra_content"] = ec
    if prov:
        buf["prov"] = prov
    if fn_prov:
        buf["fn_prov"] = fn_prov

# 处理所有chunk的工具调用
for chunk in chunks:
    for idx, tc in enumerate(delta.get("tool_calls") or []):
        _accum_tc(tc, idx)
```

**3. 处理推理内容**

```python
reasoning_parts: list[str] = []

for chunk in chunks:
    text = cls._extract_text_content(delta.get("reasoning_content"))
    if text:
        reasoning_parts.append(text)

# 合并推理内容
reasoning_content = "".join(reasoning_parts) or None
```

---

### **G. 错误处理**

```python
@classmethod
def _extract_error_metadata(cls, e: Exception) -> dict[str, Any]:
    """从异常中提取结构化错误元数据"""
    response = getattr(e, "response", None)
    headers = getattr(response, "headers", None)
    payload = (
        getattr(e, "body", None)
        or getattr(e, "doc", None)
        or getattr(response, "text", None)
    )
    
    # 提取错误类型和代码
    error_type, error_code = LLMProvider._extract_error_type_code(payload)
    
    # 提取状态码
    status_code = getattr(e, "status_code", None)
    if status_code is None and response is not None:
        status_code = getattr(response, "status_code", None)
    
    # 提取重试建议
    should_retry: bool | None = None
    if headers is not None:
        raw = headers.get("x-should-retry")
        if isinstance(raw, str):
            lowered = raw.strip().lower()
            if lowered == "true":
                should_retry = True
            elif lowered == "false":
                should_retry = False
    
    # 错误分类
    error_kind: str | None = None
    error_name = e.__class__.__name__.lower()
    if "timeout" in error_name:
        error_kind = "timeout"
    elif "connection" in error_name:
        error_kind = "connection"
    
    return {
        "error_status_code": int(status_code) if status_code is not None else None,
        "error_kind": error_kind,
        "error_type": error_type,
        "error_code": error_code,
        "error_retry_after_s": cls._extract_retry_after_from_headers(headers),
        "error_should_retry": should_retry,
    }
```

**处理异常：**

```python
@staticmethod
def _handle_error(e: Exception) -> LLMResponse:
    """处理异常并返回错误响应"""
    body = (
        getattr(e, "doc", None)
        or getattr(e, "body", None)
        or getattr(getattr(e, "response", None), "text", None)
    )
    body_text = body if isinstance(body, str) else str(body) if body is not None else ""
    msg = f"Error: {body_text.strip()[:500]}" if body_text.strip() else f"Error calling LLM: {e}"
    
    response = getattr(e, "response", None)
    retry_after = LLMProvider._extract_retry_after_from_headers(getattr(response, "headers", None))
    if retry_after is None:
        retry_after = LLMProvider._extract_retry_after(msg)
    
    return LLMResponse(
        content=msg,
        finish_reason="error",
        retry_after=retry_after,
        **OpenAICompatProvider._extract_error_metadata(e),
    )
```

---

### **H. 公共API**

#### **`chat()`** - 非流式聊天

```python
async def chat(
    self,
    messages: list[dict[str, Any]],
    tools: list[dict[str, Any]] | None = None,
    model: str | None = None,
    max_tokens: int = 4096,
    temperature: float = 0.7,
    reasoning_effort: str | None = None,
    tool_choice: str | dict[str, Any] | None = None,
) -> LLMResponse:
    # 构建参数
    kwargs = self._build_kwargs(
        messages, tools, model, max_tokens, temperature,
        reasoning_effort, tool_choice,
    )
    
    try:
        # 调用 API
        response = await self._client.chat.completions.create(**kwargs)
        # 解析响应
        return self._parse(response)
    except Exception as e:
        # 处理错误
        return self._handle_error(e)
```

#### **`chat_stream()`** - 流式聊天

```python
async def chat_stream(
    self,
    messages: list[dict[str, Any]],
    tools: list[dict[str, Any]] | None = None,
    model: str | None = None,
    max_tokens: int = 4096,
    temperature: float = 0.7,
    reasoning_effort: str | None = None,
    tool_choice: str | dict[str, Any] | None = None,
    on_content_delta: Callable[[str], Awaitable[None]] | None = None,
) -> LLMResponse:
    # 构建参数
    kwargs = self._build_kwargs(
        messages, tools, model, max_tokens, temperature,
        reasoning_effort, tool_choice,
    )
    
    # 启用流式
    kwargs["stream"] = True
    kwargs["stream_options"] = {"include_usage": True}
    
    # 空闲超时（默认90秒）
    idle_timeout_s = int(os.environ.get("STREAM_IDLE_TIMEOUT_S", "90"))
    
    try:
        # 创建流
        stream = await self._client.chat.completions.create(**kwargs)
        chunks: list[Any] = []
        stream_iter = stream.__aiter__()
        
        # 收集所有chunk
        while True:
            try:
                chunk = await asyncio.wait_for(
                    stream_iter.__anext__(),
                    timeout=idle_timeout_s,  # 90秒空闲超时
                )
            except StopAsyncIteration:
                break
            
            chunks.append(chunk)
            
            # 实时回调
            if on_content_delta and chunk.choices:
                text = getattr(chunk.choices[0].delta, "content", None)
                if text:
                    await on_content_delta(text)
        
        # 解析所有chunk
        return self._parse_chunks(chunks)
    
    except asyncio.TimeoutError:
        return LLMResponse(
            content=(
                f"Error calling LLM: stream stalled for more than "
                f"{idle_timeout_s} seconds"
            ),
            finish_reason="error",
            error_kind="timeout",
        )
    
    except Exception as e:
        return self._handle_error(e)
```

---

## 核心特性

### **1. 统一接口**
- ✅ 支持数十个 OpenAI 兼容的提供商
- ✅ 自动处理不同提供商的格式差异
- ✅ 统一的错误处理和重试机制

### **2. 智能适配**
- ✅ 自动检测模型能力（温度、token参数）
- ✅ 提供商特定参数覆盖
- ✅ 模型名称前缀处理

### **3. 缓存优化**
- ✅ 自动注入缓存标记
- ✅ 支持提供商特定的缓存策略
- ✅ 减少API调用成本

### **4. 流式支持**
- ✅ 实时内容增量回调
- ✅ 空闲超时保护（90秒）
- ✅ 工具调用的流式组装

### **5. 错误处理**
- ✅ 结构化错误元数据
- ✅ 多格式响应解析
- ✅ 自动JSON修复

### **6. 兼容性**
- ✅ 支持原始JSON和SDK对象
- ✅ 统一的工具调用ID生成
- ✅ 提供商特定字段保留

---

## 使用示例

### **基础使用**

```python
from providers import OpenAICompatProvider
from providers.registry import find_by_name

# 初始化
spec = find_by_name("zhipu")
provider = OpenAICompatProvider(
    api_key="your-api-key",
    api_base=spec.default_api_base,
    default_model="glm-5",
    spec=spec
)

# 非流式调用
response = await provider.chat(
    messages=[{"role": "user", "content": "Hello"}],
    max_tokens=1000,
    temperature=0.7
)

print(response.content)  # "Hello! How can I help you today?"
```

### **流式调用**

```python
# 流式调用
async def on_text_delta(text: str):
    print(text, end="", flush=True)

response = await provider.chat_stream(
    messages=[{"role": "user", "content": "Tell me a story"}],
    on_content_delta=on_text_delta
)

# 实时输出：
# Once upon a time...
```

### **工具调用**

```python
# 带工具调用
response = await provider.chat(
    messages=[{"role": "user", "content": "What's the weather in Beijing?"}],
    tools=[{
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "Get current weather for a location",
            "parameters": {
                "type": "object",
                "properties": {
                    "location": {
                        "type": "string",
                        "description": "The city and state, e.g. San Francisco, CA"
                    }
                },
                "required": ["location"]
            }
        }
    }]
)

if response.has_tool_calls:
    for tool_call in response.tool_calls:
        print(f"Tool: {tool_call.name}")
        print(f"Args: {tool_call.arguments}")
        # 输出:
        # Tool: get_weather
        # Args: {"location": "Beijing"}
```

### **带重试的调用**

```python
from providers.base import LLMProvider

# 继承重试功能
response = await provider.chat_with_retry(
    messages=[{"role": "user", "content": "Complex task"}],
    retry_mode="persistent",  # 持久重试
    on_retry_wait=lambda msg: print(f"⏳ {msg}")
)
```

### **多提供商切换**

```python
# OpenAI
openai_spec = find_by_name("openai")
openai_provider = OpenAICompatProvider(
    api_key="sk-xxx",
    spec=openai_spec
)

# DeepSeek
deepseek_spec = find_by_name("deepseek")
deepseek_provider = OpenAICompatProvider(
    api_key="sk-yyy",
    spec=deepseek_spec
)

# 相同的调用方式
for provider in [openai_provider, deepseek_provider]:
    response = await provider.chat(
        messages=[{"role": "user", "content": "Hello"}]
    )
    print(response.content)
```

---

## 设计优势

| 特性 | 好处 |
|------|------|
| **统一接口** | 一个类支持数十个提供商 |
| **智能适配** | 自动处理提供商差异 |
| **缓存优化** | 自动应用缓存策略，降低成本 |
| **流式支持** | 实时响应用户输入 |
| **错误处理** | 结构化错误元数据，便于调试 |
| **兼容性** | 支持多种响应格式 |
| **可扩展** | 易于添加新的提供商 |

---

## 总结

`OpenAICompatProvider` 是整个多提供商适配器的核心实现，通过以下设计实现了真正的提供商无关性：

1. **统一接口** - 所有提供商使用相同的调用方式
2. **智能适配** - 自动检测和处理提供商差异
3. **健壮的错误处理** - 结构化错误元数据和重试机制
4. **性能优化** - 缓存控制和流式响应
5. **易于扩展** - 新提供商只需添加注册表条目

开发者可以无缝切换不同的 LLM 提供商，而无需修改业务代码，真正实现了"一次编写，到处运行"的跨提供商 LLM 应用开发！
