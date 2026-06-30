# 知枢 (ZhiShu) — GraphRAG 知识管理平台

<div align="center">

![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB?logo=python)
![LangGraph](https://img.shields.io/badge/LangGraph-0.2%2B-FF6B6B)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker)

**独立开发的多Agent知识管理系统，支持多模态文档解析、知识图谱构建、GraphRAG 智能问答及分钟级增量同步。**

[快速开始](#-快速开始) · [系统架构](#-系统架构) · [功能演示](#-功能演示) · [API文档](#-api-接口)

</div>

---

## 📋 目录

- [核心亮点](#-核心亮点)
- [技术栈](#-技术栈)
- [系统架构](#-系统架构)
- [快速开始](#-快速开始)
- [功能演示](#-功能演示)
- [项目结构](#-项目结构)
- [API接口](#-api-接口)
- [关键设计决策](#-关键设计决策)
- [面试资料](#-面试资料)
- [常见问题](#-常见问题)

---

## 🎯 核心亮点

| 指标 | 优化前 | 优化后 |
|------|--------|--------|
| 新文档入库错误率 | 30% | 5% |
| 50条自建测试集 F1 | 0.68 | 0.84 |
| 更新延迟 | 全量重建(30min级) | &lt; 2min |

---

## 🛠 技术栈

| 组件 | 技术选型 | 说明 |
|------|----------|------|
| **Agent编排** | LangGraph | 设计"解析-抽取-索引-问答"4阶段状态机流水线 |
| **LLM调用** | LangChain + OpenAI | LLM应用框架 |
| **向量数据库** | ChromaDB | 多模态文档向量化存储与检索 |
| **知识图谱** | Neo4j | 实体关系存储，支持跨文档多跳推理 |
| **API框架** | FastAPI | 异步高性能REST API，自动生成Swagger文档 |
| **增量同步** | Watchdog | 分钟级文件监听与差量更新 |
| **容器化** | Docker Compose | 全链路容器化部署 |

---

## 🏗 系统架构

### 整体架构图
```
┌──────────────────────────────────────────────────────────┐
│                      用户接口层                            │
│              FastAPI REST API / Swagger Docs              │
└──────────────────────┬───────────────────────────────────┘
│
┌──────────────────────▼───────────────────────────────────┐
│           编排引擎 (LangGraph 4阶段状态机)                 │
│  ┌──────────┬──────────┬──────────┬──────────┐           │
│  │  文档解析 │  知识抽取 │  向量索引 │  智能问答 │           │
│  │ (Parse)  │ (Extract)│ (Index)  │  (Q&A)   │           │
│  └────┬─────┴────┬─────┴────┬────┴────┬─────┘           │
└───────┼──────────┼──────────┼─────────┼────────────────┘
│          │          │         │
┌───────▼──┐ ┌────▼────┐ ┌───▼───┐ ┌───▼────┐
│ 文档解析  │ │ 知识抽取 │ │ 问答  │ │ 知识更新 │
│  Agent   │ │  Agent  │ │ Agent │ │  Agent  │
│          │ │         │ │       │ │         │
│•PDF图文  │ │•NER实体  │ │•向量  │ │•Watchdog│
│ 混排解析  │ │ 识别    │ │ 检索  │ │ 文件监听 │
│•Markdown │ │•关系抽取 │ │(Chroma│ │•差量对比 │
│ 代码块提取 │ │•三元组  │ │ DB)   │ │•增量更新 │
│•多模态   │ │ 生成    │ │•图谱  │ │         │
│ 分块     │ │         │ │ 检索  │ │         │
│          │ │         │ │(Neo4j)│ │         │
│          │ │         │ │•加权  │ │         │
│          │ │         │ │ 融合  │ │         │
│          │ │         │ │ 重排序 │ │         │
│          │ │         │ │•答案  │ │         │
│          │ │         │ │ 生成  │ │         │
└────┬─────┘ └────┬────┘ └───┬───┘ └────┬────┘
│            │          │          │
│            │          │          │
┌────▼────────────▼──────────▼──────────▼────┐
│                  存储层                      │
│  ┌────────────────┐  ┌────────────────┐    │
│  │    ChromaDB     │  │     Neo4j      │    │
│  │   向量数据库      │  │    知识图谱      │    │
│  │                 │  │                 │    │
│  │ • 多模态向量化    │  │ • 实体-关系-属性 │    │
│  │   分别存储       │  │   三元组        │    │
│  │ • 加权融合检索    │  │ • 跨文档多跳推理 │    │
│  └────────────────┘  └────────────────┘    │
└────────────────────────────────────────────┘
```
### 三条工作流水线

**流水线1：文档入库**（上传文档时触发）
```
用户上传文档
│
▼
文档解析Agent
├── PDF图文混排解析
├── Markdown代码块提取
└── 多模态分块
│
▼
知识抽取Agent
├── NER实体识别
├── 关系抽取
└── 生成三元组：("张三", "就职于", "腾讯")
│
├──────────────────────┐
▼                      ▼
存入ChromaDB              存入Neo4j
(向量数据库)              (知识图谱)
```

**流水线2：智能问答**（用户提问时触发）
```
用户提问："张三负责什么业务？"
│
▼
意图识别 + 查询改写
│
├──────────────────┐
▼                  ▼
向量检索(ChromaDB)    图谱检索(Neo4j)
(语义相似度)          (关系路径查询)
│                  │
└────────┬─────────┘
▼
加权融合重排序
│
▼
LLM生成答案
```

**流水线3：增量更新**（文档修改时触发）
```
文档被修改
│
▼
Watchdog文件监听
│
▼
知识更新Agent
├── 差量分析：定位变化内容
├── 增量解析：只重新处理变更部分
└── 版本管理
│
├──────────────┐
▼              ▼
更新ChromaDB      更新Neo4j
```

---

## 🚀 快速开始

### 前置条件

- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- OpenAI API Key（或国内兼容接口）

### 步骤1：克隆项目

```bash
git clone https://github.com/Awsfgd/zhishu-graphrag.git
cd zhishu-graphrag
```
### 步骤2：配置环境变量
```bash
cp .env.example .env
```
用任意编辑器打开 `.env`，填入你的配置：
```env
# OpenAI配置

OPENAI_API_KEY=sk-你的APIKey
OPENAI_BASE_URL=https://api.openai.com/v1

# 数据库配置

NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=password
CHROMA_HOST=localhost
CHROMA_PORT=8000
```
### 步骤3：启动依赖服务
```bash
docker-compose up -d
```
检查状态：
```bash
docker-compose ps
```
### 步骤4：启动API服务
```bash
pip install -r requirements.txt
python -m api.main
```
看到 Uvicorn running on http://0.0.0.0:8080 即成功。
### 步骤5：验证
访问 http://localhost:8080/docs 查看Swagger文档。
```bash
# 健康检查
curl http://localhost:8080/api/health

# 上传文档
curl -X POST http://localhost:8080/api/ingest/upload \
  -F "file=@你的文档.pdf"

# 提问
curl -X POST http://localhost:8080/api/qa/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "这个文档讲了什么？"}'
```
## 🎬 功能演示

### 功能1：多模态文档解析
文档解析Agent可以自动识别文件类型，调用对应的解析器：
```python
from agents.doc_parser_agent import DocParserAgent

agent = DocParserAgent()

# PDF图文混排解析
chunks = await agent.parse("年度报告.pdf")
# Markdown代码块提取
chunks = await agent.parse("技术文档.md")

# 每个chunk包含：
# chunk.text      - 文本内容
# chunk.metadata  - 来源文件、页码、类型等
# chunk.embedding - 向量表示（自动生成）
```
### 功能2：知识图谱自动构建

知识抽取Agent从文本中提取三元组，自动构建知识图谱：
```python
from agents.knowledge_extract_agent import KnowledgeExtractAgent

extractor = KnowledgeExtractAgent()
result = await extractor.extract(chunks)

# 输出示例：
# entities:
#   - ("张三", Person, {"职位": "CEO"})
#   - ("腾讯", Organization, {"行业": "互联网"})
# relations:
#   - ("张三", "就职于", "腾讯")
```
在Neo4j浏览器（http://localhost:7474）中可视化查看。
### 功能3：GraphRAG 混合检索问答

问答Agent同时从向量库和知识图谱中检索，结合两个来源的信息生成答案：

```python
from agents.qa_agent import QAAgent
from services.vector_store import VectorStore
from services.knowledge_graph import KnowledgeGraphService

vs = VectorStore()
kg = KnowledgeGraphService()
qa = QAAgent(vector_store=vs, knowledge_graph=kg)

# 支持跨文档多跳推理
result = await qa.answer("张三负责的产品，主要竞争对手是谁？")

print(result.answer)      # 自然语言答案
print(result.sources)     # 来源引用
print(result.confidence)  # 置信度

# 内部流程：
# 向量检索 → ChromaDB语义相似度
# 实体链接 → 识别问题中的实体
# 图谱检索 → Neo4j关系路径查询
# 加权融合重排序 → 弥补纯向量检索语义断层
# LLM生成 → 结构化答案
```
### 功能4：Watchdog 增量更新

```python
from agents.knowledge_update_agent import KnowledgeUpdateAgent

update_agent = KnowledgeUpdateAgent()

# 场景：修改了PDF第3页
# ❌ 传统全量更新：删除1000条向量 → 重解析50页 → 重写1000条，耗时~30min
# ✅ Watchdog增量：只检测第3页变化 → 局部解析 → 差量更新入库，耗时<2min

await update_agent.process_event(event={
    "operation": "UPDATE",
    "resource_path": "/docs/年度报告.pdf",
    "changed_pages": [3]
})
```
---

## 📁 项目结构

```
zhishu-graphrag/
│
├── README.md                          ← 项目说明
├── docker-compose.yml                 ← 一键启动依赖服务（Neo4j + ChromaDB）
│
├── docs/                              ← 文档目录
│   ├── architecture.md                ← 架构设计详解
│   ├── interview-guide.md             ← 面试八股文 + STAR话术
│   ├── resume-template.md             ← 简历写法模板
│   └── images/
│       └── zhishu_architecture.png    ← 系统架构图（PNG备份）
│
├── agents/                            ← 4个核心Agent
│   ├── doc_parser_agent.py            ← 文档解析Agent（多模态）
│   ├── knowledge_extract_agent.py     ← 知识抽取Agent（NER + 关系抽取）
│   ├── qa_agent.py                    ← 问答Agent（GraphRAG混合检索）
│   └── knowledge_update_agent.py      ← 增量更新Agent（Watchdog）
│
├── orchestrator/
│   └── graph.py                       ← LangGraph状态机编排引擎
│
├── services/                          ← 核心服务
│   ├── vector_store.py                ← ChromaDB向量库服务
│   ├── knowledge_graph.py            ← Neo4j知识图谱服务
│   ├── graph_rag.py                   ← GraphRAG混合检索管道
│   └── cdc_processor.py               ← Watchdog增量处理器
│
├── api/
│   └── main.py                        ← FastAPI入口
│
├── config/
│   └── settings.py                    ← 配置管理
│
├── Dockerfile                         ← Python服务容器化
├── requirements.txt                   ← Python依赖
└── .env.example                       ← 环境变量模板
```
---

## 📡 API 接口
启动后访问 http://localhost:8080/docs
### 文档管理接口
| 方法     | 路径                   | 说明     |
| ------ | -------------------- | ------ |
| `POST` | `/api/ingest/upload` | 上传单个文档 |
| `POST` | `/api/ingest/batch`  | 批量上传文档 |

### 智能问答接口
| 方法     | 路径            | 说明   |
| ------ | ------------- | ---- |
| `POST` | `/api/qa/ask` | 智能问答 |
**请求示例：**
```json
{"question": "张三的职位？", "top_k": 5}
```
**响应示例：**
```json
{
  "answer": "根据文档，张三担任腾讯公司CEO职务。",
  "confidence": 0.94,
  "sources": [
    {"doc": "年度报告.pdf", "page": 3, "type": "vector"},
    {"entity": "张三", "relation": "就职于", "target": "腾讯", "type": "graph"}
  ]
}
```
### 管理接口
| 方法    | 路径                 | 说明                |
| ----- | ------------------ | ----------------- |
| `GET` | `/api/admin/stats` | 系统统计（文档数、实体数、关系数） |
| `GET` | `/api/health`      | 健康检查              |

---
## 🔑 关键设计决策
### 对应简历中的5个核心亮点：
- 状态机架构：初期采用单Agent串行方案，因阻塞问题改用LangGraph 4阶段状态机，实现失败隔离与自动重试，新文档入库错误率从30%降至5%。
- 多模态RAG：PDF图文混排与Markdown代码块分别向量化后加权融合检索，50条自建测试集F1从0.68提升至0.84。
- 轻量级知识图谱：基于Neo4j提取实体关系，实现跨文档多跳推理，弥补纯向量检索的语义断层。
- 增量同步：采用Watchdog实现分钟级文件监听，评估数据量后未引入Kafka避免过度设计，更新延迟<2min。
- Docker全链路部署：容器化运行Neo4j + ChromaDB + FastAPI，稳定运行3个月。
---
## ❓ 常见问题
### Q: 没有OpenAI API Key怎么办？
可使用国内兼容接口：
```env
# 通义千问
OPENAI_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
OPENAI_API_KEY=你的Key

# 本地部署（Ollama）
OPENAI_BASE_URL=http://localhost:11434/v1
OPENAI_API_KEY=ollama
OPENAI_MODEL=qwen2
```
### Q: Docker启动报错？
```bash
docker-compose ps
docker-compose logs neo4j
docker-compose restart neo4j
```
Neo4j建议分配至少4GB内存（Docker Desktop → Settings → Resources → Memory）。
### Q: 这个项目能直接用于生产环境吗？
#### 这是一个架构展示 + 学习项目。生产环境需补充：
- 用户认证（JWT / OAuth2）
- API限流与熔断
- 日志监控（Prometheus / Grafana）
- 数据备份方案
