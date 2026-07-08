# engine

在 **[AgentScope](https://github.com/agentscope-ai/agentscope)** 之上扩展的、
**类 LangGraph** 的 Agent 图编排框架，并额外提供一套**面向前端可视化**的易用接口
与 REST 服务。

AgentScope 原生只提供 `sequential_pipeline` / `fanout_pipeline` / `MsgHub` 等线性编排；
本项目补齐了 LangGraph 式的**图编排**能力（节点、边、条件边、状态、循环），
并把这些能力封装成可供前端拖拽可视化的 API。

---

## ✨ 功能特性

### 1. LangGraph 式图编排（`StateGraph` / `CompiledGraph`）
- **节点（Node）**：可包裹普通函数（同步/异步）、AgentScope agent、或子图。
- **静态边** `add_edge(a, b)`：顺序连接，支持一对多**并行扇出**。
- **条件边** `add_conditional_edges(src, fn, path_map)`：根据状态动态路由（分支/循环）。
- **`START` / `END`** 哨兵节点，`set_entry_point` / `set_finish_point` 语法糖。
- **共享状态 + 归并器（reducer）**：默认覆盖，`append_reducer`/`add_messages` 可累积消息（多 agent 汇聚）。
- **Pregel 风格超步执行**：天然支持顺序、分支、并行、循环，并带**递归步数上限**防止死循环。
- **流式执行** `astream()`：逐节点产出 `node_start`/`node_end`/`final` 事件，便于前端实时展示。

### 2. 面向前端的编排接口（`Orchestrator`）
- `create_agent(...)` —— 创建 agent，返回 id
- `add_sub_agent(parent, ...)` —— 给某个 agent 添加子 agent（可多次，添加多个）
- `connect(a, b)` / `connect_conditional(...)` —— 连接两个 agent（含条件边）
- `set_entry(id)` —— 指定入口
- `to_dict()` / `from_dict()` —— 结构序列化，供前端保存/加载/渲染
- `build_graph()` —— 一键编译为可执行图

### 3. REST 服务（FastAPI）
把上述能力暴露为 HTTP 接口，前端可直接调用完成可视化编排。

---

## 📦 安装

> AgentScope 2.x 需要 **Python >= 3.10**。推荐用 conda 环境（示例中为 `long_agent`, Python 3.12）。

```bash
conda activate long_agent

# 安装依赖：
pip install -r requirements.txt          # 一次装齐
# 或分组安装：
pip install -e ".[agentscope]"   # 接入真实 AgentScope agent（OpenAI）
pip install -e ".[server]"       # 启动 REST 服务
pip install -e ".[test]"         # 运行测试
```

### 配置 OpenAI（.env）

本项目使用 **OpenAI**（不使用 DashScope），模型配置从项目根目录的 `.env` 读取：

```bash
cp .env.example .env    # 然后编辑 .env 填入你的 key
```

`.env` 字段：

| 变量 | 说明 | 默认 |
|------|------|------|
| `OPENAI_API_KEY` | API Key（必填） | - |
| `OPENAI_MODEL` | 模型名 | `gpt-4o-mini` |
| `OPENAI_BASE_URL` | 兼容端点（自建/中转/vLLM，可选） | 官方 |
| `OPENAI_ORG` | 组织 ID（可选） | - |

代码里通过 `engine.config` 使用：

```python
from engine.config import build_openai_agent
agent = build_openai_agent("writer", "你是一名作家")  # 自动读取 .env
```

---

## 🚀 快速开始

### A. 纯图编排

```python
import asyncio
from engine import StateGraph, END, add_messages

graph = StateGraph(schema={"messages": add_messages})
graph.add_node("planner", lambda s: {"messages": ["plan"], "count": 0})
graph.add_node("worker",  lambda s: {"messages": ["work"], "count": s.get("count",0)+1})
graph.set_entry_point("planner")
graph.add_edge("planner", "worker")
# 条件边：循环 3 次后结束
graph.add_conditional_edges(
    "worker",
    lambda s: "loop" if s["count"] < 3 else "done",
    path_map={"loop": "worker", "done": END},
)

state = await graph.compile().ainvoke({"input": "start"})
```

### B. 前端友好的 Orchestrator

```python
from engine import Orchestrator

orch = Orchestrator()
boss = orch.create_agent("coordinator", model="gpt-4o-mini")
r = orch.add_sub_agent(boss, name="researcher")   # 添加子 agent
w = orch.add_sub_agent(boss, name="writer")        # 再添加一个子 agent
rev = orch.create_agent("reviewer")
orch.connect(r, rev)                               # 连接两个 agent
orch.connect(w, rev)
orch.set_entry(boss)

graph = orch.build_graph()                         # 编译执行
state = await graph.ainvoke({"input": "写一篇报告"})
structure = orch.to_dict()                          # 导出给前端渲染
```

### C. 接入真实 AgentScope agent

```python
from engine import StateGraph, add_messages
from engine.adapters import agent_node

graph = StateGraph(schema={"messages": add_messages})
graph.add_node_object(agent_node("writer", writer_agent))
graph.add_node_object(agent_node("critic", critic_agent))
graph.set_entry_point("writer")
graph.add_edge("writer", "critic")
graph.set_finish_point("critic")
```

---

## 🌐 REST 服务

```bash
python -m engine.server
# 或： uvicorn engine.server.app:app --reload
# 文档： http://127.0.0.1:8000/docs
```

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/agents` | 创建 agent |
| GET | `/api/agents` | 列出 agent |
| DELETE | `/api/agents/{id}` | 删除 agent |
| POST | `/api/agents/{id}/sub-agents` | 添加子 agent（可多次） |
| POST | `/api/connections` | 连接两个 agent（支持条件边） |
| DELETE | `/api/connections` | 断开连线 |
| POST | `/api/graph/entry` | 设置入口 |
| GET | `/api/graph` | 导出图结构（节点+边）供可视化 |
| POST | `/api/run` | 执行编排并返回最终状态 |
| GET | `/api/export` / POST `/api/import` | 导出 / 导入编排 JSON |

---

## 🗂 目录结构

```
engine/
├── constants.py     # START / END 哨兵
├── state.py         # 共享状态 + reducer
├── node.py          # 节点抽象
├── graph.py         # 核心图引擎（StateGraph / CompiledGraph）
├── adapters.py      # AgentScope agent -> 图节点
├── orchestrator.py  # 前端友好的高层编排 API
└── server/          # FastAPI REST 服务
examples/            # 可运行示例
tests/               # pytest 单元测试
```

---

## 🧪 测试与示例

```bash
PYTHONPATH=. python examples/multi_agent_workflow.py  # 多 Agent 混合工作流（真实 OpenAI 调用）
PYTHONPATH=. python -m pytest tests/ -q              # 单元测试
```

---

## 🧩 与 LangGraph 的概念对照

| LangGraph | 本项目 |
|-----------|--------|
| `StateGraph(State)` | `StateGraph(schema)` |
| `add_node` | `add_node` / `add_node_object` |
| `add_edge` | `add_edge` |
| `add_conditional_edges` | `add_conditional_edges` |
| `START` / `END` | `START` / `END` |
| `compile()` / `invoke` / `stream` | `compile()` / `ainvoke` / `astream` |
| Annotated reducer | `schema` + `reducer`（`add_messages`） |
