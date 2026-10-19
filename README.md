# freebuff2api-workers

[![License: AGPL-3.0](https://img.shields.io/badge/License-AGPL--3.0-blue.svg)](LICENSE)

> 🎉 欢迎使用与交流！有任何问题或想法欢迎提 Issue / PR。
>
> 🌐 **语言 / Language**: [中文](#freebuff2api-workers) · [English](#english-documentation)
> 开源协议：**[AGPL-3.0](#-license)**

把 **freebuff/codebuff** 的免费模型暴露成 **OpenAI-compatible API**。单文件无依赖，**推荐 Docker 容器部署**（或自建 VPS 运行），适配任意 OpenAI SDK / 客户端（QwenPaw、Hermes、ChatGPT-Next-Web、LobeChat、one-api 等）。

> ⚠️ **部署方式重要提示**：Freebuff 官方已检测 Cloudflare Worker 部署（识别 `cf-worker` / `cf-ray` 等边缘标记），**在 CF 上部署会显著增加账号被封禁的风险**。因此本项目**不推荐 Cloudflare 部署**，推荐使用 **Docker 容器**或自建 VPS 运行（见下方「[🐳 Docker 容器化部署](#-docker-容器化部署-推荐)」）。

## ✨ 特性

- ⭐ **完整访问模式模型**：Cloudflare Workers 默认使用美国出口，通常可获得 Freebuff 完整访问模式；其中 DeepSeek V4 Flash 和 MiMo 2.5 属于官方特殊的非 Premium 模型
- 🛡️ **Web 管理面板 `/admin`**：账号池 / 代理 / API Key / **Freebucks 额度**全在页面上改，**保存即生效，无需重启容器**（见下方「[🛡️ 管理面板](#️-管理面板-admin)」）
- 💰 **Freebucks 额度可见**：按次扣费的**每日钱包**（不是「每个模型每天 N 次」白名单），面板可直接查余额、每日用量、价格表和**今天打得起哪几个模型**（见下方「[💰 Freebucks 额度机制](#-freebucks-额度机制)」）
- 🔁 **多账号自动切换**：撞额度自动冷却并切换；账号在面板里加，**不是**靠 `FREEBUFF_TOKEN` 环境变量
- 💡 **优先复用活跃 session**：一个 session 约 1 小时有效，创建 session 才扣额度；只要当前模型的 session 还活跃就钉在同一账号上，用满再换，最大化额度利用率
- 📢 **广告与 streak 流程兼容**：创建新 session 前，Worker 会按官方客户端流程请求广告，并调用 `GET /api/v1/freebuff/streak` 尝试签到；相关请求失败会静默跳过，不阻塞聊天
- 🧩 **OpenAI 兼容**：`/v1/models`、`/v1/chat/completions`、`/v1/responses`（流式/非流式视接口支持情况而定）
- 📨 **Anthropic Messages API**：支持 `/v1/messages`、`/messages` 及对应的 `count_tokens` 路由，可供 Anthropic SDK / 兼容客户端尝试接入
- ❤️ **健康检查**：`GET /healthz`（免鉴权），方便监控探活
- 📦 **单文件部署**：无依赖，`worker.js` 一处代码，CF / Docker / VPS 通用

## 📨 Anthropic Messages API 支持

主代码已加入 Anthropic Messages API 适配，当前支持：

- `POST /v1/messages`
- `POST /messages`
- `POST /v1/messages/count_tokens`
- `POST /messages/count_tokens`
- Anthropic 消息格式转换为 Worker 内部使用的 OpenAI-compatible 请求
- 文本消息、`tool_use` / `tool_result`、`tool_choice`
- 非流式响应和 Anthropic SSE 流式响应
- Anthropic 风格的错误响应

> ⚠️ **测试说明**：当前项目维护者没有实际使用 Anthropic Messages API 的客户端环境，因此暂未完成真实 Anthropic 客户端的端到端测试。主代码和本地 stub / 回归测试已经处理并验证转换逻辑，但不代表所有 Anthropic SDK、工具调用组合和客户端行为都已覆盖。
>
> 如果你有 Anthropic Messages API 的实际使用场景，欢迎在不影响现有 OpenAI API 线路的前提下进行测试，并反馈请求格式、流式响应、工具调用或模型兼容性问题。反馈时请尽量附上脱敏后的请求结构、响应状态码和错误信息。
>
> Anthropic API 是新增的协议适配层，不改变现有 OpenAI `/v1/chat/completions`、`/v1/responses`、账号轮换、session 生命周期和 Freebuff 主调用链。

## 💰 Freebucks 额度机制

Freebuff 免费版的额度单位叫 **Freebucks** —— 一个**按次扣费的每日钱包**，**不是「每个模型每天 N 次」的白名单**。所以「换了个模型就报 429」通常不是被限制了，而是**这次更贵，钱包不够扣**。

> 📌 本节数据于 **2026-09-22 实测**（上游 `GET /session` 快照 + 官方源码 `freebucks-pricing.ts` / `freebuff-spend-ceilings.ts`）。官方规则会调整，**以你点「查额度」按钮看到的实时 `prices` 为准**。

### 扣费点：建 session，不是对话轮次

扣费发生在 `POST /api/v1/freebuff/session`：

- 一个 session 有效期 **1 小时**，期间复用**不再扣**（Worker 缓存剩余 >60s 就直接复用）
- 所以实际形态是「**每天 N 个 session 时段**」，不是「N 次对话」
- `DELETE session` 会**异步退款**（返回 `freebucksRefundPending: true`，余额稍后涨回）；但传**伪造的 `instanceId` 拿不到退款**（`freebucksRefund: 0`）

### 价格表（`freebucks.prices` 实测）

| 模型 | 单价 | 40/天闸下能打几次 |
|---|---|---|
| `z-ai/glm-5.3-flash` | 5 | **8** |
| `crof/kimi-k3-eco` | 5 | **8** |
| `mimo/mimo-v2.5` | 10 | **4** |
| `upstage/solar-pro4` | 10 | **4** |
| `deepseek/deepseek-v4-flash` | 15（off-peak 10） | **2**（off-peak **4**） |
| `deepseek/deepseek-v4-flash-max` | 15 | **2** |
| `deepseek/deepseek-v4-pro-max` | 20 | **2** |
| `openai/gpt-5.6-luna` | 20 | **2** |
| `mimo/mimo-v2.6-pro` | 30 | **1** |
| `google/gemini-3.8-flash` | 80 | **0 —— 单日闸内买不起一次** |

- **off-peak 时段**（`freebucks.offPeak`）部分模型降价，目前确认 `deepseek-v4-flash` 15 → **10**，对应北京时间 **06:00–14:00**
- `deepseek-v4.1-flash` / `v4.1-pro` **能建 session（实测 200 active）**，但价格表里**没有 4.1 条目**，单价未知 —— 点「查额度」看 `prices` 里有没有它
- `deepseek/deepseek-v4-pro` 实测 `409 model_unavailable`，官方已下线
- `freebucks.priceNotices` 带涨价/调价提醒，面板渲染成 `⚠ 模型: 说明`

### ⚠️ 两个 limit 并存：以快照 `daily.limit` 为准

上游同时报**两个不同的数**，别搞混：

| 来源 | 数字（会变） | 含义 |
|---|---|---|
| **429 响应体** `limit` | 40 / 25 … | 撞闸当日的闸值 —— `recentCount ≥ limit` 就 429 |
| **GET 快照** `daily.limit` | 100 / 25 … | 钱包日限额（`spent` 对着它算） |

**别拿 429 的 limit 去减 `recentCount` 推余额**，只信快照的 `daily.remaining` + `balance`。

### accessTier 与「新号只有 25 点」

额度闸是**按账号分档**的，`accessTier` 不同、日上限不同：

| accessTier | 日上限（快照 `daily.limit`） | 说明 |
|---|---|---|
| `full` | 曾实测 100（429 闸 40） | 2026-09-22 前的老号实测值，官方可随时调 |
| `limited` | **25** | **新注册账号的默认档** —— 刚注册的号只有 25 点/天，不是 bug |

实测（2026-09-23）：新号注册即 `accessTier: "limited"`、`daily.limit: 25`，glm-5.3-flash 单价 5 点/次 ≈ 一天 5 次。官方源码 `freebuff-countries.ts` 显示档位还跟**出口国**有关（US/CA/GB/AU 等 22 国 = `full`，其他国家与 VPN 出口 = `limited`），账号信任度在此之上再叠加判定。老号也可能被从 `full` 降到 `limited`。

### 重置时间

`resetAt` = **太平洋时间午夜** → **北京时间次日 15:00**（例：`2026-09-23T07:00:00Z` = 北京 2026-09-23 15:00）。

> 📌 **锚点小知识（GitHub Markdown）**：含 emoji 的标题生成锚点时，GitHub 只删 emoji 基本码位、保留变体选择符 U+FE0F。修锚链接时以渲染 HTML 的真实 id 为准。

### 官方 cap 的由来

上游源码 `freebuff-spend-ceilings.ts` 原文：

> Capacity is now limited per account — sustained automated abuse forced us to cap how much any one account can use.

`accessTier: "full"` 表示账号**没被降级**，只是撞了通用 cap。

### 扩量路径

1. **付费档** —— `$8/月` → `150 Freebucks/天`（`wallet` 里能看到）
2. **多账号轮换** —— 本项目一账号一 socks5 的架构**天然支持**，每个号一份独立额度，在面板里加号即可。**注意：新号默认只有 `limited` 档 25 点/天**（见上文「accessTier 与新号额度」），多号是乘以 25，不是乘以 100
3. **挑便宜模型** —— 优先 `glm-5.3-flash` / `kimi-k3-eco`（单价 5），贵的模型留到 off-peak

### 常见误区

| 症状 | 真相 |
|---|---|
| 换了模型就报 429 | 不是模型白名单，是**这个模型更贵**，钱包不够扣 |
| `HTTP 502: create session failed: 429 ...` | 上游额度耗尽，等 `resetAt` |
| `409 model_locked` | 会话锁冲突，**不是额度问题** —— Worker 会自动「删旧建新」重试（v1.8.11.1） |
| `429` | **不等于封禁**，只是额度；`account_suspended` 才是封号（终态，不可逆） |
| 新号怎么只有 25 点 | 新账号默认 `limited` 档、25/天，**不是 bug**（档位由出口国 + 账号信任度共同决定，见「accessTier 与新号额度」） |
| 额度掉得特别快 | 诊断性反复建 session 会连扣 —— **每次 POST 都是真扣费** |


## 🚀 快速开始

1. 获取 freebuff token（见下方「获取 FREEBUFF_TOKEN」）
2. 部署服务（见下方「部署」，**推荐 Docker 容器部署**）
3. 打开管理面板 `http://<服务器IP>:8877/admin`（初始密码 `admin`，**首登强制改密**），在**账号池**里粘贴 token，点**保存账号**
4. 用任意 OpenAI 客户端连接：
   - **Base URL**: `http://localhost:8877/v1`（Docker 部署）或 `https://你的worker名.你的子域.workers.dev/v1`（CF 部署，不推荐）
   - **API Key**: 面板「服务变量 → API Key」的值（缺省 `freebuff-default-key`）

> 🌐 **自定义域名**：如果 `*.workers.dev` 域名访问不通（部分地区被墙/受限），可给 Worker 绑定自己的域名，Base URL 改为 `https://你的域名/v1`。配置方法见下方「[自定义域名](#-自定义域名)」。

## ❤️ 健康检查

部署后可用（**无需 API key**）：

```bash
curl https://你的worker.workers.dev/healthz
# {"status":"ok","version":"1.8.11.1","accounts":1,"alive_accounts":1,"account_states":{"ok":1},"health_source":"worker_cache","time":"..."}
```

- `version` 字段=当前部署的 **worker.js 版本**，用于确认线上是否已更新（CF 边缘缓存有延迟，验证时等几秒或加随机参数）
- `status` 为 `ok` / `degraded` / `critical`，`account_states` 是各账号状态计数（`ok` / `rate_limited` / `banned` / `model_locked` …）
- ⚠️ **这里只有状态，没有额度数字** —— 查 Freebucks 请用面板的「查额度」按钮（见「[🛡️ 管理面板](#️-管理面板-admin)」）
- 适合接入 UptimeRobot / 自建监控探活

## 🔑 获取 FREEBUFF_TOKEN

freebuff 登录凭证（authToken）通过官方 CLI 同款**授权码轮询**获取。项目自带提取工具 `freebuff_tools/extract_freebuff.py`，交互方式与 `cline_oauth.py` 一致。

### 方式 A：GitHub Actions 工作流（推荐，远程提取）

仓库自带工作流 `.github/workflows/extract-token.yml`，在 GitHub Actions 里跑提取，授权链接和 token 只发到你的 Telegram，日志全程掩码（`::add-mask::`），不泄露敏感信息。

**第一步：配置 Secrets**（仓库 Settings → Secrets and variables → Actions）：

| Secret | 说明 |
|---|---|
| `TG_BOT_TOKEN` | Telegram bot token（找 @BotFather 创建，如 `123456:ABC-xxx`） |
| `TG_CHAT_ID` | 你的 Telegram 数字 chat id（给 @userinfobot 发消息获取） |

**第二步：运行工作流**：

1. 仓库页面 → **Actions** → 左侧 **获取 Freebuff authToken** → **Run workflow**
2. 可选填 `poll_timeout`（授权等待秒数，默认 300）和 `fingerprint`（留空自动生成）
3. 你的 TG 会收到登录链接，浏览器打开并登录 Google 账号
4. 脚本轮询到 token 后，完整 token 直接发到你 TG（Actions 日志里只有 `***`）
5. 跑完自动清理旧运行记录，只保留最新 1 条

> 没配 `TG_BOT_TOKEN` / `TG_CHAT_ID` 时工作流第一步直接失败，不会执行提取。

### 方式 B：本地提取

```bash
cd freebuff_tools
python3 extract_freebuff.py login   # 打印授权 URL 到终端，浏览器授权后自动轮询
python3 extract_freebuff.py show    # 显示全部账号：邮箱 + token + 存活状态 + 汇总一行一个
python3 extract_freebuff.py tgsend  # 测试 TG 连通性（配了 TG 时用）
```

本地运行 `login` 时，每个账号会**分键追加**保存到 `freebuff_tools/freebuff_credentials.json`（不覆盖已有账号，支持 Google / GitHub 登录，均自动记录）。该文件已被 `.gitignore` 忽略，不会提交到 GitHub；结构参考 `freebuff_tools/freebuff_credentials.example.json`。

其他实用命令：

```bash
python3 extract_freebuff.py export           # 汇总全部账号 token，一行一个，直接复制进 CF Workers 变量
python3 extract_freebuff.py quota            # 查用量
python3 extract_freebuff.py session          # 开/查 session
python3 extract_freebuff.py chat "你好"      # 发一条消息测试模型 API
```

> 💡 `show` 内部用 `GET /api/v1/freebuff/session` 探测每个账号（**不创建 session、0 消耗**），一次显示全部状态：存活 + 额度 / token 失效 / 被封禁 / 地区受限 / 额度用完。官方对 banned 账号会在所有接口返回 `status: banned`。多账号时 `export` 输出的每行 token 直接粘贴到 Cloudflare Worker 变量 `FREEBUFF_TOKEN`（换行分隔）即可。

## 🛡️ 管理面板 `/admin`

账号、代理、API Key、额度**全在页面上改，保存即生效，无需重启容器**。

```
http://<服务器IP>:8877/admin
```

（端口以 `-p 宿主机端口:容器端口` 的**左侧**为准 —— 本项目所有示例都是 **`8877`**；`8787` 只是容器**内部** `server.js` 的监听端口（`PORT` 默认值），宿主机上不暴露。）

### 登录与首次改密

- **初始密码固定为 `admin`**（不随机），同时打印在容器日志：`docker logs <容器> | grep 初始密码`，也写入 `credentials/.initial_password`
- **首次登录强制改密**：接口返回 `must_change_password: true`，页面跳「必须修改初始密码」，新密码 **≥ 8 位**，不改进不了面板
- 密码存 `credentials/admin.json`，格式 `sha256(salt:password)` + 随机 salt，**明文不落盘**
- 登录会话 TTL **12 小时**（`session_ttl_ms`），连续失败 5 次锁定 15 分钟
- **忘记密码**：删掉 `credentials/admin.json` 后重启容器 → 重建回初始密码 + 强制改密；账号池若为空会自动从旧凭证文件 / env 迁移（`migrateLegacyAccounts`）

### 卡片 ①：账号池（一个账号一条独立 socks5 出站）

每行字段：

| 字段 | 说明 |
|---|---|
| 名称 | 备注用，随便起（`acct-01`、`家宽2`…） |
| Token（留空=保留原值） | Freebuff 的 `authToken`，**留空保存就不动它** |
| SOCKS5 出站 | `socks5://user:pass@host:port`，**留空 = 直连** |
| 启用 | 勾选框，关掉即摘出池子 |

行内四个按钮：

| 按钮 | 作用 |
|---|---|
| **测代理** | 走这条 socks5 出口打上游，返回**出口 IP + 延迟**（`✅ 出口IP 1.2.3.4（856ms）`） |
| **测账号** | 直接打上游 `GET /api/v1/me`，返回 `HTTP <status> <body>`，`account_suspended` 一眼可见 |
| **查额度** | 拿该账号的 Freebucks 快照（见下），**GET 零消耗** |
| **删除** | 从池子里摘掉（需再点「保存账号」落盘） |

池子下方三个按钮：**＋ 添加账号**、**保存账号**（落盘 + 立即生效）、**💰 查全部额度**（逐账号查，渲染成表格）。

### 卡片 ②：服务变量

| 变量 | 作用 |
|---|---|
| API Key（客户端鉴权） | 客户端调 `/v1/chat/completions` 要带的 key |
| CODEBUFF_API（留空=官方） | 上游域名，留空 = `https://www.codebuff.com` |
| RELAY_KEY | 中继密钥（`CODEBUFF_API` 指向带鉴权的中继时用） |
| DEBUG | 开关请求级调试日志 |

点 **保存变量** 即生效。

### 卡片 ③：服务状态

数据源 `GET /admin/api/status`，返回 `version`（worker.js 版本）、`token_pool_size`、`models_count`、`accounts`（名称 / 代理 / 启用 / 掩码 token）。

### 卡片 ④：修改管理员密码 / 退出登录

当前密码 + 新密码（≥8 位）。

### 查额度按钮

- 账号行内 **查额度** → `POST /admin/api/quota`（body：`{index, token?, socks5?}`）
- 池子下方 **💰 查全部额度** → `GET /admin/api/quotas`

两者都由**服务端发起**，走该账号配置的 socks5 出站，请求上游 `GET /api/v1/freebuff/session` 并带 `include-unused-rate-limits: 1`，返回字段：

```
ok / status / access_tier / status_field
balance / wallet_balance / plan_id
daily: { limit, spent, remaining, resetAt }
prices: { 模型: 单价 }
price_notices / off_peak / price_changes
```

被封时返回 `banned: true` + `error: account_suspended`，界面显示 `❌ 账号被封：…`。

渲染效果：

```
💰 Freebucks 20/100（已用 80）· 重置 2026-09-23 15:00 · tier=full · plan=xxx
可打次数：kimi-k3-eco×4 · glm-5.3-flash×4 · mimo-v2.5×2 · deepseek-v4-flash×1 · gpt-5.6-luna×1 · gemini-3.8-flash×0
```

「可打次数」= `floor(daily.remaining / prices[模型])`，一眼看出**今天还打得起哪几个模型**（贵的直接 ×0）。

> ⏱ `resetAt` 是**太平洋时间午夜**，`fmtReset()` 按浏览器本地时区渲染（东八区 = **次日 15:00**）。

### 面板 API（可编程）

全部在 `/admin/api/` 下，登录后用 Cookie `admin_session=<sid>` 调用：

| 路径 | 方法 | 说明 |
|---|---|---|
| `/admin/api/login` | POST | `{password}` → `Set-Cookie` + `must_change_password` |
| `/admin/api/logout` | POST | 注销会话 |
| `/admin/api/accounts` | POST | `{accounts:[...]}` 全量保存账号池 |
| `/admin/api/settings` | POST | `{settings:{api_key,codebuff_api,relay_key,debug}}` |
| `/admin/api/config` | GET | 读当前配置 |
| `/admin/api/status` | GET | 服务状态（版本 / 模型数 / 账号表） |
| `/admin/api/test_proxy` | POST | `{socks5}` → 出口 IP + 延迟 |
| `/admin/api/test_account` | POST | `{index,token,socks5}` → 上游 `/me` 响应 |
| `/admin/api/quota` | POST | `{index,token,socks5}` → 单账号 Freebucks |
| `/admin/api/quotas` | GET | 全部账号 Freebucks |
| `/admin/api/change_password` | POST | `{old_password,new_password}` |

> ⚠️ 面板接口与 `/v1/*` 同端口，但**只认 `admin_session` Cookie**；`Authorization: Bearer <API_KEY>` 只用于 `/v1/*`，反过来也一样不通。

---

## 🛠️ 部署

### 🐳 Docker 容器化部署（✅ 推荐）

> 适合本地/NAS/VPS 长期运行：不受 Cloudflare Workers 限制，**不会暴露 CF 边缘标记**（`cf-worker` / `cf-ray`），账号封禁风险显著低于 CF 部署。镜像已发布到 **Docker Hub**，**无需 clone 仓库、无需构建**，一条命令即可部署。
>
> 镜像地址：`pingmike/freebuff2api:latest`（[Docker Hub 页面](https://hub.docker.com/r/pingmike/freebuff2api)）

---

#### 方式一：一键 `docker run`（最快）

> 💡 **账号不必靠环境变量**：容器起来后打开 `http://<服务器IP>:8877/admin`（初始密码 `admin`，首登强制改密），在**账号池**里粘贴 token 点**保存账号**即可，**立即生效、不用重启**。下面挂载 `freebuff_credentials.json` 是旧路径 —— 只在**面板账号池为空时**才会被自动迁移进去。
> 另注意：`-e FREEBUFF_API_KEY` 会被面板「服务变量」**覆盖**（面板值优先，见「[环境变量（真实优先级）](#环境变量真实优先级)」）。

```bash
# 1. 准备凭据文件 freebuff_credentials.json（多账号聚合格式，见「获取 authToken」）
#    用提取工具生成：python3 freebuff_tools/extract_freebuff.py login
#    或手动创建：{"accounts": {"<账号id>": {"email": "...", "authToken": "...", "name": "..."}}}

# 2. 一键启动（变量直接内联，或改用 .env 文件）
docker run -d --name freebuff2api --restart unless-stopped \
  -p 8877:8787 \
  -e PORT=8787 \
  -e HOST=0.0.0.0 \
  -e FREEBUFF_API_KEY=your-api-key \
  -e RELAY_KEY= \
  -v "$(pwd)/freebuff_credentials.json:/app/credentials/freebuff_credentials.json:ro" \
  pingmike/freebuff2api:latest

# 3. 拿初始密码并打开面板
docker logs freebuff2api | grep 初始密码
```

变量多时也可以用 `.env` 文件（`docker run --env-file .env`）：

```bash
cat > .env <<'EOF'
PORT=8787
HOST=0.0.0.0
FREEBUFF_API_KEY=your-api-key
RELAY_KEY=
EOF

docker run -d --name freebuff2api --restart unless-stopped \
  -p 8877:8787 \
  --env-file .env \
  -v "$(pwd)/freebuff_credentials.json:/app/credentials/freebuff_credentials.json:ro" \
  pingmike/freebuff2api:latest
```

---

#### 方式二：docker compose（推荐长期运行）

```bash
# 1. 一条命令：创建目录 → 写 compose → 配置 .env → 启动
mkdir -p freebuff2api && cd freebuff2api && \
cat > docker-compose.yml <<'EOF'
services:
  freebuff2api:
    image: pingmike/freebuff2api:latest
    container_name: freebuff2api
    restart: unless-stopped
    ports:
      - "8877:8787"
    environment:
      - PORT=8787
      - HOST=0.0.0.0
      - FREEBUFF_API_KEY=${FREEBUFF_API_KEY}
      - RELAY_KEY=${RELAY_KEY}
    volumes:
      - ./freebuff_credentials.json:/app/credentials/freebuff_credentials.json:ro
EOF
echo 'FREEBUFF_API_KEY=your-api-key' > .env && \
docker compose pull && docker compose up -d
```

> 💡 compose 里的 `${FREEBUFF_API_KEY}` / `${RELAY_KEY}` 会自动从同目录的 `.env` 文件读取。

**凭据文件：** 启动前/后放入账号凭据，放入后重启容器生效：

```bash
chmod 600 freebuff_credentials.json
# freebuff_credentials.json 多账号聚合格式：{"accounts": {"<账号id>": {"email": "...", "authToken": "...", "name": "..."}}}
docker compose restart          # 或 docker restart freebuff2api
```

---

#### 更新方式（不用重新构建、不用重新 pull）

镜像采用**容器引导器模式**（借鉴 [fscarmen/Argo-Nezha-Service-Container](https://github.com/fscarmen/Argo-Nezha-Service-Container)）：容器每次启动时会自动从 GitHub raw 地址拉取最新 `worker.js`（拉取失败则回退镜像内置副本）。因此服务端改完 `worker.js` 推送 GitHub 后，**主部署重启容器即自动更新**；客户端侧偶尔 pull 新镜像即可：

```bash
docker compose pull && docker compose up -d   # 拉新镜像并重建
docker compose restart                        # 或仅重启（自动拉最新 worker.js）
```

> 💡 **如何确认实际运行的版本**：由于镜像 tag 与容器内实际代码是脱钩的（tag 只代表镜像构建时间，运行时代码以启动时拉取的 `worker.js` 为准），**镜像 tag 不是版本号**。要确认当前实际运行的版本，直接查免鉴权的健康检查端点：
>
> ```bash
> curl -s http://localhost:8877/healthz
> # 返回体中的 "version" 字段即容器内 worker.js 的真实版本
> ```
>
> 返回的 `version` 字段才是容器内 `worker.js` 的真实版本，与 `docker images` 显示的 tag 无关。若担心容器因拉取失败回退到了镜像内置的旧副本，用上面的命令核对即可（比如镜像 tag 是 1.7.0，但 healthz 返回 `"version":"1.8.9"` 就说明容器已自动更新到最新）。

#### 环境变量（真实优先级）

> ⚠️ **面板里的「服务变量」会覆盖 `.env` / `-e` 传入的值**，`buildEnv()` 实际取值顺序是：**面板账号池 > 面板变量 > `.env` > 默认值**；`PORT`/`HOST` 只认 `.env`（面板无此项）。

| 变量 | 唯一来源 | 说明 |
|---|---|---|
| `FREEBUFF_TOKEN` | ✅ **面板账号池** | `-e FREEBUFF_TOKEN=...` **只在首次迁移时**被吸进面板（`migrateLegacyAccounts`），之后 `buildEnv` 只读 `getAccounts()`，**再改 env 无效** → 账号请在面板里加 |
| `FREEBUFF_API_KEY` | 面板优先，空则 `.env`，再空则 `freebuff-default-key` | **客户端调用的 key 以面板 `settings.api_key` 为准** |
| `CODEBUFF_API` | 面板优先，空则 `.env` | 上游地址，默认空 = `https://www.codebuff.com` |
| `RELAY_KEY` | 面板优先，空则 `.env` | 中继密钥（`CODEBUFF_API` 指向带鉴权的中继时必填） |
| `FREEBUFF_DEBUG` | ✅ **面板** `settings.debug` | 只在**首次创建** `admin.json` 时读一次 env，之后 env 无效 → 要改请在面板里改 |
| `PORT` / `HOST` | ✅ `.env` | 监听端口/地址，默认 `8787` / `0.0.0.0` |
| `FREEBUFF_SESSIONS_JSON` / `WEB_CREDENTIALS_FILE` | `.env` | 网页版 Cookie 认证通道 |
| `RELAY_URL` / `RELAY_METHOD` / `RELAY_HEADERS` / `RELAY_BODY` | `.env` | relay 出站，`API_READY` 触发 |

**想让面板变量失效、回落到 `.env`** → 在面板里把对应字段**清空并保存**，`buildEnv` 即回落读 env。

> ⚠️ 凭据兼容两种格式：多账号聚合 `{"accounts": {...}}`（提取工具默认输出）和单账号顶层 `authToken`。旧的 `freebuff_credentials.json` 会在**面板账号池为空时**被自动迁移进面板；迁移后以面板为准。

#### 维护者：发布新镜像到 Docker Hub

仓库已配置 `.github/workflows/docker-publish.yml`（手动触发，多架构 amd64/arm64）。在 GitHub Secrets 配置 `DOCKERHUB_USERNAME` 与 `DOCKERHUB_TOKEN` 后，到 Actions 页面手动 **Run workflow** 即可发布新镜像。

### Cloudflare Worker 部署（❌ 不推荐）

> **Freebuff 官方已检测 Cloudflare Worker 部署**（识别 `cf-worker` / `cf-ray` 等边缘标记，源码中已点名类似本项目的代理模式）。在 CF 上部署会显著增加账号被封禁的风险，**不推荐作为主要部署方式**；以下步骤仅保留给熟悉风险的用户参考。

worker 是**单文件**（`worker.js`），如仍需在 CF 部署：

### 方式 A：CF 控制台粘贴代码

最简单可控，不依赖本地环境、不关联 GitHub：

1. 打开 [dash.cloudflare.com](https://dash.cloudflare.com) → **Workers & Pages** → **创建** → **创建 Worker**
2. 名称随意（如 `freebuff2api`），点击 **部署**
3. 进入该 Worker → **编辑代码** → 把 [worker.js](worker.js) 的**全部内容**粘贴进去，覆盖默认代码 → **部署**
4. 点 **设置 → 变量和机密 → 添加**：

   | 类型 | 名称 | 值 |
   |---|---|---|
   | 机密 | `FREEBUFF_TOKEN` | 你的 freebuff token（多账号用英文逗号分隔） |
   | 机密 | `FREEBUFF_API_KEY` | 自定义访问 key（可选，不设则用 `freebuff-default-key`） |

5. 部署完成后访问验证：

   ```bash
   curl https://你的worker.workers.dev/healthz          # 健康检查（无需 key）
   curl https://你的worker.workers.dev/v1/models \
     -H "Authorization: Bearer ***"           # 模型列表
   ```

> 每次改代码只需重复第 3 步：编辑代码 → 粘贴新内容 → 部署。**不推荐关联 GitHub 自动部署**（见下文）。
> ⚠️ **版本约定**：每次部署前务必把代码里的版本号（healthz 的 `version` 字段 + `X-Freebuff2api-Version` 响应头）升一档，否则无法确认线上是否已更新。

### 关联 GitHub 自动部署（❌ 不推荐）

虽然 CF 支持连接 GitHub 仓库自动部署，但**不建议用**：

- 每次 push 都会触发上线，本地未验证的改动可能直接打到线上
- 需要额外配置构建命令/根目录，仓库里的 `freebuff_tools/` 等辅助文件也会被拉取
- secrets 与分支状态容易混乱，出问题不好排查
- 本仓库含 token 提取脚本，自动同步增加暴露面

**推荐做法**：本地改代码 → Docker 容器/自建 VPS 部署，或（了解风险的前提下）手动粘贴到 CF 控制台 → 自己点部署，完全可控。

> 免费模型对出口 IP 有 US 限制，Cloudflare Workers 默认美国出口，无需额外配置。

### 🌐 自定义域名

默认域名 `https://你的worker名.你的子域.workers.dev` 在部分地区可能访问不通（如被墙/GFW 限制）。如果遇到 `workers.dev` 连接超时或无法访问，可以给 Worker 绑定自己的域名：

1. **添加自定义域**：CF 控制台 → 你的 Worker → **设置 → 域和路由** → **添加** → **自定义域**
2. 输入你的域名（如 `api.你的域名.com`），CF 会自动引导添加 DNS 记录（CNAME 指向 `你的worker名.你的子域.workers.dev`）
3. 等待 DNS 生效（一般几分钟），自动签发免费 SSL 证书
4. 之后 Base URL 改为：`https://api.你的域名.com/v1`

> 要求：域名必须托管在 Cloudflare（或把 DNS 转到 CF）。workers.dev 子域无需配置，绑定自定义域只是给访问不通的地区多一条可用路径。

## 💬 调用示例

```bash
# 健康检查
curl https://你的worker.workers.dev/healthz

# 模型列表
curl https://你的worker.workers.dev/v1/models \
  -H "Authorization: Bearer <API_KEY>"

# 非流式
curl https://你的worker.workers.dev/v1/chat/completions \
  -H "Authorization: Bearer <API_KEY>" -H "Content-Type: application/json" \
  -d '{"model":"deepseek/deepseek-v4-flash","messages":[{"role":"user","content":"你好"}]}'

# 流式
curl -N https://你的worker.workers.dev/v1/chat/completions \
  -H "Authorization: Bearer <API_KEY>" -H "Content-Type: application/json" \
  -d '{"model":"deepseek/deepseek-v4-flash","messages":[{"role":"user","content":"你好"}],"stream":true}'
```

## 📋 模型列表

> 映射来源：Freebuff Desktop 0.0.51（`orchestrator.js` 官方 `FREEBUFF_ROOT_AGENT_ID_BY_MODEL`，2026-08-07 实测同步）。
> Worker 通过 Cloudflare Workers 访问上游，默认使用美国出口，按 Freebuff 完整访问模式说明。**额度不按模型分白名单，而是统一的 Freebucks 每日钱包** —— 每个模型有自己的**单价**，建 session 时按单价扣（见「[💰 Freebucks 额度机制](#-freebucks-额度机制)」）。实时完整列表 `curl /v1/models`（当前 20 个）。

| API 模型名 | session 模型 | 上游 agentId | 说明 |
|---|---|---|---|
| `deepseek/deepseek-v4-flash` | 同左 | `base2-free-deepseek-flash` | 主力推荐 |
| `mimo/mimo-v2.5` | 同左 | `base2-free-mimo` | 均衡性能 |
| `minimax/minimax-m3` | 同左 | `base2-free-minimax-m3` | - |
| `deepseek/deepseek-v4-pro` | 同左 | `base2-free-deepseek` | - |
| `openai/gpt-5.6-luna` | 同左 | `base2-free-luna` | - |
| `poolside/laguna-s-2.1` | 同左 | `base2-free-laguna-s-2-1` | - |
| `openrouter/poolside/laguna-s-2.1` | 同左 | `base2-free-laguna-s-2-1-openrouter` | - |
| `inclusionai/ling-3.0-flash:free` | 同左 | `base2-free-ling-3-flash` | - |
| `crof/greg-2-ultra` | 同左 | `base2-free-greg-2-ultra` | - |
| `crof/greg-2-super` | 同左 | `base2-free-greg-2-super` | - |
| `meta/muse-spark-1.2-contributor` | 同左 | `base2-free-muse-spark` | - |
| `z-ai/glm-5.2` | 同左 | `base2-free-glm` | 需 referral / streak 等官方资格，使用独立额度池 |
| `anthropic/claude-fable-5` | 同左 | `base2-free-fable` | 官方容量限制试用，可能按时段开放 |

> 📝 实测补充（2026-08-08）：`ling-3.0-flash:free` 上游可能返回 404 并提示改用付费 slug；`claude-fable-5` 免费账号建 session 可能被上游拒绝（`session_model_mismatch`）。这些现象属于上游可用性问题，不代表 Worker 映射失效。

## 👥 多账号

**多账号在面板里加**：账号池每行一个账号（名称 + Token + 一条独立 socks5 出站），点 **保存账号** 立即生效。撞额度（429/空响应）时自动冷却当前账号并切下一个，每个号**额度独立**，等于把 Freebucks 每日钱包叠加起来。

（旧的 `FREEBUFF_TOKEN` 环境变量逗号分隔方式仅在**首次迁移**时被吸进面板，之后以面板账号池为准。）

**账号选择策略**（v1.4.0 起）：

1. 优先复用**已有活跃 session 缓存**的账号——session 约 1 小时有效，创建才扣额度，复用不扣；
2. 没有活跃缓存时才轮询下一个账号。

这样多账号叠加 Freebucks 额度，利用率最大化（见「[💰 Freebucks 额度机制](#-freebucks-额度机制)」）。

> 注意：冷却状态存在 Worker 内存，冷启动后重置；并发多实例间不共享。日常使用影响不大。

## 🔍 上游门控说明

freebuff 免费模型不是"拿 token 直接调 chat"就行，而是有严格生命周期：

```
session(开) → agent-runs(主+context-pruner 子run) → chat/completions
```

- **session**：`POST /api/v1/freebuff/session`（带 `x-freebuff-model`）拿 `instanceId`；可能排队（queued）。
- **agent-runs**：`START` 主 agent（如 `base2-free-deepseek-flash`）+ `context-pruner` 子 run，并 `record_step` / `finish_run`。chat 校验 run_id 存在，缺了会 4xx。
- **chat**：`POST /api/v1/chat/completions`，带 `codebuff_metadata.run_id`、`x-freebuff-instance-id`、SDK UA、`stop:['"cb_easp"']`、`provider.data_collection=deny`。**上游强制流式**，非流式请求需聚合（超时已放宽至 45s）。

Worker 已自动处理以上全部生命周期，无需手动干预。另：system 消息必须以 `You are Buffy, the strategic coding assistant.` 开头（上游字节级校验），Worker 已自动注入。

### ⚠️ 单账号单会话限制（重要）

一个 Freebuff 账号同一时间**只能一个客户端在线**。因此：

- ❌ 禁止在 `/v1/models` 中查询上游 `GET /api/v1/freebuff/session` 探测额度/状态——该调用会占用 session 并顶掉正在进行的 chat（428 `waiting_room_required`）。
- ✅ `/v1/models` 返回**静态模型列表**（不额外调上游）。
- 上游请求通过**串行队列 + 300ms 间隔**执行，避免并发触发上游问题。

## 💡 使用体验

目前测试过以下方式，效果都不错：

1. **🌍 美国 IP 直连**：freebuff 免费模型对出口 IP 有 US 限制，非美区 IP 可能失败。Cloudflare Workers 默认美国出口，直连即可；本地客户端访问建议配合美国代理。

2. **🤖 Hermes Agent（美区 VPS）**：将 Hermes Agent 部署在美区 VPS 上。

3. **本地浏览器 + page-assist 插件**：配合 [page-assist](https://github.com/n4ze3m/page-assist) 浏览器插件使用，体验流畅，欢迎尝试。

## 🙏 感谢

感谢以下贡献者对本项目的支持与贡献（排名不分先后）：

- [@yjzsg](https://github.com/yjzsg)
- [@zipei-a](https://github.com/zipei-a)
- [@hknerdr](https://github.com/hknerdr)

## 📚 学习参考项目

本项目在开发过程中参考并学习了以下开源项目，特此感谢：

- [freebuff2api](https://github.com/XxxXTeam/freebuff2api) —— freebuff 桌面版/API 协议逆向与代理的原始实现（AGPL-3.0），本项目在其基础上进行二次开发与优化，并沿用 AGPL-3.0 开源。
- [freebuff](https://github.com/CodebuffAI/freebuff) —— freebuff 官方公开源码，本项目通过阅读其协议实现与更新日志进行学习研究。
- [Argo-Nezha-Service-Container](https://github.com/fscarmen/Argo-Nezha-Service-Container) —— **容器引导器模式**（Dockerfile 只做引导，业务逻辑由远程脚本/代码驱动），本项目 Docker 部署方式借鉴了该设计，实现"改代码即更新、重启即生效"的轻量管理。

## ⚠️ 免责声明

本项目仅供**技术交流与学习研究**使用。

- 本项目通过逆向 freebuff 桌面版/API 协议实现代理，**违反 freebuff 官方服务条款（ToS）**。
- 使用本项目存在**账号被封禁（banned）的风险**，且封禁为终态、不可恢复，请知悉并自行承担后果。
- 请勿用于商业用途或大规模滥用，请尊重 freebuff 服务提供方的运营。
- 使用者需自行遵守所在地法律法规及 freebuff 官方条款，本项目作者不对任何账号损失或纠纷负责。

## 📄 License

本项目采用 [AGPL-3.0 License](LICENSE)。本项目参考并改写了 [freebuff2api](https://github.com/XxxXTeam/freebuff2api) 的部分代码与结构（原项目为 AGPL-3.0），因此本项目同样以 AGPL-3.0 开源；使用时请保留原版权声明，欢迎自由使用、修改与分享。

---
# English Documentation

[![License: AGPL-3.0](https://img.shields.io/badge/License-AGPL--3.0-blue.svg)](LICENSE)

> 🎉 Issues and PRs are welcome.
>
> 🌐 **Language / 语言**: [中文](#freebuff2api-workers) · **English**
> License: **[AGPL-3.0](#license)**

Expose **freebuff / codebuff**'s free models as an **OpenAI-compatible API**. Single file, zero dependencies, **Docker deployment recommended** (or run it on your own VPS). Works with any OpenAI SDK / client (QwenPaw, Hermes, ChatGPT-Next-Web, LobeChat, one-api, …).

> ⚠️ **Deployment warning**: Freebuff now detects Cloudflare Worker deployments (it fingerprints edge markers such as `cf-worker` / `cf-ray`), and deploying on CF **significantly raises the risk of an account ban**. Cloudflare deployment is therefore **not recommended** — use **Docker** or a VPS instead (see [🐳 Docker deployment](#docker-deployment-recommended)).

## Features

- ⭐ **Full-access models**: Cloudflare Workers egress from the US by default, which usually lands in Freebuff's full-access mode; DeepSeek V4 Flash and MiMo 2.5 are official non-Premium models
- 🛡️ **Web admin panel `/admin`**: accounts, proxies, API key and **Freebucks quota** are all editable in the browser — **save and it takes effect, no container restart** (see [🛡️ Admin panel](#admin-panel-admin))
- 💰 **Freebucks quota visibility**: a **daily wallet charged per call** (not a "N calls per model per day" allowlist); the panel shows balance, daily usage, the price table and **which models you can still afford today** (see [💰 Freebucks quota model](#freebucks-quota-model))
- 🔁 **Multi-account rotation**: on quota exhaustion the current account cools down and the next one takes over; accounts are added in the panel, **not** via the `FREEBUFF_TOKEN` env var
- 💡 **Active-session reuse**: a session lasts ~1 hour and only creation is charged; while the current session is alive the worker stays pinned to that account and only switches when it is used up, maximising quota efficiency
- 📢 **Ads and streak flow compatible**: before creating a session the worker follows the official client flow (ad request, then `GET /api/v1/freebuff/streak` check-in). Failures are silently skipped and never block the chat
- 🧩 **OpenAI-compatible**: `/v1/models`, `/v1/chat/completions`, `/v1/responses` (streaming / non-streaming depending on the endpoint)
- 📨 **Anthropic Messages API**: `/v1/messages`, `/messages` and the matching `count_tokens` routes, for Anthropic SDKs and compatible clients
- ❤️ **Health check**: `GET /healthz` (no auth), easy to hook into monitoring
- 📦 **Single-file deploy**: no dependencies, one `worker.js`, works on CF / Docker / VPS

## Anthropic Messages API support

The main code ships with an Anthropic Messages API adapter supporting:

- `POST /v1/messages`
- `POST /messages`
- `POST /v1/messages/count_tokens`
- `POST /messages/count_tokens`
- Conversion of Anthropic message format into the worker's internal OpenAI-compatible request
- Text messages, `tool_use` / `tool_result`, `tool_choice`
- Non-streaming responses and Anthropic SSE streaming
- Anthropic-style error responses

> ⚠️ **Testing caveat**: the maintainer has no real Anthropic client environment, so end-to-end testing against actual Anthropic clients has not been done. The conversion logic is covered by local stubs and regression tests, but that does not mean every Anthropic SDK, tool-call combination and client behaviour has been exercised.
>
> If you have a real Anthropic Messages API use case, feel free to test it as long as it does not disturb the OpenAI path, and report back request shape, streaming behaviour, tool-call or model compatibility issues. Please attach redacted request structures, status codes and error messages.

> The Anthropic API is an additional protocol adapter: it does not change the existing OpenAI `/v1/chat/completions` and `/v1/responses` paths, account rotation, session lifecycle or the main Freebuff call chain.

## Freebucks quota model

Freebuff's free tier is metered in **Freebucks** — a **daily wallet charged per call**, **not** a "N calls per model per day" allowlist. So a 429 after switching models usually means **the new model is more expensive and the wallet cannot cover it**, not that you were blocked.

> 📌 The data in this section was measured on **2026-09-22** (upstream `GET /session` snapshot + official `freebucks-pricing.ts` / `freebuff-spend-ceilings.ts`). Official rules change — **always trust the live `prices` shown by the "Query quota" button in the panel**.

### What is charged: session creation, not chat turns

Charging happens on `POST /api/v1/freebuff/session`:

- A session is valid for **1 hour**; reusing it during that window is **free** (the worker reuses any cached session with >60s left)
- So the real shape is "**N session slots per day**", not "N chat turns"
- `DELETE session` triggers an **asynchronous refund** (`freebucksRefundPending: true`, balance recovers a moment later); a **forged `instanceId` gets no refund** (`freebucksRefund: 0`)

### Price table (`freebucks.prices`, measured)

| Model | Unit price | Calls at a 40/day cap |
|---|---|---|
| `z-ai/glm-5.3-flash` | 5 | **8** |
| `crof/kimi-k3-eco` | 5 | **8** |
| `mimo/mimo-v2.5` | 10 | **4** |
| `upstage/solar-pro4` | 10 | **4** |
| `deepseek/deepseek-v4-flash` | 15 (10 off-peak) | **2** (off-peak **4**) |
| `deepseek/deepseek-v4-flash-max` | 15 | **2** |
| `deepseek/deepseek-v4-pro-max` | 20 | **2** |
| `openai/gpt-5.6-luna` | 20 | **2** |
| `mimo/mimo-v2.6-pro` | 30 | **1** |
| `google/gemini-3.8-flash` | 80 | **0 — not even one call per day** |

- **Off-peak** (`freebucks.offPeak`) discounts some models: `deepseek-v4-flash` drops 15 → **10** during **06:00–14:00 Beijing time**
- `deepseek-v4.1-flash` / `v4.1-pro` **can create sessions (200 active measured)** but have **no entry in the price table** yet, so the unit price is unknown — check `prices` with the quota button
- `deepseek/deepseek-v4-pro` returns `409 model_unavailable`; it has been retired upstream
- `freebucks.priceNotices` carries price-change notices, rendered in the panel as `⚠ model: message`

### Two limits coexist: trust the snapshot's `daily.limit`

The upstream reports **two different numbers** — do not mix them up:

| Source | Number (varies) | Meaning |
|---|---|---|
| **429 body** `limit` | 40 / 25 … | That day's throttle — `recentCount ≥ limit` returns 429 |
| **GET snapshot** `daily.limit` | 100 / 25 … | Wallet daily allowance (`spent` is measured against it) |

**Never derive your balance from the 429 limit minus `recentCount`** — only trust the snapshot's `daily.remaining` and `balance`.

### accessTier and "a new account only gets 25"

The quota is **tiered per account**; different `accessTier` means a different daily ceiling:

| accessTier | Daily ceiling (`daily.limit`) | Notes |
|---|---|---|
| `full` | 100 measured (429 throttle 40) | Measured on an older account before 2026-09-22; the official cap can change at any time |
| `limited` | **25** | **Default for newly registered accounts** — a fresh account really only has 25/day, it is not a bug |

Measured on 2026-09-23: a new account is `accessTier: "limited"` with `daily.limit: 25`; at 5 points per glm-5.3-flash call that is roughly 5 calls a day.

The official `freebuff-countries.ts` shows the tier is also **derived from the account's egress country** (US/CA/GB/AU and ~18 more countries = `full`; everything else, plus any VPN, = `limited`), and account trust can layer on top of that. Older accounts can also be demoted from `full` to `limited`.

### Reset time

`resetAt` is **Pacific-time midnight**, i.e. **15:00 Beijing time the next day** (e.g. `2026-09-23T07:00:00Z` = 2026-09-23 15:00 Beijing).

> 📌 **Anchor trivia (GitHub Markdown)**: when generating anchors for headings containing emoji, GitHub strips only the base emoji code point and keeps the variation selector U+FE0F. Fix anchors against the real `id` in the rendered HTML.

### Where the official cap comes from

The upstream source `freebuff-spend-ceilings.ts` says:

> Capacity is now limited per account — sustained automated abuse forced us to cap how much any one account can use.

`accessTier: "full"` means the account is **not demoted** and merely hit the generic cap.

### Scaling up

1. **Paid tier** — `$8/month` → `150 Freebucks/day` (visible in `wallet`)
2. **Multi-account rotation** — the one-account-one-SOCKS5 architecture **supports this natively**; each account has its own allowance, add them in the panel. **Note a new account defaults to the `limited` tier at 25/day**, so N accounts multiply 25, not 100
3. **Pick cheap models** — prefer `glm-5.3-flash` / `kimi-k3-eco` (unit price 5) and save expensive models for off-peak

### Common misconceptions

| Symptom | Reality |
|---|---|
| 429 after switching models | Not a model allowlist — **that model is just more expensive** than the wallet can cover |
| `HTTP 502: create session failed: 429 ...` | Upstream quota exhausted, wait for `resetAt` |
| `409 model_locked` | Session lock conflict, **not a quota problem** — the worker deletes and recreates automatically (v1.8.11.1) |
| `429` | **Not a ban**, only quota; `account_suspended` is the actual ban (terminal, irreversible) |
| Why does a new account only have 25? | New accounts default to `limited`, 25/day — **not a bug and not an IP issue** |
| Quota drains unusually fast | Repeated diagnostic session creation charges every time — **each POST is a real charge** |

## Quick start

1. Get a freebuff token (see [Get FREEBUFF_TOKEN](#get-freebuff_token))
2. Deploy the service (see [Deployment](#deployment), **Docker recommended**)
3. Open the admin panel at `http://<server-ip>:8877/admin` (initial password `admin`, **forced change on first login**), paste the token into the **account pool** and click **Save accounts**
4. Point any OpenAI client at it:
   - **Base URL**: `http://localhost:8877/v1` (Docker) or `https://<worker>.<subdomain>.workers.dev/v1` (CF, not recommended)
   - **API Key**: the value of "Service variables → API Key" in the panel (defaults to `freebuff-default-key`)

> 🌐 **Custom domain**: if `*.workers.dev` is unreachable in your region, bind your own domain to the Worker and use `https://<your-domain>/v1` as the Base URL. See [Custom domain](#custom-domain).

## Health check

Available after deployment (**no API key needed**):

```bash
curl https://<worker>.<subdomain>.workers.dev/healthz
# {"status":"ok","version":"1.8.11.1","accounts":1,"alive_accounts":1,"account_states":{"ok":1},"health_source":"worker_cache","time":"..."}
```

- `version` is the deployed **worker.js version**, useful to confirm the latest code is live (CF edge caching lags; wait a few seconds or add a random query parameter)
- `status` is `ok` / `degraded` / `critical`; `account_states` counts accounts per state (`ok` / `rate_limited` / `banned` / `model_locked` …)
- ⚠️ **Status only, no quota numbers** — for Freebucks use the "Query quota" button in the panel (see [🛡️ Admin panel](#admin-panel-admin))
- Suitable for UptimeRobot or your own monitoring

## Get FREEBUFF_TOKEN

The freebuff credential (`authToken`) is obtained with the same **auth-code polling** flow the official CLI uses. The repo ships `freebuff_tools/extract_freebuff.py`, which behaves like `cline_oauth.py`.

### Option A: GitHub Actions workflow (recommended, remote)

The repo includes `.github/workflows/extract-token.yml`. Extraction runs inside GitHub Actions; the auth link and the token are sent only to your Telegram, and the logs are masked end to end (`::add-mask::`), so nothing sensitive leaks.

**Step 1 — configure secrets** (Settings → Secrets and variables → Actions):

| Secret | Description |
|---|---|
| `TG_BOT_TOKEN` | Telegram bot token (create one with @BotFather, e.g. `123456:ABC-xxx`) |
| `TG_CHAT_ID` | Your numeric Telegram chat id (message @userinfobot) |

**Step 2 — run the workflow**:

1. Repo page → **Actions** → **Get Freebuff authToken** → **Run workflow**
2. Optionally set `poll_timeout` (seconds to wait for authorisation, default 300) and `fingerprint` (leave blank to auto-generate)
3. You receive a login link on Telegram — open it and sign in with Google
4. Once the script polls the token, the full token is delivered to your Telegram (the Actions log only shows `***`)
5. Old runs are cleaned up automatically, keeping only the latest

> Without `TG_BOT_TOKEN` / `TG_CHAT_ID` configured, the workflow fails on its first step and never starts extraction.

### Option B: extract locally

```bash
cd freebuff_tools
python3 extract_freebuff.py login   # prints the auth URL, polls after you approve in the browser
python3 extract_freebuff.py show    # all accounts: email + token + status, one per line
python3 extract_freebuff.py tgsend  # test Telegram connectivity (when TG is configured)
```

Running `login` locally **appends per key** into `freebuff_tools/freebuff_credentials.json` (existing accounts are never overwritten; Google and GitHub sign-in are both recorded). The file is git-ignored and never committed; see `freebuff_tools/freebuff_credentials.example.json` for the shape.

Other useful commands:

```bash
python3 extract_freebuff.py export           # all account tokens, one per line, ready to paste into CF Workers vars
python3 extract_freebuff.py quota            # check usage
python3 extract_freebuff.py session          # open / inspect a session
python3 extract_freebuff.py chat "hello"     # send one message through the model API
```

> 💡 `show` probes each account with `GET /api/v1/freebuff/session` (**creates no session, costs nothing**) and reports every account at once: alive + quota / invalid token / banned / region-blocked / quota exhausted. For banned accounts the upstream returns `status: banned` on every endpoint. With multiple accounts, each line `export` prints can be pasted into the Cloudflare Worker variable `FREEBUFF_TOKEN` (newline separated).

## Admin panel `/admin`

Accounts, proxies, API key and quota are **all editable in the page, applied on save, no container restart**.

```
http://<server-ip>:8877/admin
```

(The port is the **left** side of `-p host:container` — every example in this repo uses **`8877`**; `8787` is only the container-internal port `server.js` listens on (`PORT` default) and is not exposed on the host.)

### Login and first password change

- **Initial password is the fixed string `admin`** (not randomised). It is also printed in the container log: `docker logs <container> | grep 初始密码`, and written to `credentials/.initial_password`
- **First login forces a change**: the API returns `must_change_password: true` and the page shows "you must change the initial password"; the new password must be **≥ 8 characters** or you cannot enter the panel
- The password is stored in `credentials/admin.json` as `sha256(salt:password)` with a random salt — **never in plaintext**
- Login sessions last **12 hours** (`session_ttl_ms`); 5 consecutive failures lock the panel for 15 minutes
- **Forgot the password**: delete `credentials/admin.json` and restart the container → the initial password is recreated with the forced change. If the account pool is empty, accounts are migrated from the legacy credential file / env (`migrateLegacyAccounts`)

### Card ①: account pool (one independent SOCKS5 egress per account)

Fields per row:

| Field | Description |
|---|---|
| Name | Free-form label (`acct-01`, `home-isp-2`, …) |
| Token (blank = keep current) | The Freebuff `authToken`; **leaving it blank keeps the stored value** |
| SOCKS5 egress | `socks5://user:pass@host:port`; **blank means direct** |
| Enabled | Checkbox; unchecking removes the account from rotation |

Four buttons per row:

| Button | Effect |
|---|---|
| **Test proxy** | Sends a request through that SOCKS5 egress and returns **exit IP + latency** (`✅ exit IP 1.2.3.4 (856ms)`) |
| **Test account** | Calls upstream `GET /api/v1/me`, returning `HTTP <status> <body>` — `account_suspended` shows up immediately |
| **Query quota** | Retrieves that account's Freebucks snapshot (below); **GET, zero cost** |
| **Delete** | Removes the row from the pool (click "Save accounts" to persist) |

Three buttons below the pool: **+ Add account**, **Save accounts** (persist + take effect immediately), **💰 Query all quotas** (per-account, rendered as a table).

### Card ②: service variables

| Variable | Effect |
|---|---|
| API Key (client auth) | The key clients must send to `/v1/chat/completions` |
| CODEBUFF_API (blank = official) | Upstream domain; blank means `https://www.codebuff.com` |
| RELAY_KEY | Relay key (needed when `CODEBUFF_API` points at an authenticated relay) |
| DEBUG | Toggle request-level debug logging |

Click **Save variables** to apply.

### Card ③: service status

Data from `GET /admin/api/status`, returning `version` (worker.js version), `token_pool_size`, `models_count` and `accounts` (name / proxy / enabled / masked token).

### Card ④: change admin password / log out

Current password + new password (≥ 8 characters).

### The quota buttons

- Per-row **Query quota** → `POST /admin/api/quota` (body: `{index, token?, socks5?}`)
- **💰 Query all quotas** → `GET /admin/api/quotas`

Both are **server-side**: they egress through that account's SOCKS5 and call upstream `GET /api/v1/freebuff/session` with `include-unused-rate-limits: 1`, returning:

```
ok / status / access_tier / status_field
balance / wallet_balance / plan_id
daily: { limit, spent, remaining, resetAt }
prices: { model: unit price }
price_notices / off_peak / price_changes
```

When banned it returns `banned: true` plus `error: account_suspended`, and the UI shows `❌ Account banned: …`.

Rendered example:

```
💰 Freebucks 20/100 (spent 80) · resets 2026-09-23 15:00 · tier=full · plan=xxx
Affordable calls: kimi-k3-eco×4 · glm-5.3-flash×4 · mimo-v2.5×2 · deepseek-v4-flash×1 · gpt-5.6-luna×1 · gemini-3.8-flash×0
```

"Affordable calls" is `floor(daily.remaining / prices[model])` — at a glance you see **which models you can still afford today** (expensive ones show ×0).

> ⏱ `resetAt` is **Pacific-time midnight**; `fmtReset()` renders it in the browser's local timezone (UTC+8 = **15:00 the next day**).

### Panel API (scriptable)

Everything lives under `/admin/api/` and is called with the `admin_session=<sid>` cookie after login:

| Path | Method | Description |
|---|---|---|
| `/admin/api/login` | POST | `{password}` → `Set-Cookie` + `must_change_password` |
| `/admin/api/logout` | POST | Destroy the session |
| `/admin/api/accounts` | POST | `{accounts:[...]}` replace the whole pool |
| `/admin/api/settings` | POST | `{settings:{api_key,codebuff_api,relay_key,debug}}` |
| `/admin/api/config` | GET | Read the current config |
| `/admin/api/status` | GET | Service status (version / model count / account table) |
| `/admin/api/test_proxy` | POST | `{socks5}` → exit IP + latency |
| `/admin/api/test_account` | POST | `{index,token,socks5}` → upstream `/me` response |
| `/admin/api/quota` | POST | `{index,token,socks5}` → one account's Freebucks |
| `/admin/api/quotas` | GET | Freebucks for every account |
| `/admin/api/change_password` | POST | `{old_password,new_password}` |

> ⚠️ The panel API shares the port with `/v1/*` but **only accepts the `admin_session` cookie**; `Authorization: Bearer *** works for `/v1/*` only, and the reverse is equally true.

---

## Deployment

### Docker deployment (recommended)

> Ideal for long-running local / NAS / VPS use: no Cloudflare Workers limits and **no CF edge markers** (`cf-worker` / `cf-ray`), so the ban risk is significantly lower than on CF. Images are published on **Docker Hub** — **no clone, no build**, one command to deploy.
>
> Image: `pingmike/freebuff2api:latest` ([Docker Hub page](https://hub.docker.com/r/pingmike/freebuff2api))

---

#### Option 1: one-line `docker run` (fastest)

> 💡 **Accounts do not have to come from environment variables**: once the container is up, open `http://<server-ip>:8877/admin` (initial password `admin`, forced change on first login), paste the token into the **account pool** and click **Save accounts** — **effective immediately, no restart**. The `freebuff_credentials.json` mount below is the legacy path and is only auto-migrated **when the panel account pool is empty**.
> Note: `-e FREEBUFF_API_KEY` is **overridden** by the panel's "service variables" (panel wins — see [Environment variables (real precedence)](#environment-variables-real-precedence)).

```bash
# 1. Prepare freebuff_credentials.json (multi-account aggregate format, see "Get authToken")
# Generate it with: python3 freebuff_tools/extract_freebuff.py login
# Or create it by hand: {"accounts": {"<account-id>": {"email": "...", "authToken": "...", "name": "..."}}}

# 2. Start it (inline the vars, or use a .env file)
docker run -d --name freebuff2api --restart unless-stopped \
  -p 8877:8787 \
  -e PORT=8787 \
  -e HOST=0.0.0.0 \
  -e FREEBUFF_API_KEY=your-api-key \
  -e RELAY_KEY= \
  -v "$(pwd)/freebuff_credentials.json:/app/credentials/freebuff_credentials.json:ro" \
  pingmike/freebuff2api:latest

# 3. Read the initial password and open the panel
docker logs freebuff2api | grep 初始密码
```

With many variables, use a `.env` file instead (`docker run --env-file .env`):

```bash
cat > .env <<'EOF'
PORT=8787
HOST=0.0.0.0
FREEBUFF_API_KEY=your-api-key
RELAY_KEY=
EOF

docker run -d --name freebuff2api --restart unless-stopped \
  -p 8877:8787 \
  --env-file .env \
  -v "$(pwd)/freebuff_credentials.json:/app/credentials/freebuff_credentials.json:ro" \
  pingmike/freebuff2api:latest
```

---

#### Option 2: docker compose (recommended for long-running)

```bash
# 1. One command: create dir → write compose → config .env → start
mkdir -p freebuff2api && cd freebuff2api && \
cat > docker-compose.yml <<'EOF'
services:
  freebuff2api:
    image: pingmike/freebuff2api:latest
    container_name: freebuff2api
    restart: unless-stopped
    ports:
      - "8877:8787"
    environment:
      - PORT=8787
      - HOST=0.0.0.0
      - FREEBUFF_API_KEY=${FREEBUFF_API_KEY}
      - RELAY_KEY=${RELAY_KEY}
    volumes:
      - ./freebuff_credentials.json:/app/credentials/freebuff_credentials.json:ro
EOF
echo 'FREEBUFF_API_KEY=your-api-key' > .env && \
docker compose pull && docker compose up -d
```

> 💡 `${FREEBUFF_API_KEY}` / `${RELAY_KEY}` in the compose file are read automatically from the `.env` next to it.

**Credential file:** drop it in before or after startup, then restart the container:

```bash
chmod 600 freebuff_credentials.json
# freebuff_credentials.json aggregate format: {"accounts": {"<account-id>": {"email": "...", "authToken": "...", "name": "..."}}}
docker compose restart          # or: docker restart freebuff2api
```

---

#### Updating (no rebuild, no re-pull)

The image uses a **container bootstrapper** design (inspired by [fscarmen/Argo-Nezha-Service-Container](https://github.com/fscarmen/Argo-Nezha-Service-Container)): on every start the container pulls the latest `worker.js` from GitHub raw (falling back to the copy baked into the image if the fetch fails). So after pushing a server-side `worker.js` change to GitHub, **restarting the main deployment updates it automatically**; clients only need to pull a new image occasionally:

```bash
docker compose pull && docker compose up -d   # pull new image and recreate
docker compose restart                        # or just restart (auto-pulls latest worker.js)
```

> 💡 **How to confirm what is actually running**: the image tag is decoupled from the code inside the container (the tag only reflects build time; the runtime code is whatever `worker.js` was pulled at startup), so **the image tag is not a version number**. Check the auth-free health endpoint instead:
>
> ```bash
> curl -s http://localhost:8877/healthz
> # the "version" field is the real worker.js version inside the container
> ```
>
> The returned `version` is the truth, regardless of what `docker images` shows. If you worry the container fell back to the bundled copy after a failed fetch, the same command tells you (e.g. image tag 1.7.0 but healthz returns `"version":"1.8.9"` means it auto-updated).

#### Environment variables (real precedence)

> ⚠️ **"Service variables" in the panel override `.env` / `-e`**, `buildEnv()` resolves in this order: **panel account pool > panel variables > `.env` > defaults**; `PORT` / `HOST` come from `.env` only (the panel has no such field).

| Variable | Single source of truth | Description |
|---|---|---|
| `FREEBUFF_TOKEN` | ✅ **panel account pool** | `-e FREEBUFF_TOKEN=...` is absorbed into the panel **only during the first migration** (`migrateLegacyAccounts`); afterwards `buildEnv` reads `getAccounts()` and **env changes have no effect** → manage accounts in the panel |
| `FREEBUFF_API_KEY` | panel first, then `.env`, then `freebuff-default-key` | **The client-facing key comes from the panel's `settings.api_key`** |
| `CODEBUFF_API` | panel first, then `.env` | Upstream URL, default empty = `https://www.codebuff.com` |
| `RELAY_KEY` | panel first, then `.env` | Relay key (required when `CODEBUFF_API` points at an authenticated relay) |
| `FREEBUFF_DEBUG` | ✅ **panel** `settings.debug` | The env is read only when `admin.json` is **first created**; afterwards it is ignored → change it in the panel |
| `PORT` / `HOST` | ✅ `.env` | Listen port / address, default `8787` / `0.0.0.0` |
| `FREEBUFF_SESSIONS_JSON` / `WEB_CREDENTIALS_FILE` | `.env` | Web-cookie auth channel |
| `RELAY_URL` / `RELAY_METHOD` / `RELAY_HEADERS` / `RELAY_BODY` | `.env` | Relay egress, triggered by `API_READY` |

**To make a panel variable fall back to `.env`** → clear that field in the panel and save; `buildEnv` falls back to the env value.

> ⚠️ Credentials accept two shapes: the multi-account aggregate `{"accounts": {...}}` (what the extractor writes) and the single-account top-level `authToken`. A legacy `freebuff_credentials.json` is auto-migrated **when the panel account pool is empty**; after migration the panel wins.

#### Maintainers: publish a new image to Docker Hub

The repo ships `.github/workflows/docker-publish.yml` (manual trigger, multi-arch amd64/arm64). Configure `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` under GitHub Secrets, then run **Run workflow** on the Actions page.

### Cloudflare Worker deployment (not recommended)

> **Freebuff now detects Cloudflare Worker deployments** (fingerprinting `cf-worker` / `cf-ray` edge markers, and naming proxy patterns like this project in its source). Deploying on CF significantly increases ban risk, **so it is not recommended as a primary deployment**; the steps below are kept only for users who understand the risk.

The worker is a **single file** (`worker.js`). If you still want to deploy on CF:

### Option A: paste the code in the CF console

Simplest and most controllable — no local tooling, no GitHub linkage:

1. Open [dash.cloudflare.com](https://dash.cloudflare.com) → **Workers & Pages** → **Create** → **Create Worker**
2. Name it anything (e.g. `freebuff2api`) and click **Deploy**
3. Open the Worker → **Edit code** → paste the **entire contents** of [worker.js](worker.js) over the default → **Deploy**
4. Go to **Settings → Variables and Secrets → Add**:

   | Type | Name | Value |
   |---|---|---|
   | Secret | `FREEBUFF_TOKEN` | your freebuff token (comma-separated for multiple accounts) |
   | Secret | `FREEBUFF_API_KEY` | custom access key (optional; defaults to `freebuff-default-key`) |

5. Verify after deployment:

   ```bash
   curl https://<worker>.<subdomain>.workers.dev/healthz          # health, no key needed
   curl https://<worker>.<subdomain>.workers.dev/v1/models \
     -H "Authorization: Bearer ***"                               # model list
   ```

> Each code change is just step 3 again: edit → paste → deploy. **GitHub auto-deploy is not recommended** (see below).
> ⚠️ **Versioning convention**: bump the version (the `version` field in healthz + the `X-Freebuff2api-Version` response header) before every deploy, otherwise you cannot tell whether production updated.

### Linking GitHub for auto-deploy (not recommended)

CF can connect a GitHub repo for auto-deploy, but **don't**:

- Every push goes straight to production; unverified local changes can hit live users
- You must configure build command / root directory, and helper files such as `freebuff_tools/` get pulled too
- Secrets and branch state become confusing, and incidents are harder to debug
- This repo contains a token extraction script, so auto-sync increases exposure

**Recommended**: edit locally → deploy with Docker on a VPS, or (if you accept the risk) paste into the CF console and deploy manually — fully under your control.

> Free models historically required a US egress IP; Cloudflare Workers exit from the US by default, so no extra configuration was needed.

### Custom domain

The default `https://<worker>.<subdomain>.workers.dev` may be unreachable in some regions (blocked / restricted). If `workers.dev` times out, bind your own domain:

1. **Add a custom domain**: CF console → your Worker → **Settings → Domains & Routes** → **Add** → **Custom domain**
2. Enter your domain (e.g. `api.example.com`); CF guides you through the DNS record (CNAME to `<worker>.<subdomain>.workers.dev`)
3. Wait for DNS (usually minutes); a free SSL certificate is issued automatically
4. Then use `https://api.example.com/v1` as the Base URL

> Requirement: the domain must be hosted on Cloudflare (or move its DNS to CF). Biding a custom domain just adds a second reachable path for restricted regions.

## Usage examples

```bash
# health check
curl https://<worker>.<subdomain>.workers.dev/healthz

# model list
curl https://<worker>.<subdomain>.workers.dev/v1/models \
  -H "Authorization: Bearer ***"

# non-streaming
curl https://<worker>.<subdomain>.workers.dev/v1/chat/completions \
  -H "Authorization: Bearer ***" -H "Content-Type: application/json" \
  -d '{"model":"deepseek/deepseek-v4-flash","messages":[{"role":"user","content":"hello"}]}'

# streaming
curl -N https://<worker>.<subdomain>.workers.dev/v1/chat/completions \
  -H "Authorization: Bearer ***" -H "Content-Type: application/json" \
  -d '{"model":"deepseek/deepseek-v4-flash","messages":[{"role":"user","content":"hello"}],"stream":true}'
```

## Model list

> Mapping source: Freebuff Desktop 0.0.51 (the official `FREEBUFF_ROOT_AGENT_ID_BY_MODEL` in `orchestrator.js`, synced 2026-08-07).
> The worker reaches upstream through Cloudflare Workers, which egress from the US by default and count as Freebuff full-access mode. **Quota is not a per-model allowlist but a single daily Freebucks wallet** — each model has its own **unit price**, charged when a session is created (see [💰 Freebucks quota model](#freebucks-quota-model)). Live list: `curl /v1/models` (currently 20).

| API model name | Session model | Upstream agentId | Notes |
|---|---|---|---|
| `deepseek/deepseek-v4-flash` | same | `base2-free-deepseek-flash` | primary recommendation |
| `mimo/mimo-v2.5` | same | `base2-free-mimo` | balanced |
| `minimax/minimax-m3` | same | `base2-free-minimax-m3` | - |
| `deepseek/deepseek-v4-pro` | same | `base2-free-deepseek` | - |
| `openai/gpt-5.6-luna` | same | `base2-free-luna` | - |
| `poolside/laguna-s-2.1` | same | `base2-free-laguna-s-2-1` | - |
| `openrouter/poolside/laguna-s-2.1` | same | `base2-free-laguna-s-2-1-openrouter` | - |
| `inclusionai/ling-3.0-flash:free` | same | `base2-free-ling-3-flash` | - |
| `crof/greg-2-ultra` | same | `base2-free-greg-2-ultra` | - |
| `crof/greg-2-super` | same | `base2-free-greg-2-super` | - |
| `meta/muse-spark-1.2-contributor` | same | `base2-free-muse-spark` | - |
| `z-ai/glm-5.2` | same | `base2-free-glm` | needs official eligibility (referral / streak) and uses a separate quota pool |
| `anthropic/claude-fable-5` | same | `base2-free-fable` | official capacity-limited trial, may only be open at certain hours |

> 📝 Measured notes (2026-08-08): `ling-3.0-flash:free` may return 404 upstream asking you to switch to a paid slug; `claude-fable-5` session creation may be rejected for free accounts (`session_model_mismatch`). These are upstream availability issues, not broken worker mappings.

## Multiple accounts

**Add accounts in the panel**: one row per account (name + token + its own SOCKS5 egress), click **Save accounts** to apply. On quota exhaustion (429 / empty response) the current account cools down and the next one takes over; each account has an **independent** allowance, so Freebucks wallets stack.

(The legacy comma-separated `FREEBUFF_TOKEN` env var is only absorbed into the panel during the **first migration**; afterwards the panel pool is authoritative.)

**Account selection strategy** (since v1.4.0):

1. Prefer an account with a **cached, still-active session** — a session lasts about an hour, and only creation costs quota;
2. Only rotate to the next account when no active cache exists.

This maximises Freebucks utilisation (see [💰 Freebucks quota model](#freebucks-quota-model)).

> Note: cooldown state lives in worker memory, resets on cold start and is not shared between concurrent instances. In everyday use this barely matters.

## Upstream gating

freebuff's free models cannot be called with a bare "token → chat" flow; they have a strict lifecycle:

```
session (open) → agent-runs (main + context-pruner child run) → chat/completions
```

- **session**: `POST /api/v1/freebuff/session` (with `x-freebuff-model`) returns an `instanceId`; it may queue.
- **agent-runs**: `START` the main agent (e.g. `base2-free-deepseek-flash`) plus a `context-pruner` child run, then `record_step` / `finish_run`. Chat validates that the run_id exists; without it you get a 4xx.
- **chat**: `POST /api/v1/chat/completions` with `codebuff_metadata.run_id`, `x-freebuff-instance-id`, the SDK user agent, `stop:['"cb_easp"']` and `provider.data_collection=deny`. **Upstream forces streaming**, so non-streaming requests are aggregated (timeout relaxed to 45s).

The worker handles the whole lifecycle automatically. Also: the system message must start with `You are Buffy, the strategic coding assistant.` (byte-level upstream validation) — the worker injects it for you.

### One account, one session (important)

A Freebuff account can only have **one client online at a time**. Therefore:

- ❌ Never probe quota/status via upstream `GET /api/v1/freebuff/session` inside `/v1/models` — that call occupies the session and kicks out an in-flight chat (428 `waiting_room_required`).
- ✅ `/v1/models` returns a **static list** (no upstream call).
- Upstream requests run through a **serial queue with a 300 ms gap** to avoid concurrency problems.

## Experience notes

Tested and working well:

1. **🌍 Direct US IP**: free models historically required a US egress; non-US IPs may fail. Cloudflare Workers exit from the US by default; local clients should use a US proxy.
2. **🤖 Hermes Agent (US VPS)**: run Hermes Agent on a US VPS.
3. **Local browser + page-assist extension**: works smoothly with [page-assist](https://github.com/n4ze3m/page-assist). Give it a try.

## Thanks

Thanks to the following contributors (in no particular order):

- [@yjzsg](https://github.com/yjzsg)
- [@zipei-a](https://github.com/zipei-a)
- [@hknerdr](https://github.com/hknerdr)

## Reference projects

This project learned from and references the following open-source projects:

- [freebuff2api](https://github.com/XxxXTeam/freebuff2api) — the original reverse-engineering of the freebuff desktop/API protocol (AGPL-3.0). This project is a derivative with further development and optimisation, and keeps the AGPL-3.0 licence.
- [freebuff](https://github.com/CodebuffAI/freebuff) — freebuff's official public source, studied for its protocol implementation and changelog.
- [Argo-Nezha-Service-Container](https://github.com/fscarmen/Argo-Nezha-Service-Container) — the **container bootstrapper** pattern (the Dockerfile only bootstraps; business logic is driven by remote scripts/code). This project's Docker deployment borrows that design for "edit code = update, restart = apply" lightweight management.

## Disclaimer

This project is for **technical exchange and learning/research only**.

- It proxies freebuff by reverse-engineering the desktop/API protocol, which **violates freebuff's terms of service (ToS)**.
- Using it carries the **risk of an account ban**, and bans are terminal and unrecoverable; proceed at your own risk.
- Do not use it commercially or for large-scale abuse; respect the freebuff operators.
- Users are responsible for complying with their local laws and freebuff's terms; the author is not liable for any account loss or dispute.

## License

Licensed under the [AGPL-3.0 License](LICENSE). This project references and reworks parts of [freebuff2api](https://github.com/XxxXTeam/freebuff2api) (itself AGPL-3.0), so it is released under AGPL-3.0 as well; keep the original copyright notices. Free to use, modify and share.