# MCP (Model Context Protocol) 代码分析

## 架构概览

这个项目实现了完整的 MCP 客户端功能，允许连接外部 MCP 服务器并将其工具包装为原生的 agent 工具。

## 核心组件

### 1. MCP 工具包装器 (`nanobot/agent/tools/mcp.py`)

**主要功能：**
- `MCPToolWrapper` 类：将 MCP 服务器的工具包装为 nanobot 原生工具
- `connect_mcp_servers` 函数：连接配置的 MCP 服务器并注册工具
- 支持 JSON Schema 规范化，特别是处理可空类型

**关键特性：**
```python
class MCPToolWrapper(Tool):
    def __init__(self, session, server_name: str, tool_def, tool_timeout: int = 30):
        self._name = f"mcp_{server_name}_{tool_def.name}"  # 命名规范
        self._parameters = _normalize_schema_for_openai(raw_schema)  # Schema 转换
```

**Schema 规范化处理：**
- 处理可空联合类型 (`["string", "null"]` → `{type: "string", nullable: true}`)
- 支持 `oneOf`/`anyOf` 规范化
- 递归处理嵌套属性和数组项

**关键代码片段：**
```python
def _normalize_schema_for_openai(schema: Any) -> dict[str, Any]:
    """Normalize only nullable JSON Schema patterns for tool definitions."""
    if not isinstance(schema, dict):
        return {"type": "object", "properties": {}}

    normalized = dict(schema)

    raw_type = normalized.get("type")
    if isinstance(raw_type, list):
        non_null = [item for item in raw_type if item != "null"]
        if "null" in raw_type and len(non_null) == 1:
            normalized["type"] = non_null[0]
            normalized["nullable"] = True

    # 处理 oneOf/anyOf
    for key in ("oneOf", "anyOf"):
        nullable_branch = _extract_nullable_branch(normalized.get(key))
        if nullable_branch is not None:
            branch, _ = nullable_branch
            merged = {k: v for k, v in normalized.items() if k != key}
            merged.update(branch)
            normalized = merged
            normalized["nullable"] = True
            break

    # 递归处理嵌套结构
    if "properties" in normalized:
        normalized["properties"] = {
            name: _normalize_schema_for_openai(prop)
            if isinstance(prop, dict)
            else prop
            for name, prop in normalized["properties"].items()
        }

    return normalized
```

### 2. 传输协议支持

项目支持三种 MCP 传输协议：

| 协议 | 配置方式 | 使用场景 |
|------|----------|----------|
| **stdio** | `command` + `args` | 本地进程 (npx/uvx) |
| **SSE** | `url` + `headers` | Server-Sent Events |
| **streamableHttp** | `url` + `headers` | 流式 HTTP |

**自动检测机制：**
```python
# 根据配置自动选择传输类型
if cfg.command:
    transport_type = "stdio"
elif cfg.url:
    # Convention: URLs ending with /sse use SSE transport; others use streamableHttp
    transport_type = (
        "sse" if cfg.url.rstrip("/").endswith("/sse") else "streamableHttp"
    )
```

**Stdio 传输实现：**
```python
if transport_type == "stdio":
    params = StdioServerParameters(
        command=cfg.command, args=cfg.args, env=cfg.env or None
    )
    read, write = await stack.enter_async_context(stdio_client(params))
```

**SSE 传输实现：**
```python
elif transport_type == "sse":
    def httpx_client_factory(
        headers: dict[str, str] | None = None,
        timeout: httpx.Timeout | None = None,
        auth: httpx.Auth | None = None,
    ) -> httpx.AsyncClient:
        merged_headers = {
            "Accept": "application/json, text/event-stream",
            **(cfg.headers or {}),
            **(headers or {}),
        }
        return httpx.AsyncClient(
            headers=merged_headers or None,
            follow_redirects=True,
            timeout=timeout,
            auth=auth,
        )

    read, write = await stack.enter_async_context(
        sse_client(cfg.url, httpx_client_factory=httpx_client_factory)
    )
```

**StreamableHttp 传输实现：**
```python
elif transport_type == "streamableHttp":
    # Always provide an explicit httpx client so MCP HTTP transport does not
    # inherit httpx's default 5s timeout and preempt the higher-level tool timeout.
    http_client = await stack.enter_async_context(
        httpx.AsyncClient(
            headers=cfg.headers or None,
            follow_redirects=True,
            timeout=None,  # 使用工具级别的超时
        )
    )
    read, write, _ = await stack.enter_async_context(
        streamable_http_client(cfg.url, http_client=http_client)
    )
```

### 3. 工具注册系统 (`nanobot/agent/tools/registry.py`)

**智能排序：**
- 内置工具优先排序（稳定前缀）
- MCP 工具随后排序（按名称）
- 确保缓存友好的提示词

```python
def get_definitions(self) -> list[dict[str, Any]]:
    """Get tool definitions with stable ordering for cache-friendly prompts.

    Built-in tools are sorted first as a stable prefix, then MCP tools are
    sorted and appended.
    """
    definitions = [tool.to_schema() for tool in self._tools.values()]
    builtins: list[dict[str, Any]] = []
    mcp_tools: list[dict[str, Any]] = []
    for schema in definitions:
        name = self._schema_name(schema)
        if name.startswith("mcp_"):
            mcp_tools.append(schema)
        else:
            builtins.append(schema)

    builtins.sort(key=self._schema_name)
    mcp_tools.sort(key=self._schema_name)
    return builtins + mcp_tools
```

### 4. 配置架构 (`nanobot/config/schema.py`)

**MCP 服务器配置：**
```python
class MCPServerConfig(Base):
    """MCP server connection configuration (stdio or HTTP)."""

    type: Literal["stdio", "sse", "streamableHttp"] | None = None  # auto-detected if omitted
    command: str = ""  # Stdio: command to run (e.g. "npx")
    args: list[str] = Field(default_factory=list)  # Stdio: command arguments
    env: dict[str, str] = Field(default_factory=dict)  # Stdio: extra env vars
    url: str = ""  # HTTP/SSE: endpoint URL
    headers: dict[str, str] = Field(default_factory=dict)  # HTTP/SSE: custom headers
    tool_timeout: int = 30  # seconds before a tool call is cancelled
    enabled_tools: list[str] = Field(default_factory=lambda: ["*"])  # Only register these tools; accepts raw MCP names or wrapped mcp_<server>_<tool> names; ["*"] = all tools; [] = no tools
```

### 5. Agent 集成 (`nanobot/agent/loop.py`)

**懒加载连接：**
```python
async def _connect_mcp(self) -> None:
    """Connect to configured MCP servers (one-time, lazy)."""
    if self._mcp_connected or self._mcp_connecting or not self._mcp_servers:
        return

    self._mcp_connecting = True
    from nanobot.agent.tools.mcp import connect_mcp_servers
    try:
        self._mcp_stack = AsyncExitStack()
        await self._mcp_stack.__aenter__()
        await connect_mcp_servers(self._mcp_servers, self.tools, self._mcp_stack)
        self._mcp_connected = True
    except BaseException as e:
        logger.error("Failed to connect MCP servers (will retry next message): {}", e)
        if self._mcp_stack:
            try:
                await self._mcp_stack.aclose()
            finally:
                self._mcp_stack = None
        self._mcp_connecting = False
```

**资源管理：**
- 使用 `AsyncExitStack` 管理多个 MCP 连接
- 确保正确的资源清理和异常处理
- 支持失败重试机制

### 6. 错误处理和超时

**超时处理：**
```python
async def execute(self, **kwargs: Any) -> str:
    try:
        result = await asyncio.wait_for(
            self._session.call_tool(self._original_name, arguments=kwargs),
            timeout=self._tool_timeout,
        )
    except asyncio.TimeoutError:
        logger.warning("MCP tool '{}' timed out after {}s", self._name, self._tool_timeout)
        return f"(MCP tool call timed out after {self._tool_timeout}s)"
```

**取消错误处理：**
```python
except asyncio.CancelledError:
    # MCP SDK's anyio cancel scopes can leak CancelledError on timeout/failure.
    # Re-raise only if our task was externally cancelled (e.g. /stop).
    task = asyncio.current_task()
    if task is not None and task.cancelling() > 0:
        raise
    logger.warning("MCP tool '{}' was cancelled by server/SDK", self._name)
    return "(MCP tool call was cancelled)"
```

**通用异常处理：**
```python
except Exception as exc:
    logger.exception(
        "MCP tool '{}' failed: {}: {}",
        self._name,
        type(exc).__name__,
        exc,
    )
    return f"(MCP tool call failed: {type(exc).__name__})"
```

### 7. 工具过滤机制

**enabledTools 配置：**
```python
enabled_tools: list[str] = ["*"]  # 默认启用所有工具
```

**支持两种命名方式：**
- 原始 MCP 工具名：`"read_file"`
- 包装后工具名：`"mcp_filesystem_read_file"`

**过滤逻辑：**
```python
tools = await session.list_tools()
enabled_tools = set(cfg.enabled_tools)
allow_all_tools = "*" in enabled_tools
registered_count = 0
matched_enabled_tools: set[str] = set()

for tool_def in tools.tools:
    wrapped_name = f"mcp_{name}_{tool_def.name}"
    if (
        not allow_all_tools
        and tool_def.name not in enabled_tools
        and wrapped_name not in enabled_tools
    ):
        logger.debug(
            "MCP: skipping tool '{}' from server '{}' (not in enabledTools)",
            wrapped_name,
            name,
        )
        continue
    wrapper = MCPToolWrapper(session, name, tool_def, tool_timeout=cfg.tool_timeout)
    registry.register(wrapper)
    logger.debug("MCP: registered tool '{}' from server '{}'", wrapper.name, name)
    registered_count += 1
```

**未匹配工具警告：**
```python
if enabled_tools and not allow_all_tools:
    unmatched_enabled_tools = sorted(enabled_tools - matched_enabled_tools)
    if unmatched_enabled_tools:
        logger.warning(
            "MCP server '{}': enabledTools entries not found: {}. Available raw names: {}. "
            "Available wrapped names: {}",
            name,
            ", ".join(unmatched_enabled_tools),
            ", ".join(available_raw_names) or "(none)",
            ", ".join(available_wrapped_names) or "(none)",
        )
```

## 测试覆盖 (`tests/tools/test_mcp_tool.py`)

**全面的测试用例：**
- Schema 规范化测试
- 工具执行和超时测试
- 错误处理测试
- 工具过滤测试
- 命名规范测试

**Schema 规范化测试：**
```python
def test_wrapper_preserves_non_nullable_unions() -> None:
    """测试保留非空联合类型"""
    tool_def = SimpleNamespace(
        name="demo",
        description="demo tool",
        inputSchema={
            "type": "object",
            "properties": {
                "value": {
                    "anyOf": [{"type": "string"}, {"type": "integer"}],
                }
            },
        },
    )
    wrapper = MCPToolWrapper(SimpleNamespace(call_tool=None), "test", tool_def)
    assert wrapper.parameters["properties"]["value"]["anyOf"] == [
        {"type": "string"},
        {"type": "integer"},
    ]

def test_wrapper_normalizes_nullable_property_type_union() -> None:
    """测试可空类型联合规范化"""
    tool_def = SimpleNamespace(
        name="demo",
        description="demo tool",
        inputSchema={
            "type": "object",
            "properties": {
                "name": {"type": ["string", "null"]},
            },
        },
    )
    wrapper = MCPToolWrapper(SimpleNamespace(call_tool=None), "test", tool_def)
    assert wrapper.parameters["properties"]["name"] == {"type": "string", "nullable": True}
```

**超时和取消测试：**
```python
@pytest.mark.asyncio
async def test_execute_returns_timeout_message() -> None:
    """测试超时处理"""
    async def call_tool(_name: str, arguments: dict) -> object:
        await asyncio.sleep(1)
        return SimpleNamespace(content=[])

    wrapper = _make_wrapper(SimpleNamespace(call_tool=call_tool), timeout=0.01)
    result = await wrapper.execute()
    assert result == "(MCP tool call timed out after 0.01s)"

@pytest.mark.asyncio
async def test_execute_re_raises_external_cancellation() -> None:
    """测试外部取消重新抛出"""
    started = asyncio.Event()

    async def call_tool(_name: str, arguments: dict) -> object:
        started.set()
        await asyncio.sleep(60)
        return SimpleNamespace(content=[])

    wrapper = _make_wrapper(SimpleNamespace(call_tool=call_tool), timeout=10)
    task = asyncio.create_task(wrapper.execute())
    await asyncio.wait_for(started.wait(), timeout=1.0)

    task.cancel()

    with pytest.raises(asyncio.CancelledError):
        await task
```

**工具过滤测试：**
```python
@pytest.mark.asyncio
async def test_connect_mcp_servers_enabled_tools_supports_raw_names(
    fake_mcp_runtime: dict[str, object | None],
) -> None:
    """测试支持原始工具名过滤"""
    fake_mcp_runtime["session"] = _make_fake_session(["demo", "other"])
    registry = ToolRegistry()
    stack = AsyncExitStack()
    await stack.__aenter__()
    try:
        await connect_mcp_servers(
            {"test": MCPServerConfig(command="fake", enabled_tools=["demo"])},
            registry,
            stack,
        )
    finally:
        await stack.aclose()

    assert registry.tool_names == ["mcp_test_demo"]

@pytest.mark.asyncio
async def test_connect_mcp_servers_enabled_tools_supports_wrapped_names(
    fake_mcp_runtime: dict[str, object | None],
) -> None:
    """测试支持包装工具名过滤"""
    fake_mcp_runtime["session"] = _make_fake_session(["demo", "other"])
    registry = ToolRegistry()
    stack = AsyncExitStack()
    await stack.__aenter__()
    try:
        await connect_mcp_servers(
            {"test": MCPServerConfig(command="fake", enabled_tools=["mcp_test_demo"])},
            registry,
            stack,
        )
    finally:
        await stack.aclose()

    assert registry.tool_names == ["mcp_test_demo"]
```

## 配置示例

**Stdio 传输：**
```json
{
  "tools": {
    "mcpServers": {
      "filesystem": {
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/dir"]
      }
    }
  }
}
```

**HTTP 传输：**
```json
{
  "tools": {
    "mcpServers": {
      "my-remote-mcp": {
        "url": "https://example.com/mcp/",
        "headers": {
          "Authorization": "Bearer xxxxx"
        }
      }
    }
  }
}
```

**带超时配置：**
```json
{
  "tools": {
    "mcpServers": {
      "my-slow-server": {
        "url": "https://example.com/mcp/",
        "toolTimeout": 120
      }
    }
  }
}
```

**工具过滤配置：**
```json
{
  "tools": {
    "mcpServers": {
      "filesystem": {
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/dir"],
        "enabledTools": ["read_file", "mcp_filesystem_write_file"]
      }
    }
  }
}
```

**enabledTools 规则：**
- 省略 `enabledTools` 或设置为 `["*"]`：注册所有工具
- 设置为 `[]`：不注册任何工具
- 设置为非空列表：仅注册指定工具

## 设计优势

1. **兼容性**：与 Claude Desktop/Cursor 配置格式兼容
2. **可扩展性**：支持动态添加任意数量的 MCP 服务器
3. **健壮性**：完善的错误处理和超时机制
4. **灵活性**：支持工具白名单和多种传输协议
5. **集成性**：MCP 工具与内置工具无缝集成
6. **缓存友好**：工具定义稳定排序，优化提示词缓存
7. **资源管理**：使用 AsyncExitStack 确保资源正确清理
8. **智能重试**：连接失败时支持下次重试

## 架构流程

```
用户配置 → MCPServerConfig → connect_mcp_servers()
                              ↓
                    创建 MCP 客户端会话
                              ↓
                    列出可用工具 → list_tools()
                              ↓
                    过滤工具 → enabledTools 检查
                              ↓
                    包装工具 → MCPToolWrapper
                              ↓
                    注册到 ToolRegistry
                              ↓
                    Agent 调用工具
                              ↓
                    超时控制和错误处理
                              ↓
                    返回结果给 Agent
```

## 关键设计决策

1. **懒加载连接**：仅在首次使用时连接 MCP 服务器，避免启动延迟
2. **命名规范**：`mcp_{server_name}_{tool_name}` 避免命名冲突
3. **Schema 规范化**：处理不同 MCP 服务器的 Schema 差异
4. **资源隔离**：每个 MCP 连接独立管理，故障隔离
5. **渐进式错误**：工具调用失败不中断整个会话

这个 MCP 实现提供了一个强大而灵活的工具扩展系统，让 agent 能够轻松使用外部 MCP 服务器提供的各种工具能力。