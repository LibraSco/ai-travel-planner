# 智能旅行助手 · LangChain 多 Agent + 高德 MCP

**输入城市、日期和偏好，输出一份包含景点、天气、酒店、餐饮和预算的可编辑行程。**

基于 HelloAgents 教程骨架，独立完成 LangChain 多 Agent 重构。

> **本仓库是展示仓库**：只包含 README、截图与指标，不含源码（参赛作品，暂未开源）。
> 想了解实现细节，可以现场演示，或按需提供只读访问。
---

## 演示截图

<img width="1902" height="1032" alt="屏幕截图 2026-10-21 202941" src="https://github.com/user-attachments/assets/9f124f4d-36a8-4ec8-a99b-268c98d5ffec" />

<img width="1807" height="1032" alt="屏幕截图 2026-10-21 203230" src="https://github.com/user-attachments/assets/0bb40d3a-5393-4e67-9ad9-6437ee79ad1f" />

<img width="1905" height="1037" alt="屏幕截图 2026-10-21 203051" src="https://github.com/user-attachments/assets/960a97de-d185-4eb6-bf69-5a63e2a4dc2c" />

<img width="1885" height="1040" alt="屏幕截图 2026-10-21 203800" src="https://github.com/user-attachments/assets/1e3afaa9-19a6-43a7-a309-ee36dbda906f" />

<img width="1913" height="1032" alt="屏幕截图 2026-10-21 203036" src="https://github.com/user-attachments/assets/cf26bbc2-d47e-4a1a-adda-9af965d351cb" />

---

## 架构
```text
- Vue3 前端 → `POST /api/trip/plan`
- FastAPI → 多 Agent 编排
  - 景点 Agent：maps_text_search + maps_search_detail
  - 天气 Agent：maps_weather
  - 酒店 Agent：maps_text_search + maps_search_detail
- 三个查询 Agent → 规划 Agent（不持工具）→ TripPlan JSON
```
---

## 技术栈

| 层 | 选型 |
| --- | --- |
| 接口 | FastAPI、Pydantic v2、uvicorn |
| 编排 | LangChain `create_agent`、`ChatOpenAI`（OpenAI 兼容接口） |
| 工具 | `langchain-mcp-adapters` + `MultiServerMCPClient`（stdio）、amap-mcp-server 0.1.11 |
| 外部服务 | 高德地图 MCP（16 个工具） |
| 前端 | Vue3 + TypeScript（教程版基础上用 AI 编码工具改造） |
| 运行环境 | Python 3.9+、uv（`uvx` 拉起 MCP 子进程） |

> 兼容性备注：amap-mcp-server 0.1.11 依赖的 mcp 1.x 会从 `pydantic._internal._typing_extra` 导入 `eval_type_backport`，该符号在 pydantic 2.14 已移除，因此用 `uvx --with "pydantic<2.14" amap-mcp-server` 固定依赖，否则子进程启动即崩。

---

## 我负责的部分

**基于教程骨架重构，不是从零开发。** 具体边界如下：

- **重写**：LLM 接入层（`HelloAgentsLLM` → `ChatOpenAI`）、高德 MCP 工具层、多 Agent 编排层、三个路由文件、配置与依赖清单。
- **保留**：FastAPI 路由路径、请求/响应模型，接口契约不变。
- **修掉的工程问题**：POI 坐标缺失、MCP 返回的 text 块解包、异步加载与并发首请求、pydantic 版本兼容。
- **前端改造**：在教程版基础上用 AI 编码工具调整页面和交互，负责需求拆解、接口契约与联调验收。
  
---

## 三个关键难点

### 1. POI 搜索不返回坐标

高德 `maps_text_search` 只返回 id、name、address、typecode，没有经纬度，前端地图标记全落在 (0,0)。修复是拿 POI id 再调 `maps_search_detail` 补齐。

**边界**：这个补齐在 `AmapService` 层，覆盖 `/api/map/poi` 和 `/api/poi/search`；生成行程走的是 Agent 直接调 MCP 工具，不经过这一层，所以给景点和酒店 Agent 补了 `maps_search_detail` 工具，并在提示词里要求对候选补坐标。前者是代码保证，后者是提示词约束。

### 2. 异步懒加载 + 并发首请求

MCP 工具加载是异步的，不能在构造函数里阻塞等待，所以做成首次请求时懒加载 + 单例缓存。代价是并发首请求会重复拉起子进程，于是加了一把 `asyncio.Lock` 做 single-flight：已缓存走快路径，没有就抢锁，锁内再查一次缓存。

**踩过的坑**：`asyncio.Lock` 建在模块级时，Python 3.9 会在 import 阶段就绑定当时的 loop，uvicorn 启动新 loop 后第一次 `acquire` 报 `attached to a different loop`。改成在协程内第一次调用时创建才正常。桩测试里 5 个并发调用只触发 1 次加载；加载失败不写缓存，下一个请求会自动重试。

### 3. 16 个工具按 Agent 分组

一开始 16 个工具全给三个查询 Agent，靠提示词约束，工具多了容易误选，schema 也一直占上下文。改成白名单分组：景点和酒店各 2 个、天气 1 个、规划 Agent 不给工具，MCP 返回的工具名对不上就直接抛错并打印实际名字，不会静默变成空工具集。

---

## 联调检查

| 检查项 | 结果 |
| --- | --- |
| 配置 | `.env` 加载与必填校验通过 |
| LLM | OpenAI 兼容接口连通 |
| MCP | 16 个工具加载成功 |
| Agent | 4 个 Agent 创建，工具按组分配 |
| 路由 | `POST /api/trip/plan` 返回真实景点名 |
| 健康检查 | `GET /api/trip/health` 返回 `tools_count: 16` |

---

## 本地运行需要什么

- Python 3.9+
- [uv](https://github.com/astral-sh/uv)（用于 `uvx` 拉取 amap-mcp-server）
- Node.js（教程版前端）
- 环境变量（放在 `.env`，不要提交）：
  - `AMAP_API_KEY`：高德 Web 服务 Key，需要开通 MCP 对应接口
  - `LLM_API_KEY` / `LLM_BASE_URL` / `LLM_MODEL_ID`：任意 OpenAI 兼容服务
- 不需要 Docker

---

## 已知限制

- 三个查询 Agent 是串行的。`asyncio.gather` 没上，因为三个 Agent 共用一个 stdio MCP 会话，并发工具调用不一定真并行，且 4 次 LLM 调用才是延迟大头，要先测收益。
- 整链路失败时返回兜底行程（占位景点名、空天气），接口不 500。这是演示取舍；生产应返回 503 + 明确错误码，或把兜底标记为 `degraded`。
- 没有提交到仓库的自动化测试，只有改分组和锁时跑过的桩测试；下一步是 mock MCP + 固定样例回归。
- 前端基于教程版，用 AI 编码工具改造；代码由工具生成，我负责需求、接口和验收。
- single-flight 锁只在单进程内有效，`uvicorn --workers N` 会每个进程各加载一次。

---

## 关于开源

项目正在参赛，源码暂不公开。这个仓库只用于展示设计、指标和真实运行效果。

如果你需要评估实现，可以约时间现场演示；在比赛结束后可以再谈开源。

---

<sub>© 2026 李欣 · 保留所有权利。未经许可请勿转载截图与文档内容。</sub>

