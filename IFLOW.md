# PandaAI QuantFlow 量化工作流平台

## 项目概述

**PandaAI QuantFlow** 是一个集成的量化交易和机器学习工作流平台，旨在为量化研究人员提供完整的端到端解决方案。该平台基于 Python 开发，采用微服务架构，提供可视化工作流编排、机器学习集成、因子分析与回测策略验证等功能。

### 核心技术栈
- **后端框架**: FastAPI + Uvicorn
- **数据库**: MongoDB (主数据库), Redis (缓存), MySQL (辅助数据)
- **机器学习**: XGBoost, LightGBM, TensorFlow, PyTorch, scikit-learn
- **消息队列**: RabbitMQ (云模式)
- **数据处理**: pandas, numpy
- **容器化**: Docker

### 项目架构
```
panda_quantflow/
├── src/                        # 源代码目录
│   ├── common/                 # 通用工具和配置
│   │   ├── config/            # 配置管理
│   │   ├── connector/         # 数据库连接器
│   │   ├── logging/           # 日志系统
│   │   └── utils/             # 工具函数
│   ├── panda_backtest/        # 回测系统
│   │   ├── api/               # API接口
│   │   ├── backtest_common/   # 回测通用组件
│   │   ├── strategy/          # 策略模块
│   │   └── data/              # 数据模块
│   ├── panda_ml/              # 机器学习组件
│   ├── panda_plugins/         # 插件系统
│   │   ├── base/              # 插件基类
│   │   ├── internal/          # 内置插件
│   │   └── custom/            # 自定义插件
│   ├── panda_server/          # API服务
│   │   ├── config/            # 服务器配置
│   │   ├── models/            # 数据模型
│   │   ├── routes/            # 路由
│   │   └── services/          # 服务层
│   ├── panda_trading/         # 交易执行系统
│   ├── panda_schedule/        # 任务调度
│   └── panda_web/             # 前端静态资源
├── user_data/                  # 用户数据目录
├── pyproject.toml             # 项目配置和依赖
└── Dockerfile                 # Docker配置
```

## 构建与运行

### 环境要求
- Python >= 3.12
- MongoDB (副本集模式)
- Redis
- (可选) MySQL
- (可选) RabbitMQ (云模式)

### 安装与运行

#### 1. 从源码安装
```bash
# 克隆项目
git clone https://github.com/PandaAI-Tech/panda_quantflow.git
cd panda_quantflow

# 安装依赖
pip install -e .

# 启动服务
python src/panda_server/main.py
```

#### 2. Docker 方式
```bash
# 构建镜像
docker build -t panda_quantflow .

# 运行容器
docker run -p 8000:8000 panda_quantflow
```

### 访问地址
- 工作流界面: http://127.0.0.1:8000/quantflow/
- 超级图表: http://127.0.0.1:8000/charts/

### 运行模式
- **LOCAL模式**: 直接操作数据库，适合开发和小规模部署
- **CLOUD模式**: 使用RabbitMQ队列，支持分布式部署

## 开发约定

### 插件开发
- 自定义插件需继承 `BaseWorkNode` 类
- 必须实现 `input_model()`, `output_model()` 和 `run()` 方法
- 使用 `@work_node` 装饰器注册插件
- 插件放置在 `src/panda_plugins/custom/` 目录下

### 插件示例
```python
from panda_plugins.base import BaseWorkNode, work_node
from pydantic import BaseModel

class InputModel(BaseModel):
    number1: int
    number2: int

class OutputModel(BaseModel):
    result: int

@work_node(name="示例-两数求和", group="测试节点")
class ExamplePluginAddition(BaseWorkNode):
    @classmethod
    def input_model(cls):
        return InputModel
    
    @classmethod
    def output_model(cls):
        return OutputModel
    
    def run(self, input: BaseModel) -> BaseModel:
        result = input.number1 + input.number2
        return OutputModel(result=result)
```

### 日志规范
- 使用 `self.logger` 记录节点执行日志
- 支持多级别日志: debug, info, warning, error, critical
- 日志会自动关联到工作流执行上下文

### 配置管理
- 配置文件位于 `src/common/config/config.py`
- 支持环境变量覆盖配置
- 主要配置项: MongoDB、Redis、MySQL连接参数

## 测试

### 运行测试
```bash
# 安装开发依赖
pip install -e ".[dev]"

# 运行测试
pytest
```

### 测试覆盖
- 单元测试覆盖核心业务逻辑
- 集成测试验证插件系统
- API测试确保接口稳定性

## 部署

### 环境变量
- `RUN_MODE`: 运行模式 (LOCAL/CLOUD)
- `SERVER_ROLE`: 服务器角色 (CONSUMER/ALL)
- `MONGO_URI`: MongoDB连接地址
- `REDIS_HOST`: Redis主机地址
- `PANDA_SERVER_WORKERS`: 服务器工作进程数

### 生产部署
1. 使用 Docker Compose 进行多服务编排
2. 配置 Nginx 反向代理
3. 设置 MongoDB 副本集
4. 配置 RabbitMQ 集群 (云模式)

## 常用命令

### 开发调试
```bash
# 启动开发服务器 (热重载)
python src/panda_server/main.py

# 查看日志
tail -f logs/panda.log

# 数据库操作
# MongoDB 连接
mongo --host localhost:27017 -u panda -p panda --authenticationDatabase admin

# Redis 连接
redis-cli -h localhost -p 6379
```

### 插件管理
```bash
# 重新加载所有插件
# (在服务启动时自动执行)

# 测试插件
python src/panda_plugins/custom/examples/your_plugin.py
```

## 注意事项

1. **数据库依赖**: 系统依赖 MongoDB 副本集模式，单实例 MongoDB 可能导致功能异常
2. **插件开发**: 自定义插件需要遵循规范，确保输入输出模型定义正确
3. **日志系统**: 使用统一的日志系统，避免直接使用 print 语句
4. **异步操作**: 插件中的异步操作需要正确处理，避免阻塞工作流执行
5. **资源管理**: 注意资源清理，特别是数据库连接和文件句柄

## 相关项目

- [PandaFactor](https://github.com/PandaAI-Tech/panda_factor): 因子分析框架
- [PandaData](https://github.com/PandaAI-Tech/panda_data): 数据服务模块

## 社区支持

- 微信群: 扫描 README 中的二维码加入
- GitHub Issues: 提交问题和建议
- 贡献指南: 查看 CONTRIBUTING.md