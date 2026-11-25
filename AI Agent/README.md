# AI Agent 工程助手

面向工程与设计场景的本地 AI Agent：在浏览器里和模型多轮对话，按需调用技能、MCP 工具与本地知识库，也可以把常用操作封装成插件或脚本一键执行。

后端使用 **FastAPI + LangChain**，会话与配置存储在 **SQLite**；前端是零构建的原生 JS 单页应用，回答通过 **SSE** 流式返回。

## 主要能力

- **流式对话**：SSE 推送文本增量、工具调用、审批和用量事件。
- **多会话管理**：会话、分组、工作空间；历史落库，刷新不丢。
- **Skills 技能包**：`skills/<id>/SKILL.md` 描述能力，加载后作为 Agent 工具。
- **MCP 工具**：`mcp.json` 声明 stdio MCP 服务，启动时拉起，按风险等级审批。
- **本地知识库**：导入 PDF / Office 文档 → 解析 → 切片 → BM25 + 向量检索，输入框用 `@` 引用。
- **插件与脚本**：带参数表单的插件、直接执行的脚本；子进程隔离运行，清洗敏感环境变量并设置超时。
- **工具审批与运行追踪**：`ask / auto / full` 三档权限，追踪面板可查看每一步工具调用。
- **上下文管理**：接近上下文窗口上限时自动压缩历史，保留最近消息。
- **模型可切换**：默认 DeepSeek（OpenAI 兼容接口），支持自定义模型与连通性探测。

## 技术栈

| 层 | 选型 |
| --- | --- |
| 后端 | FastAPI、Uvicorn |
| Agent | LangChain 1.x、LangGraph、langchain-openai |
| 存储 | SQLite、SQLAlchemy 2（异步）、Alembic |
| 配置 | pydantic-settings、`.env` |
| 前端 | 原生 HTML / CSS / JS，无构建步骤，SSE |
| 模型 | DeepSeek 或任意 OpenAI 兼容服务 |
| 可选 | MinerU（PDF 解析）、pywin32（CAD 联动） |

## 架构

```text
浏览器（web/）
   │ HTTP / SSE
   ▼
FastAPI（app/）
   ├─ /api/chat         对话入口，SSE 事件流
   ├─ /api/sessions     会话 / 分组 / 上下文
   ├─ /api/workspaces   工作空间与任务产物
   ├─ /api/skills       Skills 列表与开关
   ├─ /api/mcp          MCP 服务管理
   ├─ /api/knowledge    知识库、文档、检索
   ├─ /api/automation   插件与脚本
   └─ /api/trace        运行追踪
          │
          ▼
   LangChain Agent（create_agent）
   ├─ 工具：Skills / MCP / 知识库 / CAD / 审批
   └─ 上下文：历史消息 + 摘要压缩
          │
          ▼
   SQLite（data/agent.db）+ 文件（data/、workspace/）
```

## 目录结构

```text
agent-app/
├─ app/                 后端：FastAPI 路由、Agent、数据访问
├─ web/                 前端：index.html / app.js / style.css
├─ data/                SQLite 数据库、知识库切片、工作空间索引
├─ workspace/           默认工作空间（Agent 读写文件的位置）
├─ skills/              技能包，每个子目录一个 SKILL.md
├─ mcp.json             MCP 服务清单
├─ run.py               一键启动
└─ requirements.txt
```

## 快速开始

```bash
python -m venv .venv
.venv\Scripts\activate          # Windows
pip install -r requirements.txt

copy .env.example .env          # 填入 DEEPSEEK_API_KEY
python run.py
```

打开 <http://127.0.0.1:8000>。

`.env` 主要配置：

| 变量 | 说明 |
| --- | --- |
| `DEEPSEEK_API_KEY` | 模型 API Key |
| `DEEPSEEK_BASE_URL` | OpenAI 兼容地址，默认 `https://api.deepseek.com` |
| `DEFAULT_MODEL` | 默认模型 |
| `HOST` / `PORT` | 监听地址，默认 `127.0.0.1:8000` |

## SSE 事件约定

`POST /api/chat` 返回 `text/event-stream`，每块格式为 `event: <name>\ndata: <json>\n\n`。

| 事件 | 数据 | 含义 |
| --- | --- | --- |
| `session` | `{session_id}` | 新会话 ID |
| `delta` | `{text}` | 回答文本增量 |
| `tool_call` | `{id, name, arguments, risk}` | 开始调用工具 |
| `approval_required` | `{approval_id, ...}` | 高风险工具等待确认 |
| `tool_result` | `{id, status, output}` | 工具执行结果 |
| `notice` / `context` / `usage` | — | 提示、压缩信息、用量 |
| `done` | `{session_id}` | 本轮结束 |
| `error` | `{message}` | 出错信息 |

## 扩展点

- **加技能**：新建 `skills/<name>/SKILL.md`，写清触发场景和步骤。
- **加 MCP 服务**：在 `mcp.json` 的 `servers` 中增加一条 stdio 服务配置。
- **加知识库**：通过页面导入文档，系统自动解析、切片并建立索引。
- **加插件 / 脚本**：在「插件 & 脚本」页面新建；代码在独立子进程中执行，默认工作目录为工作空间。
