# `nanobot/agent/tools` 代码分析

## 目录结构

```
nanobot/agent/tools/
├── __init__.py
├── base.py           # 工具基类和抽象接口
├── registry.py       # 工具注册表
├── filesystem.py     # 文件系统工具（读、写、编辑、列表）
├── search.py         # 搜索工具（Glob、Grep）
├── shell.py          # Shell 命令执行工具
├── web.py            # Web 工具（搜索、抓取）
├── message.py        # 消息发送工具
├── spawn.py          # 子代理生成工具
├── cron.py           # 定时任务工具
├── mcp.py            # MCP（Model Context Protocol）集成
└── schema/           # 参数类型定义
    ├── __init__.py
    ├── primitives.py # 基础类型（String、Integer、Boolean等）
    └── advanced.py   # 高级类型（Array、Object、Union等）
```

## 核心架构

### 1. 三层架构

```
┌─────────────────────────────────────┐
│     ToolRegistry (工具注册表)        │  ← 管理所有工具
├─────────────────────────────────────┤
│        Tool (抽象基类)              │  ← 定义工具接口
├─────────────────────────────────────┤
│   具体工具实现 (ReadFileTool等)     │  ← 实际功能
└─────────────────────────────────────┘
```

### 2. 工具注册表 (registry.py)

**核心职责**：
- 工具的注册、注销、查找
- 工具定义的生成（OpenAI Function Schema）
- 参数验证和执行

**关键方法**：

| 方法 | 作用 |
|---|---|
| `register(tool)` | 注册工具到注册表 |
| `get(name)` | 根据名称获取工具 |
| `get_definitions()` | 获取所有工具的 OpenAI Schema |
| `prepare_call(name, params)` | 验证并准备工具调用 |
| `execute(name, params)` | 执行工具 |

**工具定义排序**：

```python
def get_definitions(self) -> list[dict[str, Any]]:
    """获取工具定义，内置工具在前，MCP工具在后"""
    definitions = [tool.to_schema() for tool in self._tools.values()]
    builtins: list[dict[str, Any]] = []
    mcp_tools: list[dict[str, Any]] = []

    for schema in definitions:
        name = self._schema_name(schema)
        if name.startswith("mcp_"):
            mcp_tools.append(schema)  # MCP工具
        else:
            builtins.append(schema)   # 内置工具

    builtins.sort(key=self._schema_name)
    mcp_tools.sort(key=self._schema_name)
    return builtins + mcp_tools  # 内置工具在前，MCP工具在后
```

**为什么这样排序？**

- **缓存友好**: 内置工具顺序稳定，提示词缓存命中率更高
- **性能优化**: 常用工具优先，减少 token 消耗
- **可预测性**: LLM 总是先看到内置工具

```python
  definitions = [
      # 1️⃣内置工具（字母顺序）
      {
          "type": "function",
          "function": {
              "name": "exec",
              "description": "Execute shell commands",
              "parameters": {
                  "type": "object",
                  "properties": {
                      "command": {"type": "string"}
                  }
              }
          }
      },
```


```python
  作用: 将参数转换为正确的类型

  示例:

  # 原始参数（LLM 返回的可能是字符串）
  params = {
      "path": "test.py",
      "offset": "1",      # ← 字符串
      "limit": "100"      # ← 字符串
  }

  # 类型转换后
  cast_params = tool.cast_params(params)
  # → {
  #     "path": "test.py",
  #     "offset": 1,       # ← 整数
  #     "limit": 100       # ← 整数
  # }
```

---

### 3. 工具基类 (base.py)

**Tool 抽象类**：

```python
class Tool(ABC):
    @property
    @abstractmethod
    def name(self) -> str:
        """工具名称"""
        ...

    @property
    @abstractmethod
    def description(self) -> str:
        """工具描述"""
        ...

    @property
    @abstractmethod
    def parameters(self) -> dict[str, Any]:
        """JSON Schema 参数定义"""
        ...

    @abstractmethod
    async def execute(self, **kwargs: Any) -> Any:
        """执行工具"""
        ...
```

**并发控制属性**：

```python
@property
def read_only(self) -> bool:
    """是否只读（无副作用）"""
    return False

@property
def concurrency_safe(self) -> bool:
    """是否可以并发执行"""
    return self.read_only and not self.exclusive

@property
def exclusive(self) -> bool:
    """是否独占运行"""
    return False
```

**并发策略**：

| 属性 | 值 | 行为 |
|---|---|---|
| `read_only=True` | 只读工具 | 可以与其他只读工具并发 |
| `exclusive=True` | 独占工具 | 必须单独执行 |
| 两者都 False | 普通工具 | 串行执行 |

---

### 4. 参数验证系统

**Schema 验证流程**：

```python
# 1. 类型转换（cast_params）
params = tool.cast_params({
    "count": "123",  # 字符串
    "enabled": "yes", # 字符串
})
# → {"count": 123, "enabled": True}

# 2. 参数验证（validate_params）
errors = tool.validate_params(params)
if errors:
    return f"Error: {errors}"

# 3. 执行工具
result = await tool.execute(**params)
```

**自动类型转换**：

```python
def _cast_value(self, val: Any, schema: dict[str, Any]) -> Any:
    t = self._resolve_type(schema.get("type"))

    # 字符串 → 数字
    if t in ("integer", "number") and isinstance(val, str):
        return int(val) if t == "integer" else float(val)

    # 字符串 → 布尔
    if t == "boolean" and isinstance(val, str):
        if val.lower() in ("true", "1", "yes"):
            return True
        if val.lower() in ("false", "0", "no"):
            return False

    # 任意 → 字符串
    if t == "string":
        return str(val)

    return val
```

**支持的类型验证**：

| JSON Schema 类型 | Python 类型 | 验证规则 |
|---|---|---|
| `string` | `str` | minLength, maxLength, enum |
| `integer` | `int` | minimum, maximum |
| `number` | `int, float` | minimum, maximum |
| `boolean` | `bool` | - |
| `array` | `list` | minItems, maxItems, items |
| `object` | `dict` | required, properties |

#### `validate_json_schema_value()` - 核心验证方法

**方法签名**:

```python
@staticmethod
def validate_json_schema_value(
    val: Any,                  # 要验证的值
    schema: dict[str, Any],    # JSON Schema 定义
    path: str = ""             # 参数路径（用于错误信息）
) -> list[str]:                # 返回错误列表（空表示验证通过）
```

**执行流程**:

```
1. 处理 nullable（允许 null 值）
      ↓
2. 基础类型验证（integer, string, boolean 等）
      ↓
3. 枚举值验证（enum）
      ↓
4. 数值范围验证（minimum, maximum）
      ↓
5. 字符串长度验证（minLength, maxLength）
      ↓
6. 对象属性验证（required, properties）
      ↓
7. 数组项验证（minItems, maxItems, items）
      ↓
返回错误列表
```

**验证示例**:

```python
# 基础类型验证
schema = {"type": "integer", "minimum": 1, "maximum": 100}
validate_json_schema_value(50, schema)    # → []
validate_json_schema_value(0, schema)    # → ["parameter must be >= 1"]
validate_json_schema_value(101, schema)  # → ["parameter must be <= 100"]

# 字符串长度验证
schema = {"type": "string", "minLength": 3, "maxLength": 10}
validate_json_schema_value("hello", schema)     # → []
validate_json_schema_value("hi", schema)       # → ["parameter must be at least 3 chars"]
validate_json_schema_value("hello world", schema) # → ["parameter must be at most 10 chars"]

# 对象嵌套验证
schema = {
    "type": "object",
    "properties": {
        "name": {"type": "string", "minLength": 3},
        "age": {"type": "integer", "minimum": 0}
    },
    "required": ["name", "age"]
}

validate_json_schema_value(
    {"name": "Alice", "age": 25},
    schema
)  # → []

validate_json_schema_value(
    {"name": "Al", "age": 25},
    schema, "user"
)  # → ["user.name must be at least 3 chars"]

# 数组项验证
schema = {
    "type": "array",
    "items": {"type": "integer", "minimum": 0},
    "minItems": 1
}

validate_json_schema_value([1, 2, 3], schema)  # → []
validate_json_schema_value([], schema)       # → ["parameter must have at least 1 items"]
validate_json_schema_value([1, -2], schema, "numbers")  # → ["numbers[1] must be >= 0"]
```

**核心特性**:

1. **递归验证**: 支持任意深度的嵌套对象和数组
2. **路径跟踪**: 清楚指出错误位置（如 `user.name`, `numbers[0]`）
3. **多错误收集**: 一次返回所有验证错误
4. **标准兼容**: 完全符合 JSON Schema 规范

**支持的验证规则**:

| 验证类型 | JSON Schema 关键字 | 示例 |
|---|---|---|
| **可空性** | `nullable`, `type: ["...", "null"]` | `{"type": ["string", "null"]}` |
| **类型** | `type` | `{"type": "integer"}` |
| **枚举** | `enum` | `{"enum": ["red", "green"]}` |
| **数值范围** | `minimum`, `maximum` | `{"minimum": 0, "maximum": 100}` |
| **字符串长度** | `minLength`, `maxLength` | `{"minLength": 3}` |
| **必填属性** | `required` | `{"required": ["name", "age"]}` |
| **数组长度** | `minItems`, `maxItems` | `{"minItems": 1}` |
| **嵌套验证** | `properties`, `items` | 递归验证嵌套结构 |

---

### 5. 装饰器模式 - `@tool_parameters`

**作用**: 自动注入 `parameters` 属性，避免手动编写冗长的 JSON Schema。

#### 完整示例

```python
@tool_parameters(
    tool_parameters_schema(
        path=StringSchema("The file path to edit"),
        old_text=StringSchema("The text to find and replace"),
        new_text=StringSchema("The text to replace with"),
        replace_all=BooleanSchema(description="Replace all occurrences (default false)"),
        required=["path", "old_text", "new_text"],
    )
)
class EditFileTool(_FsTool):
    """编辑文件工具"""
    async def execute(self, path: str, old_text: str, new_text: str, replace_all: bool = False):
        ...
```

#### 生成的 JSON Schema

```python
{
    "type": "object",
    "properties": {
        "path": {
            "type": "string",
            "description": "The file path to edit"
        },
        "old_text": {
            "type": "string",
            "description": "The text to find and replace"
        },
        "new_text": {
            "type": "string",
            "description": "The text to replace with"
        },
        "replace_all": {
            "type": "boolean",
            "description": "Replace all occurrences (default false)"
        }
    },
    "required": ["path", "old_text", "new_text"]
}
```

#### 对比：有装饰器 vs 无装饰器

**使用装饰器（简洁）**:

```python
@tool_parameters(
    tool_parameters_schema(
        path=StringSchema("File path"),
        content=StringSchema("File content"),
        required=["path", "content"],
    )
)
class WriteFileTool(Tool):
    async def execute(self, path: str, content: str):
        ...
```

**不使用装饰器（繁琐）**:

```python
class WriteFileTool(Tool):
    @property
    def parameters(self) -> dict[str, Any]:
        return {
            "type": "object",
            "properties": {
                "path": {
                    "type": "string",
                    "description": "File path"
                },
                "content": {
                    "type": "string",
                    "description": "File content"
                }
            },
            "required": ["path", "content"]
        }
    
    async def execute(self, path: str, content: str):
        ...
```

#### 常见 Schema 类型

| Schema 类型 | 说明 | 示例 |
|---|---|---|
| `StringSchema` | 字符串参数 | `StringSchema("User name", min_length=3)` |
| `IntegerSchema` | 整数参数 | `IntegerSchema("Age", minimum=0, maximum=150)` |
| `BooleanSchema` | 布尔参数 | `BooleanSchema("Is active")` |
| `NumberSchema` | 浮点数参数 | `NumberSchema("Price", minimum=0.0)` |
| `ArraySchema` | 数组参数 | `ArraySchema(items=StringSchema("Tag"))` |
| `ObjectSchema` | 对象参数 | `ObjectSchema(properties={...})` |

#### 装饰器的优势

| 优势 | 说明 |
|---|---|
| **类型安全** | 使用 Schema 类确保类型正确 |
| **代码简洁** | 避免手写冗长的 JSON Schema |
| **可读性强** | 参数定义一目了然 |
| **易于维护** | 修改参数只需修改一行 |
| **自动注入** | 装饰器自动实现抽象方法 |

#### `tool_parameters()` 函数实现

**函数签名**:

```python
def tool_parameters(schema: dict[str, Any]) -> Callable[[type[_ToolT]], type[_ToolT]]:
    """类装饰器：附加 JSON Schema 并注入 parameters 属性"""
```

**执行流程**:

```
1. 接收 JSON Schema
   ↓
2. 返回装饰器函数
   ↓
3. 装饰器修改类
   ├─ 深拷贝 Schema
   ├─ 定义 parameters 属性
   ├─ 存储原始 Schema
   ├─ 注入属性到类
   └─ 移除抽象方法
   ↓
4. 返回增强后的类
```

**关键步骤详解**:

```python
def decorator(cls: type[_ToolT]) -> type[_ToolT]:
    # 步骤 1: 深拷贝 Schema（防止外部修改）
    frozen = deepcopy(schema)
    
    # 步骤 2: 定义 parameters 属性
    @property
    def parameters(self: Any) -> dict[str, Any]:
        return deepcopy(frozen)  # 每次访问返回新拷贝
    
    # 步骤 3: 存储原始 Schema
    cls._tool_parameters_schema = deepcopy(frozen)
    
    # 步骤 4: 注入属性
    cls.parameters = parameters
    
    # 步骤 5: 移除抽象方法
    abstract = getattr(cls, "__abstractmethods__", None)
    if abstract is not None and "parameters" in abstract:
        cls.__abstractmethods__ = frozenset(abstract - {"parameters"})
    
    return cls
```

**为什么需要深拷贝？**

```python
# 场景: 防止外部修改影响内部状态
tool = MyTool()
params1 = tool.parameters
params1["properties"]["hack"] = "value"  # 外部修改

params2 = tool.parameters
print("hack" in params2["properties"])  # False (不受影响)
```

**自动实现抽象方法**:

```python
# 父类定义
class Tool(ABC):
    @property
    @abstractmethod
    def parameters(self):
        """子类必须实现"""
        ...

# 使用装饰器后
@tool_parameters({...})
class MyTool(Tool):
    pass  # 不需要手动实现 parameters

# 验证
from inspect import isabstract
print(isabstract(MyTool))  # False (可以实例化)
```

**核心价值**:

| 特性 | 实现 | 好处 |
|---|---|---|
| **自动化** | 自动实现抽象方法 | 无需手动编写 @property |
| **安全性** | 深拷贝防止意外修改 | 每次访问返回独立副本 |
| **持久化** | 存储原始 Schema | 便于反射和调试 |
| **类型安全** | 保持类型信息 | IDE 可以自动补全 |

#### 泛型类型变量：`_ToolT`

**函数签名中的泛型**:

```python
def tool_parameters(
    schema: dict[str, Any]
) -> Callable[[type[_ToolT]], type[_ToolT]]:
    ...
```

**TypeVar 定义**:

```python
from typing import TypeVar

_ToolT = TypeVar("_ToolT", bound="Tool")
```

**参数解析**:

| 部分 | 含义 | 作用 |
|---|---|---|
| `_ToolT` | 类型变量名称 | 表示 Tool 相关的泛型类型（内部使用） |
| `TypeVar` | 类型变量 | Python 的泛型机制 |
| `bound="Tool"` | 类型约束 | 限制为 Tool 或其子类 |

**为什么使用 TypeVar？**

```python
# 场景 1: 保持类型信息（有 TypeVar）
_ToolT = TypeVar("_ToolT", bound="Tool")

def decorator(cls: type[_ToolT]) -> _ToolT:
    return cls

result = decorator(ReadFileTool)
# result 的类型是 ReadFileTool（具体类型）✓

# 场景 2: 类型信息丢失（无 TypeVar）
def decorator(cls: type[Tool]) -> type[Tool]:
    return cls

result = decorator(ReadFileTool)
# result 的类型是 Tool（泛化类型）✗
```

**类型安全示例**:

```python
# ✓ 有效：Tool 的子类
@tool_parameters({...})
class ReadFileTool(Tool):
    pass

# ✓ 有效：Tool 的子类
@tool_parameters({...})
class WriteFileTool(Tool):
    pass

# ✗ 无效：不是 Tool 的子类（类型检查器会报错）
@tool_parameters({...})
class NotATool:
    pass
```

**核心价值**:

| 特性 | 说明 |
|---|---|
| **类型约束** | 确保只接受 Tool 类及其子类 |
| **保留类型信息** | 返回具体的子类类型，而不是泛化的 Tool |
| **IDE 支持** | 自动补全和类型推断 |
| **编译时检查** | 在代码编写阶段就发现类型错误 |

---

## 具体工具实现

### 1. 文件系统工具 (filesystem.py)

| 工具 | 功能 | 并发安全性 |
|---|---|---|
| `ReadFileTool` | 读取文件内容 | read_only=True ✓ |
| `WriteFileTool` | 写入文件 | read_only=False |
| `EditFileTool` | 编辑文件（基于diff） | read_only=False |
| `ListDirTool` | 列出目录内容 | read_only=True ✓ |

**路径安全控制**：

```python
def _resolve_path(
    path: str,
    workspace: Path | None = None,
    allowed_dir: Path | None = None,
    extra_allowed_dirs: list[Path] | None = None,
) -> Path:
    """解析路径并强制执行目录限制"""
    p = Path(path).expanduser()

    # 相对路径 → 工作区
    if not p.is_absolute() and workspace:
        p = workspace / p

    resolved = p.resolve()

    # 检查是否在允许的目录内
    if allowed_dir:
        all_dirs = [allowed_dir] + (extra_allowed_dirs or [])
        if not any(resolved.is_relative_to(d) for d in all_dirs):
            raise PermissionError(f"Path {path} is outside allowed directory")

    return resolved
```

### 2. 搜索工具 (search.py)

| 工具 | 功能 | 示例 |
|---|---|---|
| `GlobTool` | 文件名模式匹配 | `**/*.py` 查找所有 Python 文件 |
| `GrepTool` | 内容搜索 | `TODO:` 查找所有 TODO 注释 |

### 3. Shell 工具 (shell.py)

**ExecTool** - 命令执行工具：

```python
class ExecTool(Tool):
    def __init__(
        self,
        working_dir: str,
        timeout: int = 30,
        restrict_to_workspace: bool = False,
        sandbox: bool = False,
        path_append: list[str] | None = None,
    ):
        self._working_dir = working_dir
        self._timeout = timeout
        self._restrict_to_workspace = restrict_to_workspace
        self._sandbox = sandbox  # 沙箱模式
        self._path_append = path_append  # PATH 环境变量追加
```

**安全特性**：

- 超时控制（默认 30 秒）
- 工作区限制
- 沙箱模式
- PATH 环境变量控制

### 4. Web 工具 (web.py)

| 工具 | 功能 | 并发安全性 |
|---|---|---|
| `WebSearchTool` | 网络搜索 | read_only=True ✓ |
| `WebFetchTool` | 抓取网页内容 | read_only=True ✓ |

### 5. 通信工具 (message.py, spawn.py)

**MessageTool** - 发送消息到用户：

```python
class MessageTool(Tool):
    def __init__(self, send_callback: Callable[[OutboundMessage], Awaitable[None]]):
        self._send_callback = send_callback
        self._sent_in_turn = False

    async def execute(self, content: str) -> str:
        """发送消息给用户"""
        await self._send_callback(OutboundMessage(
            channel=self._channel,
            chat_id=self._chat_id,
            content=content,
        ))
        self._sent_in_turn = True
        return "Message sent to user."
```

**SpawnTool** - 创建子代理：

```python
class SpawnTool(Tool):
    def __init__(self, manager: SubagentManager):
        self._manager = manager

    async def execute(self, task: str) -> str:
        """在后台启动子代理执行任务"""
        return await self._manager.spawn(
            task=task,
            origin_channel=self._channel,
            origin_chat_id=self._chat_id,
        )
```

### 6. 定时任务工具 (cron.py)

**CronTool** - 定时任务管理：

```python
class CronTool(Tool):
    async def execute(
        self,
        schedule: str,      # Cron 表达式: "0 9 * * *"
        prompt: str,        # 要执行的任务
        recurring: bool = True,
        durable: bool = False,
    ) -> str:
        """创建定时任务"""
        job_id = await self._cron_service.create(
            cron=schedule,
            prompt=prompt,
            recurring=recurring,
            durable=durable,
        )
        return f"Job {job_id} scheduled."
```

---

## 工具执行流程

### 完整流程图

```
1. LLM 返回工具调用
   {"name": "read_file", "arguments": {"path": "test.py"}}
         ↓
2. ToolRegistry.prepare_call()
   - 查找工具: tool = registry.get("read_file")
   - 类型转换: {"path": "test.py"} → {"path": Path("test.py")}
   - 参数验证: validate_params({"path": Path("test.py")})
         ↓
3. 并发控制
   - 检查工具的 concurrency_safe 属性
   - 如果安全，与其他工具并发执行
   - 如果不安全，串行执行
         ↓
4. Tool.execute()
   result = await tool.execute(path="test.py")
         ↓
5. 结果返回
   - 成功: 返回结果字符串或内容块
   - 失败: 返回错误信息 + 提示
```

### 错误处理

```python
async def execute(self, name: str, params: dict[str, Any]) -> Any:
    """执行工具，带错误提示"""
    _HINT = "\n\n[Analyze the error above and try a different approach.]"

    # 1. 准备调用
    tool, params, error = self.prepare_call(name, params)
    if error:
        return error + _HINT  # 工具不存在或参数错误

    # 2. 执行工具
    try:
        result = await tool.execute(**params)
        if isinstance(result, str) and result.startswith("Error"):
            return result + _HINT  # 工具返回错误
        return result
    except Exception as e:
        return f"Error executing {name}: {str(e)}" + _HINT  # 执行异常
```

---

## 设计模式

### 1. 注册表模式

```python
registry = ToolRegistry()

# 注册工具
registry.register(ReadFileTool(...))
registry.register(WriteFileTool(...))

# 查找和执行
tool = registry.get("read_file")
await tool.execute(path="test.txt")
```

### 2. 策略模式

```python
# 不同的工具有不同的执行策略
class ReadFileTool(Tool):
    async def execute(self, path: str):
        return Path(path).read_text()

class GrepTool(Tool):
    async def execute(self, pattern: str, path: str):
        return self._grep(pattern, path)
```

### 3. 模板方法模式

```python
class _FsTool(Tool):
    """文件系统工具的模板基类"""

    def _resolve(self, path: str) -> Path:
        """公共的路径解析逻辑"""
        return _resolve_path(path, self._workspace, self._allowed_dir)

class ReadFileTool(_FsTool):
    async def execute(self, path: str):
        resolved = self._resolve(path)  # 使用继承的模板方法
        return resolved.read_text()
```

### 4. 装饰器模式

```python
@tool_parameters(
    tool_parameters_schema(
        path=StringSchema("The file path to read"),
        offset=IntegerSchema(1, minimum=1),
    )
)
class ReadFileTool(_FsTool):
    """装饰器自动注入 parameters 属性"""
    async def execute(self, path: str, offset: int = 1):
        ...
```

---

## 关键设计优势

### 1. 类型安全

- **自动类型转换**: 字符串 "123" → 整数 123
- **参数验证**: JSON Schema 验证所有参数
- **错误提示**: 清晰的错误信息引导 LLM 重试

### 2. 并发控制

- **只读工具**: 可以并发执行，提高效率
- **独占工具**: 确保数据一致性
- **灵活配置**: 每个工具可自定义并发行为

### 3. 安全性

- **路径限制**: 防止访问工作区外的文件
- **超时控制**: 防止长时间运行
- **沙箱模式**: 隔离执行环境

### 4. 可扩展性

- **插件化**: 通过 MCP 集成外部工具
- **继承体系**: _FsTool 等基类简化开发
- **装饰器**: @tool_parameters 简化参数定义

### 5. 性能优化

- **工具排序**: 内置工具优先，提高缓存命中率
- **并发执行**: 多个只读工具可同时运行
- **惰性验证**: 只在执行时才验证参数

---

## 总结

`nanobot/agent/tools` 目录实现了一个**功能完整、类型安全、并发可控**的工具系统：

1. **清晰的架构**: 注册表 → 基类 → 具体工具
2. **强大的验证**: JSON Schema 参数验证 + 自动类型转换
3. **灵活的并发**: 基于工具属性的智能并发控制
4. **丰富的工具**: 文件、搜索、Shell、Web、通信、定时任务
5. **安全可控**: 路径限制、超时、沙箱等安全机制
6. **易于扩展**: 装饰器、继承、MCP 集成等多种扩展方式

这个工具系统是整个 AI Agent 的核心能力来源，让 LLM 能够安全、高效地执行各种操作。
