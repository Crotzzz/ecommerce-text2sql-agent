<div align="center">
  <h1>「电商问数」智能数据分析 Agent</h1>
  <h4><b>shopkeeper-agent</b></h4>
  <p><em>元数据知识库 + 混合检索 + LangGraph 编排 + SQL 生成校验执行</em></p>

![Python](https://img.shields.io/badge/Python-3.14-3772AB?logo=python&logoColor=white)
![LangGraph](https://img.shields.io/badge/LangGraph-Agentic%20Workflow-1C3C3C)
![FastAPI](https://img.shields.io/badge/FastAPI-Async-009688?logo=fastapi)
![License](https://img.shields.io/badge/License-MIT-green)

</div>

## 📖 项目简介

在真实的经营分析场景里，业务同学通常不会写 SQL，数据同学也很难随时记住所有表结构、字段含义、指标口径和字段取值。把问题直接交给大模型生成 SQL，很容易出现选错表、选错字段、理解错指标和 SQL 幻觉。

`shopkeeper-agent` 要解决的正是这个问题：

- 用户用自然语言提问，例如「统计华北地区上个月的销售总额」
- 系统先从元数据知识库中召回相关字段、指标和字段取值
- 大模型基于裁剪后的上下文生成 SQL
- SQL 经 `EXPLAIN` 预检查，失败则带数据库报错自动修正一次
- 查询结果与各节点执行进度通过 SSE 实时返回前端

整个系统围绕两条主线展开：

| 主线 | 做什么 | 涉及模块 |
| --- | --- | --- |
| 元数据知识库构建 | 抽取数仓中的表、字段、指标和字段取值，分别写入结构化库、向量库和全文索引 | `MySQL` / `Qdrant` / `Elasticsearch` / `TEI` |
| 自然语言问数 | 完成多路召回、上下文补全与裁剪、SQL 生成校验执行，并流式返回 | `LangGraph` / `FastAPI` / `SSE` / `React` |

## ✨ 核心能力

- **检索 + 推理 + 生成，而不是模型直出 SQL**
  - 先召回相关字段、指标和值域，再组织上下文生成 SQL，链路更稳、更可控。
- **面向业务场景的混合检索**
  - `Qdrant`：字段与指标的语义召回，字段按「名称 / 描述 / 别名」建立多语义入口。
  - `Elasticsearch`：字段真实取值的全文检索，使用 IK 中文分词，解决「华北地区」与「华北」对不上的问题。
  - `MySQL`：保存完整、权威的结构化元数据与指标口径。
- **字段、指标、取值三路并行召回**
  - 三路互不依赖，由图并行调度，最后按实体 id 合并去重。
- **上下文补全与裁剪**
  - 自动补全指标依赖字段、回填字段真实取值、补齐表的主外键（JOIN 路径）。
  - 过滤阶段让模型只做「选择」，由程序完成结构裁剪，避免模型改写复杂结构出错。
- **SQL 生成与执行闭环**
  - `EXPLAIN` 预检查拦截表名、字段名幻觉；失败后携带数据库原始报错修正一次。
- **全链路流式可观测**
  - 节点进度、最终结果、异常统一通过 SSE 推送；日志带 `request_id`，便于串联一次请求。

## 🏗️ 系统架构

![电商问数系统架构图](docs/images/shopkeeper-agent-system-architecture.svg)

查询链路：

```text
POST /api/query
      │
      ▼
extract_keywords（jieba 抽词）
      │
      ├──► recall_column（Qdrant 语义召回） ┐
      ├──► recall_value （ES 全文召回）     ├─ 并行
      └──► recall_metric（Qdrant 语义召回） ┘
      │
      ▼
merge_retrieved_info（合并 / 补依赖字段 / 回填取值 / 补主外键）
      │
      ├──► filter_table  ┐
      └──► filter_metric ┘─ 并行
      │
      ▼
add_extra_context（日期 / 方言 / 版本）
      │
      ▼
generate_sql ──► validate_sql（EXPLAIN）
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
      run_sql             correct_sql
          │                   │
          └──────► run_sql ◄──┘
```

![查询结果页](docs/images/shopkeeper-agent-query-result.jpg)

## 🛠️ 技术栈

| 模块 | 技术 | 作用 |
| --- | --- | --- |
| 教学数仓 | `MySQL` | 模拟事实表、维度表和分析型查询环境 |
| 元数据库 | `MySQL` / `SQLAlchemy` | 保存表、字段、指标、字段指标关系等结构化元数据 |
| 向量检索 | `Qdrant` | 保存字段和指标向量，支持语义召回 |
| 全文检索 | `Elasticsearch` | 保存字段真实取值，支持关键词和值域检索 |
| Embedding | `TEI` / `BAAI/bge-large-zh-v1.5` | 将字段、指标、问题等文本转成 1024 维向量 |
| 智能体编排 | `LangGraph` | 组织多阶段问数工作流，支持并行分支与条件边 |
| 模型接入 | `LangChain` | 封装 LLM 与 Embedding 调用 |
| 后端接口 | `FastAPI` | 提供问数 API、依赖注入和生命周期管理 |
| 流式协议 | `SSE` | 实时返回节点进度、查询结果和错误消息 |
| 前端 | `React` / `Vite` / `Tailwind CSS` | 聊天式问数界面和流程展示 |
| 日志追踪 | `ContextVar` / `loguru` | 为并发请求注入 `request_id`，便于排查链路 |
| 依赖管理 | `uv` / `pnpm` | 管理 Python 后端和前端依赖 |

## 📁 项目结构

```text
ecommerce-text2sql-agent/
├── app/
│   ├── agent/            # LangGraph 图、状态、上下文和各类节点
│   ├── api/              # FastAPI 路由、依赖注入、生命周期和请求结构
│   ├── clients/          # MySQL、Qdrant、Elasticsearch、Embedding 客户端管理
│   ├── conf/             # 配置 dataclass 与配置加载工具
│   ├── core/             # 日志、request_id 上下文等通用能力
│   ├── entities/         # 更贴近业务语义的数据对象
│   ├── models/           # SQLAlchemy ORM 模型
│   ├── prompt/           # Prompt 加载工具
│   ├── repositories/     # MySQL、Qdrant、Elasticsearch 数据访问层
│   ├── scripts/          # 元数据知识库构建脚本
│   └── services/         # 元数据构建服务和问数查询服务
├── conf/                 # app_config.yaml、meta_config.yaml
├── docker/               # Docker Compose、MySQL 初始化 SQL、ES IK 插件
├── frontend/             # React + Vite + Tailwind CSS 前端项目
├── prompts/              # SQL 生成、修正、过滤等 Prompt 模板
├── main.py               # FastAPI 应用入口
└── pyproject.toml        # Python 项目依赖与工具配置
```

## 🚀 快速开始

### 1. 环境要求

- Python `>= 3.14`、`uv`
- Docker 与 Docker Compose
- Node.js 与 `pnpm`

### 2. 获取代码

```bash
git clone https://github.com/Crotzzz/ecommerce-text2sql-agent.git
cd ecommerce-text2sql-agent
```

### 3. 安装后端依赖

```bash
uv sync
```

### 4. 配置大模型 API Key

```bash
cp .env.example .env
```

在 `.env` 中填入真实密钥：

```bash
LLM_API_KEY=your_real_api_key
```

默认使用兼容 OpenAI 接口的硅基流动服务，配置见 `conf/app_config.yaml`：

```yaml
llm:
  model_name: zai-org/GLM-5.3
  api_key: ${oc.env:LLM_API_KEY}
  base_url: https://api.siliconflow.cn/v1
```

如需换成其他兼容 OpenAI API 的模型平台，修改 `model_name` 与 `base_url` 即可。

### 5. 准备 Embedding 模型

TEI 容器需要本地挂载 `BAAI/bge-large-zh-v1.5` 模型文件（体积较大，不随仓库提交）：

```bash
uv run hf download BAAI/bge-large-zh-v1.5 --local-dir docker/embedding/bge-large-zh-v1.5
```

手动下载时解压到同一目录即可。首次启动 TEI 需要加载模型，CPU 环境下约需 2~3 分钟，`/health` 返回 200 后才可用。

### 6. 启动基础服务

```bash
docker compose -f docker/docker-compose.yaml up -d
```

| 服务 | 端口 |
| --- | --- |
| MySQL | `3306` |
| Elasticsearch | `9200` |
| Kibana | `5601` |
| Qdrant | `6333` |
| Embedding (TEI) | `8081` |

`docker/mysql/meta.sql` 与 `docker/mysql/dw.sql` 会在 MySQL 容器首次启动时自动初始化元数据库与教学数仓。

### 7. 构建元数据知识库

```bash
uv run python -m app.scripts.build_meta_knowledge -c conf/meta_config.yaml
```

该步骤会把表字段元数据写入 MySQL，把字段和指标向量写入 Qdrant，并把字段真实取值写入 Elasticsearch。

> 注意：当前构建脚本不是幂等的（MySQL 为直接 INSERT，Qdrant 使用随机 uuid）。重复执行前需清空 `meta` 库相关表并删除 Qdrant 中的 `column_info_collection`、`metric_info_collection`，否则会出现主键冲突或向量点翻倍。

### 8. 启动后端

```bash
uv run fastapi dev main.py
```

接口地址：

```text
POST http://127.0.0.1:8000/api/query
```

请求示例：

```json
{
  "query": "统计华北地区的销售总额"
}
```

SSE 消息类型：

| 类型 | 含义 |
| --- | --- |
| `progress` | 节点执行进度 |
| `result` | 最终查询结果 |
| `error` | 全局异常消息 |

### 9. 启动前端

```bash
cd frontend
pnpm install
pnpm dev
```

前端默认通过 Vite 代理把 `/api` 转发到 `http://127.0.0.1:8000`。如需修改：

```bash
cd frontend
cp .env.example .env
# VITE_DEV_PROXY_TARGET=http://127.0.0.1:8000
```

## 🧩 本地化适配记录

本仓库在复现过程中，针对本地环境做了以下调整：

- **Windows 下统一使用 `127.0.0.1`**
  `localhost` 可能优先解析到 IPv6 回环 `::1`，经 WSL 端口转发访问 Qdrant 时会出现空响应 503；`conf/app_config.yaml` 中 5 处 host 已改为 `127.0.0.1`。
- **本地 MySQL 端口冲突**
  若本机已安装 MySQL 服务占用 `3306`，Docker 容器会启动失败。需先停止本机服务，或将容器端口映射改为其他端口。
- **Embedding 服务本地化**
  使用 TEI 挂载本地模型目录，避免运行时依赖外网下载；同时兼容本地 TEI 的 `/embed` 接口。
- **模型切换**
  原配置中的 `Pro/zai-org/GLM-5.1` 已在硅基流动下线，现切换为 `zai-org/GLM-5.3`。

## 🚧 能力边界

本项目聚焦智能问数主链路的工程实现，暂未覆盖生产治理能力：

- 用户登录、角色权限与数据权限控制
- 多租户隔离
- SQL 安全审计、只读白名单、查询超时与行数限制
- 查询缓存、限流与性能治理
- 系统化评测集与自动化回归
- 监控告警、链路追踪平台与灰度发布
- 多轮问数记忆、追问改写与会话管理

## 📄 来源与许可

本项目基于开源项目 [didilili/shopkeeper-agent](https://github.com/didilili/shopkeeper-agent)（MIT License）复现，并完成本地环境适配与调优。原项目 README 注明其整理自尚硅谷「大模型智能体掌柜问数」项目。

原始版权声明与许可条款保留在 [LICENSE](LICENSE) 中，本项目继续以 MIT License 发布。
