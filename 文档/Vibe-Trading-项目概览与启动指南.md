# Vibe-Trading 项目概览与启动指南

> 基于仓库 v0.1.15（`pyproject.toml`）实测整理

---

## 一、这个项目是做什么的

**Vibe-Trading**（HKUDS/Vibe-Trading，v0.1.15）：一个**自然语言驱动的金融研究 + 回测 AI Agent**。

你用大白话下指令（例如"回测 BTC-USDT 20/50 均线策略 2024 年的表现"），它会自己：解析标的 → 拉行情（27 个数据源自动降级）→ 写策略代码 → 在沙箱里跑回测 → 出指标/报告 → 可选通过券商连接器做只读/实盘操作。

核心能力块：

- **回测引擎**：A股 / 港股 / 美股 / 英股 / 加股 / 韩股 / 印度 / 越南 / 加密 / 期货 / 外汇，含期权、永续合约（USD-M 保证金与强平）、因子库（Alpha Zoo 460+ 因子）
- **Agent 内核**：`agent/src/agent/loop.py`（LangGraph 驱动）、Tools（82 个工具文件）、Skills（389 个 SKILL.md + 代码）、Swarm 多智能体（30 个 preset）、Grounding 闸门（防模型编造价格）
- **金融数学**：`src/quantlib`（25 个模块，期权 / 债券 / 信用 / 计量经济 / VaR / 业绩归因等），估值引擎 DCF / Comps / 三表联动
- **实盘与治理**：14 家券商连接器、授权 mandate、kill-switch、哈希链式审计账本
- **接入面**：CLI / TUI、Web UI、REST API、MCP Server（74 个工具）、16 个 IM 通道（飞书 / 钉钉 / Telegram / Slack 等）

---

## 二、技术栈（按层）

| 层 | 技术 | 位置 |
|---|---|---|
| 后端 | **Python 3.11+**，FastAPI + Uvicorn，LangChain 1.x / LangGraph，Pydantic | `agent/`（`src/`、`backtest/`、`cli/`、`api_server.py`、`mcp_server.py`） |
| 数据 | pandas / numpy / duckdb，akshare、tushare、yfinance、ccxt、baostock 等 | `agent/src/market_data.py`、`agent/backtest/loaders/`（41 个 loader） |
| 前端 | **React 19 + TypeScript + Vite 8 + Tailwind + zustand + ECharts**，Vitest | `frontend/` |
| 桌面 | Electron 43（社区非官方构建，含 Windows NSIS 打包） | `desktop/electron/` |
| 文档站 | 静态站点 + Cloudflare Wrangler | `wiki/` |
| 部署 | Docker 多阶段构建（前端 → Python venv → 运行时），Compose 带安全加固 | `Dockerfile`、`docker-compose.yml` |
| 测试 | pytest（`agent/tests`，686 个文件），coverage / ruff / black | `pyproject.toml` |

关键端口：**后端 8899**，**前端 dev 5899**（Vite 把 `/api` 等路径代理到 8899）。

### 目录速览

```
Vibe-Trading-main/
├── agent/                 # Python 后端（主代码）
│   ├── src/               # 核心：agent、tools、skills、trading、live、factors、quantlib、swarm...
│   ├── backtest/          # 回测：engines(15) / loaders(41) / optimizers / metrics / runner
│   ├── cli/               # 命令行与 TUI（commands、components、ui）
│   ├── tests/             # pytest 测试套件
│   ├── api_server.py      # FastAPI 入口
│   ├── mcp_server.py      # MCP Server 入口
│   └── .env.example       # 环境变量模板
├── frontend/              # React 19 + Vite 前端（dev 端口 5899）
├── desktop/electron/      # Electron 桌面外壳（Windows NSIS 打包）
├── wiki/                  # 文档站（静态 + wrangler.toml）
├── tools/ scripts/        # CI 门禁脚本
├── Dockerfile             # 三阶段构建
└── docker-compose.yml     # 后端 8899 / 可选 frontend(5899) profile
```

---

## 三、怎么启动

### 前置条件

- 任意一个受支持 provider 的 **LLM API key**，或用 **Ollama** 本地运行（免 key）
- Python **3.11+**（本地路径）
- Node **>= 22.22**（前端开发）
- Docker（Docker 路径）

> 支持的 providers：OpenRouter、Requesty、OpenAI、Anthropic（原生 Messages API）、DeepSeek、Gemini、Groq、DashScope/Qwen、Zhipu、Moonshot/Kimi、MiniMax、SiliconFlow、Xiaomi MIMO、Novita AI、iFlytek 星火、Z.ai、NVIDIA NIM、ModelScope、GitHub Copilot、Ollama。
>
> 由于自动 fallback，**所有市场在没有任何 API key 的情况下也能跑**：yfinance/Yahoo（港美股、加拿大、英国）、OKX（加密）、mootdx（A股，TCP 直连）、AKShare（A股/美股/港股/期货/外汇）都是免费的。

---

### 方式 A：Docker（最快，零本地配置）

```powershell
Copy-Item agent\.env.example agent\.env   # 编辑：取消注释一个 provider，填 API key
docker compose up --build
```

打开 `http://localhost:8899`（容器内已内置打包好的前端）。

Docker 默认把后端发布在 `127.0.0.1:8899`，以非 root 用户运行。若要暴露到本机之外，需设置强 `API_AUTH_KEY`，客户端发 `Authorization: Bearer <key>`。

---

### 方式 B：本地源码启动（推荐开发用）

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1              # 被拦截时：Set-ExecutionPolicy -Scope Process -ExecutionPolicy RemoteSigned
pip install -e .
Copy-Item agent\.env.example agent\.env   # 配置 LANGCHAIN_PROVIDER / <PROVIDER>_API_KEY / LANGCHAIN_MODEL_NAME
vibe-trading                              # 交互式 CLI / TUI
```

Linux / macOS 激活：`source .venv/bin/activate`

**Web UI（两个终端）**

```powershell
vibe-trading serve --port 8899            # 终端1：FastAPI
cd frontend; npm install; npm run dev     # 终端2：Vite，访问 http://localhost:5899
```

**生产模式（单进程，FastAPI 直接托管前端静态文件）**

```powershell
cd frontend; npm run build; cd ..
vibe-trading serve --port 8899            # 访问 http://localhost:8899
```

**三条命令**

| 命令 | 用途 |
|---|---|
| `vibe-trading` | 交互式 CLI / TUI |
| `vibe-trading serve` | 启动 FastAPI Web Server（默认 8899） |
| `vibe-trading-mcp` | 启动 MCP Server（stdio，给 Claude Desktop / Cursor 等） |

> `vibe-trading serve` 绑定 `0.0.0.0`，但**默认只信任 loopback**；从另一台机器 / 虚拟机宿主机 / 局域网手机访问时，敏感接口会返回 403。需在 `agent/.env` 设 `API_AUTH_KEY`，重启，并在 Settings 中填入同一个 key。

---

### 方式 C：只跑一次研究任务

```powershell
vibe-trading init
vibe-trading run -p "Backtest a BTC-USDT 20/50 moving-average strategy for 2024 and summarize return and drawdown"
```

---

### 方式 D：MCP 插件 / ClawHub

```bash
vibe-trading-mcp                              # stdio MCP Server
npx clawhub@latest install vibe-trading --force # 一条命令，无需 clone
```

---

### 方式 E：桌面版（Windows，Electron）

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -e .
cd frontend; npm ci; npm run build; cd ..\desktop\electron
npm ci
npm run build
npm run prepare:electron
npx electron .
```

打包：`npm run pack:win`（需先 `npm run runtime:win` 组装嵌入式 Python 运行时）。
桌面外壳会自动拉起 `vibe-trading serve`（随机 loopback 端口 + 每进程独立密钥 + 父进程看门狗）。

---

### 常用环境变量（`agent/.env`）

| 变量 | 必需 | 说明 |
|---|:---:|---|
| `LANGCHAIN_PROVIDER` | Yes | provider 名称（`openrouter`、`deepseek`、`groq`、`ollama` 等） |
| `<PROVIDER>_API_KEY` | Yes* | API key（`OPENROUTER_API_KEY`、`DEEPSEEK_API_KEY` 等） |
| `<PROVIDER>_BASE_URL` | Yes | API endpoint URL |
| `LANGCHAIN_MODEL_NAME` | Yes | 模型名称（如 `deepseek-v4-pro`） |
| `TUSHARE_TOKEN` | No | A股 Tushare Pro token（会 fallback 到 AKShare） |
| `TIMEOUT_SECONDS` | No | LLM 调用超时，默认 120s |
| `API_AUTH_KEY` | 网络部署推荐 | 非本地客户端访问时要求的 Bearer token |
| `VIBE_TRADING_ENABLE_SHELL_TOOLS` | No | 远程 API / MCP-SSE 部署中显式启用 shell 类工具 |
| `VIBE_TRADING_ALLOWED_FILE_ROOTS` | No | 文档和券商日志导入额外允许的根目录（逗号分隔） |
| `VIBE_TRADING_ALLOWED_RUN_ROOTS` | No | 生成代码 run 目录额外允许的根目录（逗号分隔） |
| `VIBE_TW_STOCK_DB` | No | 台湾市场 SQLite 快照路径 |
| `VIBE_TRADING_EXTRA_CORS_ORIGINS` | No | 在回环 CORS 默认值之上追加的源（逗号分隔） |
| `CONTENT_FILTER_WARNING_THRESHOLD` | No | 内容过滤告警比例阈值（默认 0.05） |

<sub>* Ollama 不需要 API key；OpenAI Codex 走 ChatGPT OAuth（`vibe-trading provider login openai-codex`），不写入 `.env`。</sub>

---

## 附：启动前检查项

- **无 `agent/.env`**：需从 `agent/.env.example` 复制并配置一个 provider（或本地 Ollama，免 key）。

修复顺序建议：建 venv / `pip install -e .` → 配 `.env` → `vibe-trading serve --port 8899` + `npm run dev`。

---
---

# 四、对外提供接口：小程序 / H5 两套方案

## 4.0 总览：能不能做成接口给别人调？

**能。** 后端本来就是 FastAPI REST + SSE，自带的 React 前端就是调这一套 API，所以"小程序"和"H5"只是换一个客户端。

但有三条硬约束来自源码，决定了架构怎么搭：

| 约束 | 源码位置 | 后果 |
|---|---|---|
| **非本地访问必须有 `API_AUTH_KEY`** | `src/api/security.py:492` `_validate_api_auth` | 不设 key，远程一律 `403 API_AUTH_KEY is required for non-local API access`；设了 key，**连 loopback 也必须带** `Authorization: Bearer <key>` |
| **浏览器跨站写请求被拒** | `src/api/security.py:423` `_reject_cross_site_browser_request` | `Sec-Fetch-Site: cross-site` 或 `Origin` 与请求 host 不一致 → `403 Cross-site request denied`（仅对 POST/PUT/DELETE/PATCH 生效，GET/HEAD/OPTIONS 豁免） |
| **密钥是"角色"不是"身份"** | `src/api/security.py:454` `SHARED_KEY_SUBJECT` | 所有持 key 者共用一个 `Principal(attributable=False)`，**没有多租户/用户隔离**，注释明确写了不能当身份用 |

另外两个通用事实：

- **`POST /sessions/{id}/messages` 是异步的**：创建 Attempt 后立即返回 `{message_id, attempt_id}`（`src/session/service.py:304`），agent 在后台跑，结果写进消息表。所以"提交 + 轮询"是原生设计支持的用法。
- **同一 session 同时只能跑一个 attempt**，重复提问返回 **409**（`SessionBusyError`）。

### 核心接口清单

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/sessions` | 建会话，body `{"title":"","config":null}` → `session_id` |
| POST | `/sessions/{id}/messages` | 提问，body `{"content":"..."}`（1~5000 字）→ 立即返回 `{message_id, attempt_id}` |
| GET | `/sessions/{id}/messages?limit=100` | 拉消息列表，**轮询这里拿最终回答** |
| GET | `/sessions/{id}/events` | SSE 实时事件流（需 ticket 或 Bearer） |
| POST | `/sessions/{id}/cancel` | 取消正在跑的 run |
| POST | `/auth/sse-ticket` | 用 Bearer 换**一次性、60 秒**的 SSE ticket |
| GET | `/runs`、`/runs/{run_id}`、`/runs/{id}/code` | 回测结果、代码 |
| GET | `/health`、`/live` | 健康检查（无需鉴权） |

---

## 4.1 方案 A：微信小程序

### 推荐架构

```
小程序  ──wx.request──▶  你的 BFF 中间层  ──Bearer key──▶  Vibe-Trading (8899)
(只带自己的登录态)        (Node/PHP/Python)
                          · 注入 Authorization 头
                          · 按 openid 隔离 session
                          · 限流 / 审计 / 超时兜底
```

**不要直连。** 理由：

1. **`API_AUTH_KEY` 不能放进小程序**：包体可被反编译，key 泄露 = agent、回测沙箱、券商只读接口全部暴露。
2. **无用户隔离**：共享 key 下所有用户看到同一批 session，必须在 BFF 按 `openid` 映射 `session_id`。
3. **微信侧硬门槛**：生产环境必须 `https` + **已 ICP 备案域名** + 小程序后台配置 `request 合法域名`；`http://127.0.0.1` 只能在开发者工具勾选"不校验合法域名"时跑。
4. **合规红线（比技术更致命）**：金融/证券投资咨询类小程序类目需要相应资质，个人主体基本无法过审。先确认资质再动手。

### 时序（轮询方案，推荐）

```
小程序                BFF                    Vibe-Trading
  │  1 登录拿 code      │                          │
  ├───────────────────▶│                          │
  │  2 POST /chat       │  POST /sessions         │
  │  {question}         ├────────────────────────▶│  session_id
  │                     │  POST /sessions/{id}/messages
  │                     ├────────────────────────▶│  {message_id, attempt_id}  立即返回
  │  3 {attempt_id}     │                          │  ⏳ 后台跑 agent
  │◀───────────────────┤                          │
  │  4 每 2s 轮询结果    │  GET /sessions/{id}/messages
  │───────────────────▶├────────────────────────▶│
  │                     │◀────────────────────────┤  [... messages]
  │  5 拿到 assistant 回复（linked_attempt_id == attempt_id）
  │◀───────────────────┤                          │
```

### 小程序端代码

```js
// utils/chat.js
const BASE = 'https://your-bff.example.com'   // 你的中间层，绝不直连后端

export async function ask(question, sessionId) {
  // 1. 提问（秒回，不等待 agent）
  const { attempt_id } = await req('POST', `/sessions/${sessionId}/messages`, { content: question })

  // 2. 轮询最终回答
  for (let i = 0; i < 150; i++) {          // 最长约 5 分钟
    await sleep(2000)
    const msgs = await req('GET', `/sessions/${sessionId}/messages?limit=20`)
    const hit = [...msgs].reverse().find(
      m => m.role === 'assistant' && m.linked_attempt_id === attempt_id
    )
    if (hit) return hit.content
  }
  throw new Error('timeout')
}
```

### 接口封装建议（BFF 只暴露 2~3 个）

| BFF 接口 | 内部转发 |
|---|---|
| `POST /chat` | 建 session（首次）+ `POST /sessions/{id}/messages` |
| `GET /chat/result?attempt_id=` | `GET /sessions/{id}/messages` |
| `POST /chat/cancel` | `POST /sessions/{id}/cancel` |

### 小程序特有的坑

- **SSE 基本用不了**：小程序没有 `EventSource`；`wx.request` 的 `enableChunked` 分片接收能力和平台支持都有限。**直接用轮询**，别在这上面耗。
- **超时**：`wx.request` 默认 60s 超时，可在 `app.json` 里调 `networkTimeout`。因为提接口是异步返回的，正常情况不会撞上；但 BFF 自身要设更长的上游超时并做兜底。
- **不要调 `/mandate`、`/live`**：下单与实盘接口绝不对小程序开放。
- **shell 工具默认关闭**：远程部署时 `VIBE_TRADING_ENABLE_SHELL_TOOLS=false`，保持默认。
- **耗时预期**：回测类问题几十秒到数分钟；UI 必须给"思考中 / 工具调用中"的中间态，否则体验很差。

---

## 4.2 方案 B：H5 网页聊天

### 最重要的一条：**同源**

浏览器会带 `Origin` 和 `Sec-Fetch-Site`，而 `_reject_cross_site_browser_request`（`security.py:423`）对非安全方法做两道拦截：

1. `Sec-Fetch-Site: cross-site` → 直接 403；
2. `Origin` 的 host/port 与请求 host 不一致，且不是 loopback → 403。

**配 `CORS_ORIGINS` 解决不了第 2 条**（CORS 中间件只管响应头，不管这个 403 判断）。实测结论：

| H5 与 API 的关系 | 能否直连 | 说明 |
|---|---|---|
| **同源**（同一 scheme+host+port，如 Nginx 把静态资源和 `/api` 挂在一个域名下） | ✅ | `Sec-Fetch-Site: same-origin`，无跨站问题 |
| 同站不同子域（`h5.example.com` → `api.example.com`） | ❌ | `same-site` 过了第 1 关，但 Origin host ≠ 请求 host，第 2 关 403 |
| 完全跨域 | ❌ | `cross-site` 直接 403 |

> 这也是为什么项目自带的前端在开发时用 Vite 代理（`/api` → 8899）、生产时由 FastAPI 同源托管 `frontend/dist`——全程同源。

### 三种落地形态（按推荐度排序）

#### 形态 1：直接二开官方前端（最省，强烈推荐）

项目自带的 `frontend/` **本身就是一个 H5 网页聊天**（React 19 + Vite，含会话、Run Detail、回测图表、Options Lab、i18n 八种语言）。

```bash
cd frontend && npm ci && npm run build && cd ..
vibe-trading serve --port 8899     # FastAPI 同源托管 dist/
```

- 零跨域、零鉴权改造（loopback 场景不需要 key）
- 只需要按你的品牌改 UI/文案；`frontend/src/pages/` 有 24 个页面可裁剪
- 外网部署时：Nginx 反代 + 同源托管 + 由 Nginx 注入 `Authorization` 头（见下）

#### 形态 2：自建 H5 + Nginx 同源反代

```nginx
server {
    listen 443 ssl;
    server_name chat.example.com;

    # H5 静态资源
    location / {
        root /var/www/h5;
        try_files $uri /index.html;
    }

    # API 同源反代：浏览器看到的还是 chat.example.com，天然同源
    location /api/ { ... }                 # 如需要可加前缀重写
    location /sessions/ {
        proxy_pass http://127.0.0.1:8899;
        proxy_set_header Host $host;
        proxy_set_header Authorization "Bearer ${VIBE_API_KEY}";   # 关键：服务端注入
        proxy_http_version 1.1;
        proxy_read_timeout 600s;           # agent 可能跑几分钟
    }
    location /auth/ {
        proxy_pass http://127.0.0.1:8899;
        proxy_set_header Authorization "Bearer ${VIBE_API_KEY}";
    }
    location /runs/ {
        proxy_pass http://127.0.0.1:8899;
        proxy_set_header Authorization "Bearer ${VIBE_API_KEY}";
    }

    # SSE 必须关缓冲
    location /events/ {
        proxy_pass http://127.0.0.1:8899;
        proxy_set_header Authorization "Bearer ${VIBE_API_KEY}";
        proxy_buffering off;
        proxy_cache off;
        chunked_transfer_encoding on;
        proxy_read_timeout 3600s;
    }
}
```

**关键点：API key 由 Nginx 注入，浏览器永远不接触 key。** 这样 H5 端代码里不用写任何密钥。

#### 形态 3：完全跨域独立部署

不推荐：POST 会被 `Cross-site request denied` 拦。若必须这么做，只能让 H5 调你自己的 BFF（BFF 与 H5 同源，BFF 再服务端转发到 Vibe-Trading），本质上退化成"形态 2 + 一层 BFF"。

### H5 的流式输出：可以用 SSE

同源场景下浏览器原生 `EventSource` 可用，但 `EventSource` **不能自定义请求头**。两条路：

**路 A（推荐）：反代层注入 Authorization**
`require_event_stream_auth`（`security.py:591`）接受 Bearer 头，反代注入后前端直接：

```js
const es = new EventSource(`/sessions/${sid}/events?replay=active`)
es.addEventListener('message', e => render(JSON.parse(e.data)))
es.addEventListener('attempt.completed', () => es.close())
```

**路 B：用官方 ticket 流程**
1. `POST /auth/sse-ticket`（带 Bearer，可由反代注入）→ `{ticket}`
2. `new EventSource('/sessions/{sid}/events?ticket=xxx')`
3. ticket **60 秒有效、一次性**，断线重连必须重新申请

### H5 前端示例（不用 SSE 也能跑）

```js
const API = ''   // 同源，相对路径

async function ask(question, sessionId) {
  const r = await fetch(`${API}/sessions/${sessionId}/messages`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },   // 无需 Authorization，反代已注入
    body: JSON.stringify({ content: question }),
  })
  if (r.status === 409) throw new Error('上一条还在跑，请稍候')
  const { attempt_id } = await r.json()

  return new Promise((resolve, reject) => {
    const t = setInterval(async () => {
      const msgs = await (await fetch(`${API}/sessions/${sessionId}/messages?limit=20`)).json()
      const hit = [...msgs].reverse().find(m => m.role === 'assistant' && m.linked_attempt_id === attempt_id)
      if (hit) { clearInterval(t); resolve(hit.content) }
    }, 2000)
    setTimeout(() => { clearInterval(t); reject(new Error('timeout')) }, 300000)
  })
}
```

### H5 特有的坑

- **CSP**：服务端默认下发严格 CSP（`default-src 'self'`）。若你的 H5 引入外部脚本/字体/图表 CDN，会被拦。要么全部自托管，要么设 `VIBE_TRADING_CSP_REPORT_ONLY=1` 降级为 Report-Only。
- **移动端适配**：官方前端是桌面优先的（多栏布局、ECharts 图表密集）；做手机 H5 需要自己写移动版 UI 或裁剪页面。
- **SSE 与 Nginx 缓冲**：忘记 `proxy_buffering off` 会导致前端一直收不到事件，看起来像"卡住"。
- **长连接被中间层切断**：企业网关/云 WAF 常有 60s 空闲超时，SSE 需要服务端心跳或客户端定时重连。
- **外网必须 https**：否则浏览器混合内容 + 微信/分享场景都会出问题。

---

## 4.3 小程序 vs H5 对比

| 维度 | 小程序 | H5 |
|---|---|---|
| 流式（SSE） | ❌ 基本不可用，**用轮询** | ✅ 原生 EventSource，体验更好 |
| 跨域/跨站限制 | 不适用（非浏览器） | ⚠️ **必须同源**，否则 POST 403 |
| 密钥存放 | ❌ 绝不能放包里 | ⚠️ 绝不能放 JS 里，由反代注入 |
| 中间层必要性 | **必需**（BFF 做隔离+鉴权） | 同源反代即可（Nginx 够用） |
| 域名/备案 | 必须 https + 备案 + 后台白名单 | 只需 https（建议备案域名） |
| 合规资质 | ⚠️ 金融类目资质要求高 | 相对宽松（仍属金融信息服务） |
| UI 成本 | 从零写 | 可直接二开官方 `frontend/` |
| 开发调试 | 开发者工具"不校验合法域名" | 浏览器直连 `localhost:5899` |
| 推荐度 | 中（合规是最大不确定性） | **高（尤其形态 1/2）** |

**一句话建议**：先做 H5（同源反代 + 二开官方前端），跑通后再考虑小程序——小程序 90% 的工作量（BFF、鉴权、轮询、超时、合规）是 H5 的父集，反过来不成立。

---

## 4.4 两套方案共同的注意事项

1. **`API_AUTH_KEY` 必须设**（只要不是纯本机 loopback 访问），且通过 `Authorization: Bearer` 头传递；**不要用 `?api_key=`**（只有 SSE ticket 端点接受 query，长 key 进 URL 会泄进日志和 Referer）。
2. **轮询间隔 2s**，并给"思考中/工具调用中"中间态；一次回测可能几十秒到数分钟。
3. **一个用户一个 session**：同一 session 并发提问会 409。
4. **提供取消入口**：`POST /sessions/{id}/cancel`，否则用户只能干等。
5. **不要暴露** `/mandate`、`/live`（下单/实盘）、`/system/shutdown`、设置写接口。
6. **限流与配额**：LLM + 数据源都有成本，BFF 层必须做（用户级、会话级）。
7. **日志审计**：BFF 记录 openid/user → session_id → attempt_id，出问题可追溯。
8. **结果渲染**：回答是 Markdown（含表格、公式 KaTeX、代码块），小程序需 `towxml` 之类渲染库；H5 可用 `react-markdown`（官方前端已带 `remark-gfm` + `rehype-katex` + `highlight.js`）。
