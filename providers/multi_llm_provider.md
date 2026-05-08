# 多 LLM 供应商适配机制分析

> 源码:
> - `nanobot/providers/registry.py` — 供应商注册表（元数据驱动）
> - `nanobot/providers/openai_compat_provider.py` — 统一适配层
> - `nanobot/providers/transcription.py` — 语音转文字（独立模块）

---

## 一、整体架构

```
┌─────────────────────────────────────────────────────────┐
│                    registry.py (注册表)                   │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │
│  │ ProviderSpec │ ProviderSpec │ ProviderSpec │ ...      │   │
│  │  openai  │ │ deepseek │ │ dashscope│ │ anthropic│   │
│  │ backend=  │ │ backend=  │ │ backend=  │ │ backend=  │   │
│  │openai_compat│openai_compat│openai_compat│ anthropic │   │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘   │
│         │            │            │                       │
│         └────────────┼────────────┘                       │
│                      ▼                                    │
│        PROVIDERS = (所有供应商的元数据)                     │
└─────────────────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│          openai_compat_provider.py (统一适配层)            │
│                                                         │
│  OpenAICompatProvider(LLMProvider)                       │
│  ┌─────────────────────────────────────────────┐        │
│  │  一个类，适配所有 OpenAI 兼容的 API            │        │
│  │                                             │        │
│  │  __init__(api_key, api_base, spec)           │        │
│  │    → 创建 AsyncOpenAI(base_url=spec.default) │        │
│  │                                             │        │
│  │  chat()    → 调 LLM                         │        │
│  │  chat_stream() → 流式调 LLM                  │        │
│  │  _parse()  → 统一解析不同供应商的响应          │        │
│  └─────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│          transcription.py (语音转文字)                    │
│                                                         │
│  OpenAITranscriptionProvider  — OpenAI Whisper           │
│  GroqTranscriptionProvider    — Groq Whisper             │
│  (独立于 LLM，辅助功能)                                    │
└─────────────────────────────────────────────────────────┘
```

---

## 二、registry.py — 供应商注册表（元数据驱动）

**核心思想**：不写 if/else，用**数据驱动**来描述每个供应商的差异。

### ProviderSpec — 供应商的"身份证"

```python
@dataclass(frozen=True)
class ProviderSpec:
    name: str                          # 配置字段名，如 "deepseek"
    keywords: tuple[str, ...]          # 模型名匹配关键词，如 ("deepseek",)
    env_key: str                       # API Key 环境变量名，如 "DEEPSEEK_API_KEY"
    backend: str = "openai_compat"     # 后端实现类型
    default_api_base: str = ""         # API 地址
    is_gateway: bool = False           # 是否为网关型（能路由任何模型）
    strip_model_prefix: bool = False   # 是否去除模型名前缀
    model_overrides: tuple = ()        # 特定模型的参数覆盖
    ...
```

### 关键字段说明

| 字段 | 作用 | 示例 |
|---|---|---|
| `name` | 配置中的唯一标识 | `"deepseek"` |
| `keywords` | 模型名匹配关键词 | `("deepseek",)` → 匹配 `deepseek-chat` |
| `env_key` | API Key 环境变量名 | `"DEEPSEEK_API_KEY"` |
| `backend` | 用哪个实现类 | `"openai_compat"` / `"anthropic"` |
| `default_api_base` | 默认 API 地址 | `"https://api.deepseek.com"` |
| `is_gateway` | 网关型，能路由任何模型 | OpenRouter、AiHubMix |
| `strip_model_prefix` | 去除模型名前缀 | `"anthropic/claude-3"` → `"claude-3"` |
| `model_overrides` | 特定模型参数覆盖 | Kimi K2.5 强制 `temperature=1.0` |
| `detect_by_key_prefix` | 通过 API Key 前缀识别 | OpenRouter: `"sk-or-"` |
| `detect_by_base_keyword` | 通过 URL 关键词识别 | `"openrouter"` |
| `supports_prompt_caching` | 支持 prompt 缓存 | OpenRouter、Anthropic |
| `is_oauth` | OAuth 认证（非 API Key） | OpenAI Codex、GitHub Copilot |

### 目前注册的供应商（28个）

```
分类              供应商
─────────────────────────────────────────────────────
网关(Gateway)     OpenRouter, AiHubMix, SiliconFlow, VolcEngine, BytePlus
                  (能路由任何模型，按 api_key/api_base 识别)

标准供应商        OpenAI, Anthropic, DeepSeek, Gemini, Zhipu(智谱),
                  DashScope(通义), Moonshot(月之暗面), MiniMax, Mistral,
                  StepFun(阶跃星辰), Xiaomi MIMO, Qianfan(百度千帆)

本地部署          vLLM, Ollama, OpenVINO

特殊              Azure OpenAI, OpenAI Codex(OAuth), GitHub Copilot(OAuth)

语音转录          Groq (主要用于 Whisper)
```

### 匹配优先级 = 元组顺序

```python
PROVIDERS: tuple[ProviderSpec, ...] = (
    ProviderSpec(name="custom", ...),         # 1. 最高优先
    ProviderSpec(name="azure_openai", ...),   # 2.
    ProviderSpec(name="openrouter", ...),     # 3. 网关优先
    ...
    ProviderSpec(name="openai", ...),         # 标准供应商
    ProviderSpec(name="deepseek", ...),
    ...
)
```

**网关排前面**，因为网关能路由任何模型，匹配优先级应该更高。

---

## 三、openai_compat_provider.py — 统一适配层

**核心思想**：绝大多数 LLM 供应商都兼容 OpenAI API 格式，所以用 **一个类适配所有**。

### 构造函数 — 通过 spec 差异化

```python
class OpenAICompatProvider(LLMProvider):
    def __init__(self, api_key, api_base, default_model, extra_headers, spec):
        # 1. spec 决定 API 地址
        effective_base = api_base or (spec.default_api_base if spec else None)

        # 2. 统一用 OpenAI SDK，只是 base_url 不同
        self._client = AsyncOpenAI(
            api_key=api_key or "no-key",
            base_url=effective_base,      # ← 关键：指向不同供应商
        )
```

```
同一个 OpenAI SDK，不同的 base_url：

DeepSeek:  base_url = "https://api.deepseek.com"
通义:      base_url = "https://dashscope.aliyuncs.com/compatible-mode/v1"
智谱:      base_url = "https://open.bigmodel.cn/api/paas/v4"
Gemini:    base_url = "https://generativelanguage.googleapis.com/v1beta/openai/"
OpenAI:    base_url = None (使用 SDK 默认值)
```

### _build_kwargs — 根据供应商特性调整请求参数

```python
def _build_kwargs(self, messages, tools, model, ...):
    spec = self._spec
    model_name = model or self.default_model

    # 1. 模型名前缀处理（AiHubMix 不认识 "anthropic/claude-3"）
    if spec and spec.strip_model_prefix:
        model_name = model_name.split("/")[-1]  # "anthropic/claude-3" → "claude-3"

    # 2. max_tokens vs max_completion_tokens（OpenAI 独有）
    if spec and spec.supports_max_completion_tokens:
        kwargs["max_completion_tokens"] = max_tokens
    else:
        kwargs["max_tokens"] = max_tokens

    # 3. 特定模型的参数覆盖（如 Kimi K2.5 强制 temperature=1.0）
    for pattern, overrides in spec.model_overrides:
        if pattern in model_name:
            kwargs.update(overrides)  # 覆盖 temperature
            break

    # 4. 供应商特有的思考模式参数
    if spec.name == "dashscope":
        extra = {"enable_thinking": thinking_enabled}
    elif spec.name in ("volcengine", "byteplus"):
        extra = {"thinking": {"type": "enabled"}}
```

### _parse — 兼容不同供应商的响应格式

不同供应商返回的 JSON 结构**略有差异**，`_parse` 方法统一处理：

```python
def _parse(self, response) -> LLMResponse:
    # 路径1：dict 格式（原始 JSON）
    if isinstance(response_map, dict):
        choices = response_map.get("choices")
        content = msg0.get("content")
        ...

    # 路径2：SDK Pydantic 对象（OpenAI SDK 返回）
    else:
        choice = response.choices[0]
        content = choice.message.content
        ...

    # 统一返回 LLMResponse
    return LLMResponse(content=content, tool_calls=..., usage=...)
```

### _extract_usage — 兼容不同供应商的 token 统计

```python
# 不同供应商的 cached_tokens 字段位置不同
for path in (
    ("prompt_tokens_details", "cached_tokens"),  # OpenAI/智谱/MiniMax
    ("cached_tokens",),                          # 阶跃星辰/月之暗面（顶层）
    ("prompt_cache_hit_tokens",),                # DeepSeek/SiliconFlow
):
    cached = self._get_nested_int(usage_map, path)
    if cached:
        result["cached_tokens"] = cached  # 统一归一化
```

### chat() 和 chat_stream() — 统一调用入口

```python
async def chat(self, messages, tools, model, ...) -> LLMResponse:
    kwargs = self._build_kwargs(...)                # 构建请求参数
    return self._parse(                             # 统一解析响应
        await self._client.chat.completions.create(**kwargs)  # 调用 API
    )

async def chat_stream(self, messages, ...) -> LLMResponse:
    kwargs = self._build_kwargs(...)
    kwargs["stream"] = True
    stream = await self._client.chat.completions.create(**kwargs)
    chunks = []
    while True:
        chunk = await asyncio.wait_for(stream.__anext__(), timeout=90)
        chunks.append(chunk)
    return self._parse_chunks(chunks)
```

---

## 四、transcription.py — 语音转文字（独立模块）

这个文件与 LLM 适配无关，是**辅助功能**，但设计思路类似：

```python
class OpenAITranscriptionProvider:        # OpenAI Whisper
    api_url = "https://api.openai.com/v1/audio/transcriptions"
    model = "whisper-1"

class GroqTranscriptionProvider:          # Groq Whisper（更快，有免费额度）
    api_url = "https://api.groq.com/openai/v1/audio/transcriptions"
    model = "whisper-large-v3"
```

两个类结构完全相同，都有 `transcribe(file_path)` 方法，只是 API 地址和模型名不同。

---

## 五、设计模式总结

```
                    ┌──────────────┐
                    │ ProviderSpec │ ← 元数据（数据驱动）
                    │  (注册表)     │
                    └──────┬───────┘
                           │ spec 传入
                           ▼
┌──────────────┐    ┌──────────────────┐    ┌──────────────┐
│  LLMProvider  │←───│OpenAICompatProvider│───→│  AsyncOpenAI  │
│  (抽象基类)   │    │   (统一适配)        │    │   (SDK客户端) │
└──────────────┘    └──────────────────┘    └──────────────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
         DeepSeek      DashScope     Gemini
         api.deepseek  dashscope...  google...
```

| 设计要点 | 说明 |
|---|---|
| **数据驱动** | 供应商差异用 `ProviderSpec` 描述，不写 if/else |
| **一个类适配所有** | `OpenAICompatProvider` 处理 20+ 供应商 |
| **SDK 复用** | 统一用 OpenAI 的 `AsyncOpenAI` SDK，只换 `base_url` |
| **双格式解析** | `_parse` 同时支持 dict 和 Pydantic 对象 |
| **扩展简单** | 新增供应商只需加一个 `ProviderSpec`，无需改代码 |

**一句话总结**：nanobot 的多供应商适配，本质就是 **"OpenAI 协议已成为事实标准，换 URL 就是一家新供应商"**。
