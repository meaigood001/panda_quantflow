# Python 代码渲染成界面节点学习指南

## 概述

本文档详细介绍 PandaAI QuantFlow 平台中，Python 代码是如何被渲染成可视化界面节点的完整流程。通过理解这个流程，您可以更好地开发自定义插件和扩展系统功能。

## 整体架构图

```
Python 插件代码 → 装饰器注册 → 动态加载 → API 转换 → 前端渲染
     ↓              ↓           ↓          ↓         ↓
  定义插件类      注册到字典    扫描导入   JSON Schema  可视化节点
```

## 1. 插件定义阶段

### 1.1 基础插件结构

所有插件都必须继承 `BaseWorkNode` 基类，并使用 `@work_node()` 装饰器注册。

```python
from panda_plugins.base import BaseWorkNode, work_node
from pydantic import BaseModel

# 1. 定义输入模型
class InputModel(BaseModel):
    data: list
    parameter: float = 1.0

# 2. 定义输出模型  
class OutputModel(BaseModel):
    result: list
    processed_count: int

# 3. 创建插件类
@work_node(
    name="数据处理节点",           # 节点显示名称
    group="数据处理/基础操作",     # 节点分组（支持层级）
    type="data_processing",       # 节点类型
    box_color="blue"             # 节点颜色
)
class DataProcessingNode(BaseWorkNode):
    """数据处理节点示例"""
    
    # 4. 定义输入模型
    @classmethod
    def input_model(cls):
        return InputModel
    
    # 5. 定义输出模型
    @classmethod  
    def output_model(cls):
        return OutputModel
    
    # 6. 实现核心逻辑
    def run(self, input: InputModel) -> OutputModel:
        self.log_info(f"开始处理 {len(input.data)} 条数据")
        
        # 处理逻辑
        result = [x * input.parameter for x in input.data]
        
        self.log_info(f"处理完成，输出 {len(result)} 条结果")
        return OutputModel(
            result=result,
            processed_count=len(result)
        )
```

### 1.2 关键要素说明

| 要素 | 说明 | 示例 |
|------|------|------|
| **输入模型** | 定义节点接收的数据结构 | `class InputModel(BaseModel)` |
| **输出模型** | 定义节点输出的数据结构 | `class OutputModel(BaseModel)` |
| **装饰器** | 注册节点并设置元信息 | `@work_node(name="节点名")` |
| **核心方法** | 实现节点业务逻辑 | `def run(self, input)` |

## 2. 装饰器注册机制

### 2.1 work_node 装饰器

位置：`src/panda_plugins/base/work_node_registery.py`

```python
def work_node(
    name: Optional[str],                    # 节点显示名称
    group: Optional[str] = "自定义节点",    # 分组路径
    order: Optional[int] = 1,              # 顺序（已废弃）
    type: Optional[str] = "general",       # 节点类型
    box_color: Optional[str] = "black"     # 节点颜色
) -> Callable:
```

### 2.2 注册流程

```python
def decorator(cls: Type[BaseWorkNode]) -> Type[BaseWorkNode]:
    # 1. 类型检查
    if not issubclass(cls, BaseWorkNode):
        raise TypeError(f"Node {cls.__name__} must inherit from BaseWorkNode")
    
    # 2. 设置类属性
    setattr(cls, "__work_node_name__", cls.__name__)
    setattr(cls, "__work_node_display_name__", name)
    setattr(cls, "__work_node_group__", group)
    # ... 更多属性设置
    
    # 3. 注册到全局字典
    ALL_WORK_NODES[cls.__name__] = cls
    return cls
```

### 2.3 全局注册表

```python
# 所有已注册的节点存储在这里
ALL_WORK_NODES: Dict[str, Type[BaseWorkNode]] = {}

# 例如：
# {
#     "DataProcessingNode": <class 'DataProcessingNode'>,
#     "FactorAnalysisNode": <class 'FactorAnalysisNode'>,
#     ...
# }
```

## 3. 动态加载机制

### 3.1 插件目录结构

```
src/panda_plugins/
├── internal/          # 内置插件
│   ├── factor_analysis/
│   ├── machine_learning/
│   └── ...
└── custom/            # 用户自定义插件
    ├── examples/
    └── your_plugins/
```

### 3.2 加载流程

位置：`src/panda_plugins/utils/work_node_loader.py`

```python
def load_all_nodes():
    """加载所有插件节点"""
    # 1. 获取插件根目录
    plugins_root = pathlib.Path(__file__).parent.parent
    
    # 2. 分别加载内置和自定义插件
    internal_folder = os.path.join(plugins_root, "internal")
    custom_folder = os.path.join(plugins_root, "custom")
    
    # 3. 扫描并加载
    load_all_nodes_from_folder(internal_folder)
    load_all_nodes_from_folder(custom_folder)

def load_all_nodes_from_folder(folder_path: str):
    """从文件夹加载所有插件"""
    # 1. 添加到 Python 路径
    sys.path.append(folder_path)
    
    # 2. 遍历所有 Python 文件
    for root, _, files in os.walk(folder_path):
        for fname in files:
            if fname.endswith(".py") and not fname.startswith("__"):
                # 3. 动态导入模块
                fpath = os.path.join(root, fname)
                modulename = os.path.splitext(fname)[0]
                
                spec = importlib.util.spec_from_file_location(modulename, fpath)
                if spec and spec.loader:
                    module = importlib.util.module_from_spec(spec)
                    spec.loader.exec_module(module)  # 执行模块代码
```

### 3.3 加载时机

```python
# 在服务器启动时加载
# src/panda_server/main.py
@asynccontextmanager
async def lifespan(app: FastAPI):
    # ...
    logger.info("Loading work nodes...")
    load_all_nodes()  # 加载所有插件
    logger.info("Work nodes loading completed")
    # ...
```

## 4. API 转换机制

### 4.1 API 接口

位置：`src/panda_server/routes/plugins_routes.py`

```python
@router.get("/all", response_model=AllPluginsResponse)
async def get_all_plugins():
    """获取所有可用的插件"""
    return await get_all_plugins_logic()
```

### 4.2 转换逻辑

位置：`src/panda_server/logic/get_all_plugins_logic.py`

```python
async def get_all_plugins_logic():
    all_plugins = []
    
    # 1. 遍历所有已注册的节点
    for name, node_class in ALL_WORK_NODES.items():
        # 2. 创建插件信息对象
        plugin_info = PluginInfo(
            name=getattr(node_class, "__work_node_name__"),
            display_name=getattr(node_class, "__work_node_display_name__"),
            group=getattr(node_class, "__work_node_group__"),
            # ...
        )
        
        # 3. 获取输入模型的 JSON Schema
        input_model = node_class.input_model()
        if input_model:
            plugin_info.input_schema = input_model.model_json_schema()
        
        # 4. 获取输出模型的 JSON Schema
        output_model = node_class.output_model()
        if output_model:
            plugin_info.output_schema = output_model.model_json_schema()
        
        all_plugins.append(plugin_info)
    
    # 5. 构建层级结构
    formatted_plugins = build_nested_structure(group_tree)
    
    return AllPluginsResponse(data=formatted_plugins)
```

### 4.3 JSON Schema 生成

Pydantic 模型自动生成 JSON Schema：

```python
# 输入模型
class InputModel(BaseModel):
    data: list
    parameter: float = 1.0

# 自动生成的 JSON Schema
{
    "type": "object",
    "properties": {
        "data": {
            "type": "array",
            "items": {}
        },
        "parameter": {
            "type": "number",
            "default": 1.0
        }
    },
    "required": ["data"]
}
```

### 4.4 层级结构构建

```python
def build_nested_structure(tree_node):
    """构建嵌套的层级结构"""
    result = []
    
    # 1. 按组名排序
    sorted_groups = sorted(tree_node.keys())
    
    for group_name in sorted_groups:
        group_data = tree_node[group_name]
        children = []
        
        # 2. 先添加子组
        if group_data['subgroups']:
            subgroup_items = build_nested_structure(group_data['subgroups'])
            children.extend(subgroup_items)
        
        # 3. 再添加插件
        if group_data['plugins']:
            sorted_plugins = sorted(group_data['plugins'], key=lambda p: p.display_name)
            children.extend(sorted_plugins)
        
        # 4. 创建组对象
        group_item = PluginGroup(
            name=group_name,
            children=children
        )
        
        result.append(group_item)
    
    return result
```

## 5. 前端渲染机制

### 5.1 数据获取

前端通过 API 获取插件数据：

```javascript
// 伪代码示例
async function loadPlugins() {
    const response = await fetch('/api/plugins/all');
    const data = await response.json();
    
    // data 结构：
    // {
    //   "data": [
    //     {
    //       "name": "数据处理",
    //       "children": [
    //         {
    //           "name": "基础操作",
    //           "children": [
    //             {
    //               "object_type": "plugin",
    //               "display_name": "数据处理节点",
    //               "input_schema": {...},
    //               "output_schema": {...}
    //             }
    //           ]
    //         }
    //       ]
    //     }
    //   ]
    // }
}
```

### 5.2 节点渲染

前端根据 JSON Schema 动态生成节点界面：

```javascript
// 伪代码示例
function renderNode(plugin) {
    // 1. 创建节点容器
    const node = createNodeElement(plugin.display_name);
    
    // 2. 根据输入 schema 生成输入端口
    plugin.input_schema.properties.forEach(prop => {
        const inputPort = createInputPort(prop.name, prop.type);
        node.addInputPort(inputPort);
    });
    
    // 3. 根据输出 schema 生成输出端口
    plugin.output_schema.properties.forEach(prop => {
        const outputPort = createOutputPort(prop.name, prop.type);
        node.addOutputPort(outputPort);
    });
    
    // 4. 设置节点样式
    node.setColor(plugin.box_color);
    
    return node;
}
```

## 6. 完整示例

### 6.1 创建自定义插件

```python
# src/panda_plugins/custom/my_analysis/stock_filter.py

from panda_plugins.base import BaseWorkNode, work_node
from pydantic import BaseModel
from typing import List, Optional

class StockFilterInput(BaseModel):
    """股票筛选输入"""
    stock_list: List[str]  # 股票代码列表
    min_price: Optional[float] = 0.0  # 最低价格
    max_price: Optional[float] = 1000.0  # 最高价格
    market_cap: Optional[str] = "all"  # 市值筛选

class StockFilterOutput(BaseModel):
    """股票筛选输出"""
    filtered_stocks: List[str]  # 筛选后的股票列表
    filter_count: int  # 筛选数量
    filter_criteria: dict  # 筛选条件

@work_node(
    name="股票筛选器",
    group="股票分析/数据筛选",
    type="stock_filter",
    box_color="green"
)
class StockFilterNode(BaseWorkNode):
    """股票筛选节点"""
    
    @classmethod
    def input_model(cls):
        return StockFilterInput
    
    @classmethod
    def output_model(cls):
        return StockFilterOutput
    
    def run(self, input: StockFilterInput) -> StockFilterOutput:
        self.log_info(f"开始筛选股票，总数: {len(input.stock_list)}")
        
        # 模拟筛选逻辑
        filtered = []
        for stock in input.stock_list:
            # 这里应该调用真实的数据接口
            # 模拟价格筛选
            if self._check_price_criteria(stock, input.min_price, input.max_price):
                filtered.append(stock)
        
        result = StockFilterOutput(
            filtered_stocks=filtered,
            filter_count=len(filtered),
            filter_criteria={
                "min_price": input.min_price,
                "max_price": input.max_price,
                "market_cap": input.market_cap
            }
        )
        
        self.log_info(f"筛选完成，符合条件的股票: {len(filtered)} 只")
        return result
    
    def _check_price_criteria(self, stock: str, min_price: float, max_price: float) -> bool:
        """检查价格条件（模拟）"""
        # 实际应用中应该调用数据接口获取真实价格
        import random
        price = random.uniform(1, 100)  # 模拟价格
        return min_price <= price <= max_price
```

### 6.2 插件注册流程

1. **服务器启动** → `main.py:lifespan()`
2. **加载插件** → `load_all_nodes()`
3. **扫描文件** → `load_all_nodes_from_folder()`
4. **导入模块** → `importlib.util.spec_from_file_location()`
5. **执行装饰器** → `@work_node()` 注册到 `ALL_WORK_NODES`
6. **API 提供** → `/api/plugins/all` 接口
7. **前端获取** → JSON 格式的插件信息
8. **界面渲染** → 根据 Schema 生成可视化节点

## 7. 调试技巧

### 7.1 查看已注册节点

```python
# 在 Python 控制台中
from panda_plugins.base.work_node_registery import ALL_WORK_NODES

print(f"已注册节点数量: {len(ALL_WORK_NODES)}")
for name, node_class in ALL_WORK_NODES.items():
    print(f"- {name}: {getattr(node_class, '__work_node_display_name__')}")
```

### 7.2 测试单个插件

```python
# 创建测试脚本
from your_plugin import YourPluginNode
from your_plugin import InputModel

# 实例化节点
node = YourPluginNode()

# 创建测试输入
test_input = InputModel(data=[1, 2, 3, 4, 5])

# 执行节点
result = node.run(test_input)
print(result)
```

### 7.3 查看生成的 JSON Schema

```python
from your_plugin import InputModel, OutputModel

print("输入 Schema:")
print(InputModel.model_json_schema(indent=2))

print("\n输出 Schema:")
print(OutputModel.model_json_schema(indent=2))
```

## 8. 最佳实践

### 8.1 插件设计原则

1. **单一职责**：每个插件只做一件事
2. **输入明确**：清晰定义输入参数和类型
3. **输出稳定**：保证输出格式的一致性
4. **错误处理**：完善的异常处理和日志记录
5. **性能考虑**：避免长时间阻塞操作

### 8.2 命名规范

- **类名**：使用 PascalCase，以 `Node` 结尾
- **文件名**：使用 snake_case
- **分组**：使用层级结构，如 `数据处理/清洗/缺失值处理`

### 8.3 日志规范

```python
def run(self, input: InputModel) -> OutputModel:
    # 开始日志
    self.log_info("开始执行数据处理", 
                  input_size=len(input.data),
                  parameters=input.parameters)
    
    try:
        # 处理逻辑
        result = self.process_data(input)
        
        # 成功日志
        self.log_info("数据处理完成", 
                      output_size=len(result),
                      execution_time=elapsed_time)
        
        return result
        
    except Exception as e:
        # 错误日志
        self.log_error("数据处理失败", 
                       error=str(e),
                       error_type=type(e).__name__)
        raise
```

## 9. 常见问题

### Q1: 插件没有出现在界面中？

**可能原因：**
1. 插件文件没有放在正确的目录
2. 装饰器使用错误
3. 继承了错误的基类
4. 代码语法错误

**解决方法：**
1. 检查文件路径：`src/panda_plugins/custom/your_plugin/`
2. 确认装饰器：`@work_node(name="节点名")`
3. 确认继承：`class YourNode(BaseWorkNode)`
4. 查看服务器日志中的错误信息

### Q2: 输入输出参数不正确？

**可能原因：**
1. Pydantic 模型定义错误
2. 类型注解不正确
3. 必填字段未标记

**解决方法：**
1. 检查模型定义：`class InputModel(BaseModel)`
2. 确认类型注解：`data: List[str]`
3. 标记必填字段：`required_field: str`（无默认值）

### Q3: 节点执行失败？

**可能原因：**
1. `run()` 方法逻辑错误
2. 输入数据格式不匹配
3. 依赖库未安装

**解决方法：**
1. 添加详细日志：`self.log_info()`, `self.log_error()`
2. 验证输入数据：`isinstance(input.data, list)`
3. 检查依赖：`pip install required_package`

## 10. 进阶功能

### 10.1 动态配置

```python
@work_node(name="动态配置节点")
class DynamicConfigNode(BaseWorkNode):
    def run(self, input: InputModel) -> OutputModel:
        # 获取动态配置
        config = self.get_config("some_key", "default_value")
        
        # 使用配置执行逻辑
        result = self.process_with_config(input, config)
        return result
```

### 10.2 异步操作

```python
import asyncio

class AsyncNode(BaseWorkNode):
    async def run_async(self, input: InputModel) -> OutputModel:
        # 异步获取数据
        data = await self.fetch_data_async(input.url)
        
        # 处理数据
        result = self.process_data(data)
        return OutputModel(result=result)
    
    def run(self, input: InputModel) -> OutputModel:
        # 在同步方法中调用异步方法
        loop = asyncio.new_event_loop()
        asyncio.set_event_loop(loop)
        try:
            return loop.run_until_complete(self.run_async(input))
        finally:
            loop.close()
```

### 10.3 缓存机制

```python
from functools import lru_cache

class CachedNode(BaseWorkNode):
    @lru_cache(maxsize=128)
    def expensive_calculation(self, data_hash: str):
        # 耗时计算
        return complex_calculation(data_hash)
    
    def run(self, input: InputModel) -> OutputModel:
        # 计算数据哈希
        data_hash = hash(str(input.data))
        
        # 使用缓存
        result = self.expensive_calculation(data_hash)
        return OutputModel(result=result)
```

## 总结

通过本文档，您应该已经了解了 PandaAI QuantFlow 平台中 Python 代码渲染成界面节点的完整流程。这个流程主要包括：

1. **插件定义**：继承基类，定义输入输出模型
2. **装饰器注册**：使用 `@work_node()` 装饰器注册节点
3. **动态加载**：系统启动时扫描并加载所有插件
4. **API 转换**：将 Python 类转换为 JSON Schema
5. **前端渲染**：根据 Schema 生成可视化节点

掌握这个流程后，您就可以开发自己的自定义插件，扩展平台功能了。建议从简单的示例开始，逐步尝试更复杂的功能。

如有问题，可以参考 `src/panda_plugins/custom/examples/` 目录中的示例代码，或者在社区中寻求帮助。