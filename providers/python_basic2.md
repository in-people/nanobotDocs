# Python 基础知识

## `@dataclass` 装饰器详解

### 什么是 `@dataclass`？

`@dataclass` 是 Python 3.7+ 引入的一个**装饰器**，用来自动创建类（class）的常用方法。

### 没有 @dataclass 时（传统写法）

```python
class ToolCallRequest:
    def __init__(self, id: str, name: str, arguments: dict):
        self.id = id
        self.name = name
        self.arguments = arguments
    
    def __repr__(self):
        return f"ToolCallRequest(id={self.id}, name={self.name})"
    
    def __eq__(self, other):
        if not isinstance(other, ToolCallRequest):
            return False
        return self.id == other.id and self.name == other.name
```

### 使用 @dataclass 后（简化写法）

```python
from dataclasses import dataclass

@dataclass
class ToolCallRequest:
    id: str
    name: str
    arguments: dict[str, Any]
```

## @dataclass 自动生成的方法

当你使用 `@dataclass` 装饰器时，Python 自动为你的类生成以下方法：

| 方法 | 作用 | 示例 |
|------|------|------|
| `__init__()` | 构造函数 | `obj = ToolCallRequest(id="1", name="test")` |
| `__repr__()` | 字符串表示 | `print(obj)` → `ToolCallRequest(id='1', name='test')` |
| `__eq__()` | 相等比较（比较字段值） | `obj1 == obj2` |
| `__ne__()` | 不等比较 | `obj1 != obj2` |

### `__eq__` 比较机制详解

**重要：** `@dataclass` 的 `__eq__` 方法比较的是**字段的值**，而不是对象的内存地址（指针）！

#### 普通 Python 类 vs @dataclass 的��较

```python
# 普通 Python 类（没有 @dataclass）
class ToolCallRequest:
    def __init__(self, id: str, name: str, arguments: dict):
        self.id = id
        self.name = name
        self.arguments = arguments

# 创建两个内容相同但内存地址不同的对象
tool_call1 = ToolCallRequest(id="call_123", name="search_web", arguments={"query": "test"})
tool_call2 = ToolCallRequest(id="call_123", name="search_web", arguments={"query": "test"})

print(tool_call1 == tool_call2)  # False ❌ 比较内存地址
print(id(tool_call1), id(tool_call2))  # 不同的内存地址
```

```python
# 使用 @dataclass
from dataclasses import dataclass

@dataclass
class ToolCallRequest:
    id: str
    name: str
    arguments: dict

# 创建两个内容相同但内存地址不同的对象
tool_call1 = ToolCallRequest(id="call_123", name="search_web", arguments={"query": "test"})
tool_call2 = ToolCallRequest(id="call_123", name="search_web", arguments={"query": "test"})

print(tool_call1 == tool_call2)  # True ✅ 比较字段值
print(tool_call1 is tool_call2)  # False ❌ 比较内存地址
print(id(tool_call1), id(tool_call2))  # 仍然是不同的内存地址
```

#### @dataclass 的 `__eq__` 实现原理

自动生成的 `__eq__` 方法类似于：

```python
def __eq__(self, other):
    if not isinstance(other, ToolCallRequest):
        return False
    return (
        self.id == other.id and
        self.name == other.name and
        self.arguments == other.arguments
    )
```

#### 比较过程示例

```python
@dataclass
class ToolCallRequest:
    id: str           # 第1个字段
    name: str         # 第2个字段
    arguments: dict   # 第3个字段

tool_call1 = ToolCallRequest(id="call_123", name="search_web", arguments={"query": "test"})
tool_call2 = ToolCallRequest(id="call_123", name="search_web", arguments={"query": "test"})

# tool_call1 == tool_call2 的比较过程：
# 1. 检查类型：isinstance(other, ToolCallRequest) ✓
# 2. 比较 id: "call_123" == "call_123" ✓
# 3. 比较 name: "search_web" == "search_web" ✓
# 4. 比较 arguments: {"query": "test"} == {"query": "test"} ✓
# 5. 全部相等 → 返回 True
```

#### 对比实验

```python
from dataclasses import dataclass

@dataclass
class ToolCallRequest:
    id: str
    name: str
    arguments: dict

# 实验1：完全相同
obj1 = ToolCallRequest(id="1", name="test", arguments={"key": "value"})
obj2 = ToolCallRequest(id="1", name="test", arguments={"key": "value"})
print(obj1 == obj2)  # True ✅ (字段值相等)

# 实验2：部分不同
obj3 = ToolCallRequest(id="1", name="different", arguments={"key": "value"})
print(obj1 == obj3)  # False ❌ (name 字段不同)

# 实验3：同一个对象
obj4 = obj1
print(obj1 == obj4)  # True ✅ (字段值相等)
print(obj1 is obj4)  # True ✅ (内存地址相同)
```

#### 内存地址 vs 值比较总结

| 比较方式 | @dataclass 的 `==` | `is` 操作符 |
|----------|-------------------|-------------|
| **比较内容** | 字段的值 | 内存地址 |
| **用途** | 逻辑相等性 | 对象同一性 |
| **示例** | `tool1 == tool2` → `True` | `tool1 is tool2` → `False` |

**核心要点：**
- `@dataclass` 的 `==` 比较**字段值**，逐个字段比较
- `is` 比较**内存地址**，判断是否为同一个对象
- 两个不同对象可以有相同的字段值，此时 `==` 返回 `True`，但 `is` 返回 `False`

## 实际应用示例

```python
from dataclasses import dataclass
from typing import Any

@dataclass
class ToolCallRequest:
    """A tool call request from the LLM."""
    id: str
    name: str
    arguments: dict[str, Any]
    extra_content: dict[str, Any] | None = None
    provider_specific_fields: dict[str, Any] | None = None
```

### 使用示例

```python
# 创建实例（自动生成 __init__）
tool_call = ToolCallRequest(
    id="call_123",
    name="search_web",
    arguments={"query": "Python dataclass"},
    extra_content={"index": 0}  # 可选参数
)

# 自动生成 __repr__
print(tool_call)
# 输出: ToolCallRequest(id='call_123', name='search_web', arguments={'query': 'Python dataclass'}, extra_content={'index': 0}, provider_specific_fields=None, function_provider_specific_fields=None)

# 自动生成 __eq__
tool_call2 = ToolCallRequest(id="call_123", name="search_web", arguments={"query": "Python dataclass"})
print(tool_call == tool_call2)  # True

# 访问属性
print(tool_call.name)  # "search_web"
```

## 高级特性

### 1. 默认值

```python
@dataclass
class ToolCallRequest:
    id: str                      # 必填参数
    name: str                    # 必填参数
    extra_content: dict | None = None  # 可选参数，默认 None
```

### 2. 不可变类 (frozen=True)

```python
from dataclasses import dataclass

@dataclass(frozen=True)  # frozen=True 使类不可变
class ProviderSpec:
    name: str
    api_key: str

spec = ProviderSpec(name="openai", api_key="sk-xxx")
spec.name = "changed"  # ❌ 报错：无法赋值给不可变字段
```

### 3. 继承

```python
@dataclass
class BaseRequest:
    id: str
    name: str

@dataclass
class ToolCallRequest(BaseRequest):
    arguments: dict  # 继承 + 新增字段
```

## @dataclass vs 传统类的对比

| 特性 | 传统类 | @dataclass |
|------|--------|------------|
| 代码量 | 多（需手写方法） | 少（自动生成） |
| 可读性 | 较低 | 高（专注数据） |
| 维护性 | 易出错 | 简单安全 |
| 适用场景 | 复杂逻辑 | 数据容器 |

## 总结

`@dataclass` 的核心作用：
- ✅ **减少样板代码** - 不需要手写 `__init__`, `__repr__` 等方法
- ✅ **提高可读性** - 代码更简洁，专注于数据结构
- ✅ **类型安全** - 配合类型注解使用，提供更好的类型检查
- ✅ **适用场景** - 主要用于**数据容器类**（存储数据的类）

在项目中，`@dataclass` 广泛用于定义数据结构，如 `ToolCallRequest`、`ProviderSpec` 等，这些都是纯粹的数据容器类。

## 项目中的实际应用

### providers/registry.py 中的 ProviderSpec
```python
@dataclass(frozen=True)
class ProviderSpec:
    """One LLM provider's metadata."""
    # identity
    name: str  # config field name, e.g. "dashscope"
    keywords: tuple[str, ...]  # model-name keywords for matching (lowercase)
    env_key: str  # env var for API key, e.g. "DASHSCOPE_API_KEY"
    display_name: str = ""  # shown in status display
    
    # which provider implementation to use
    backend: str = "openai_compat"
    
    # ... 其他字段
```

使用 `frozen=True` 确保提供商配置在运行时不会被意外修改。

---

## `@property` 装饰器详解

### 什么是 `@property`？

`@property` 是 Python 的一个**装饰器**，用于将方法转换为**属性**，让你可以像访问属性一样调用方法。

### 没有 @property 时（传统方法）

```python
class ChatResponse:
    def __init__(self):
        self.tool_calls = []
    
    def has_tool_calls(self) -> bool:
        """Check if response contains tool calls."""
        return len(self.tool_calls) > 0

# 使用时需要加括号调用方法
response = ChatResponse()
if response.has_tool_calls():  # ❌ 需要括号
    print("Has tool calls")
```

### 使用 @property 后（属性风格）

```python
class ChatResponse:
    def __init__(self):
        self.tool_calls = []
    
    @property
    def has_tool_calls(self) -> bool:
        """Check if response contains tool calls."""
        return len(self.tool_calls) > 0

# 使用时不需要括号，像访问属性一样
response = ChatResponse()
if response.has_tool_calls:  # ✅ 不需要括号
    print("Has tool calls")
```

## @property 的核心作用

### 1. 让方法像属性一样访问

```python
class Circle:
    def __init__(self, radius):
        self._radius = radius
    
    @property
    def area(self):
        """计算圆的面积（只读属性）"""
        return 3.14159 * self._radius ** 2

circle = Circle(5)
print(circle.area)  # ✅ 像属性一样访问，不需要括号
# 输出: 78.53975

# circle.area()  # ❌ 报错：'float' object is not callable
```

### 2. 实现只读属性

```python
class Person:
    def __init__(self, birth_year):
        self._birth_year = birth_year
    
    @property
    def age(self):
        """年龄是只读属性，无法直接设置"""
        import datetime
        current_year = datetime.datetime.now().year
        return current_year - self._birth_year

person = Person(1990)
print(person.age)  # 36 (假设当前年份2026)

# person.age = 25  # ❌ 报错：can't set attribute (无法设置)
```

### 3. 控制属性访问和验证

```python
class Student:
    def __init__(self, score):
        self._score = score
    
    @property
    def score(self):
        """获取分数"""
        return self._score
    
    @score.setter
    def score(self, value):
        """设置分数（带验证）"""
        if not isinstance(value, (int, float)):
            raise TypeError("Score must be a number")
        if not 0 <= value <= 100:
            raise ValueError("Score must be between 0 and 100")
        self._score = value

student = Student(85)
print(student.score)  # 85

student.score = 90    # ✅ 正常设置
# student.score = 150  # ❌ 报错：Score must be between 0 and 100
# student.score = "A"  # ❌ 报错：Score must be a number
```

## @property 的三个装饰器

```python
class Person:
    def __init__(self):
        self._name = ""
    
    # 1. getter（获取器）
    @property
    def name(self):
        """获取名字"""
        return self._name
    
    # 2. setter（设置器）
    @name.setter
    def name(self, value):
        """设置名字"""
        if not isinstance(value, str):
            raise TypeError("Name must be a string")
        if len(value) > 50:
            raise ValueError("Name too long")
        self._name = value
    
    # 3. deleter（删除器）
    @name.deleter
    def name(self):
        """删除名字"""
        print("Deleting name")
        self._name = ""

# 使用示例
person = Person()
person.name = "Alice"  # 调用 setter
print(person.name)     # 调用 getter，输出: Alice
del person.name        # 调用 deleter
```

## 实际应用场景

### 1. 数据验证

```python
class Email:
    def __init__(self, address):
        self._address = address
    
    @property
    def address(self):
        return self._address
    
    @address.setter
    def address(self, value):
        if "@" not in value:
            raise ValueError("Invalid email address")
        self._address = value

email = Email("user@example.com")
email.address = "new@example.com"  # ✅
# email.address = "invalid"        # ❌ 报错
```

### 2. 延迟计算

```python
class DataProcessor:
    def __init__(self, data):
        self.data = data
        self._result = None
    
    @property
    def result(self):
        """只在第一次访问时计算结果"""
        if self._result is None:
            print("Computing result...")
            self._result = len(self.data) * 2  # 复杂计算
        return self._result

processor = DataProcessor([1, 2, 3, 4, 5])
print(processor.result)  # 第一次：Computing result... → 10
print(processor.result)  # 第二次：10 (直接返回缓存)
```

### 3. API 兼容性（重构时）

```python
# 旧版本：直接使用属性
class OldVersion:
    def __init__(self):
        self.temperature = 25

# 新版本：需要添加验证逻辑，但保持 API 兼容
class NewVersion:
    def __init__(self):
        self._temperature = 25
    
    @property
    def temperature(self):
        return self._temperature
    
    @temperature.setter
    def temperature(self, value):
        if value < -273.15:
            raise ValueError("Temperature below absolute zero")
        self._temperature = value

# 用户代码不需要修改
old = OldVersion()
old.temperature = 30  # 旧版本

new = NewVersion()
new.temperature = 30  # 新版本，API 保持一致
```

## @property vs 普通方法

| 特性 | @property | 普通方法 |
|------|-----------|----------|
| **调用方式** | `obj.property` | `obj.method()` |
| **语义含义** | 表达对象的**状态/属性** | 表达对象的**行为/动作** |
| **适用场景** | 计算属性、验证、只读属性 | 执行操作、业务逻辑 |
| **示例** | `circle.area` | `circle.calculate_area()` |

## 项目中的实际应用

### providers/base.py 中的 has_tool_calls

```python
class ChatResponse:
    def __init__(self):
        self.tool_calls = []
    
    @property
    def has_tool_calls(self) -> bool:
        """Check if response contains tool calls."""
        return len(self.tool_calls) > 0

# 使用示例
response = ChatResponse()
response.tool_calls = [{"id": "1", "name": "search"}]

# 更自然的语义
if response.has_tool_calls:  # ✅ "如果有工具调用"
    print("Has tool calls")

# vs 传统方法
if response.has_tool_calls():  # ❌ "如果有工具调用()"
    print("Has tool calls")
```

## 总结

`@property` 的核心作用：
- ✅ **让方法像属性一样访问** - 不需要括号
- ✅ **更好的语义** - 表达状态而非行为
- ✅ **数据验证** - 控制属性设置
- ✅ **只读属性** - 保护数据不被修改
- ✅ **延迟计算** - 按需计算并缓存
- ✅ **API 兼容性** - 重构时保持接口不变

### 何时使用 @property：
- 计算得出的属性（如 `area`, `age`）
- 需要验证的属性设置
- 想要暴露只读数据
- 延迟计算或缓存结果

### 何时不使用：
- 操作有副作用（如修改数据）
- 执行耗时操作（如网络请求）
- 需要传递参数

在你的代码中，`has_tool_calls` 是一个**计算属性**，表示"是否有工具调用"的状态，使用 `@property` 让代码更自然易读！
