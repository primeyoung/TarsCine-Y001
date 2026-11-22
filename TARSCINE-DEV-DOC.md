# TarsCine-Y001 · 总开发文档（含追加与修正）

> **文档版本**：v3.1-full ｜ **生成时间**：2026-11-22 23:00
> **合并来源**：`AGENTS.md` v2.1（一期主线）+ `AGENTS_ADDENDUM.md` v2.6-Delta（二期追加）+ `docs/TARSCINE-DESIGN.md` v3.0-merged（实现态）+ 本轮全部实测修正
> **定位**：**唯一自包含交接文档**。接手者只读这一份即可掌握项目全貌、当前真实状态、以及「契约说的和代码实际做的哪里不一样」。
> **对应代码**：镜像 `tarscine-y001:latest`（265MB），前端 bundle `index-RAYodupv.js` / `index-chn6zVNt.css`
>
> **状态图例**：✅ 已实现并实测 ｜ 🚧 部分实现 / 已知断链 ｜ ⬜ 未实现 ｜ ⚠️ 契约与实现有差异

---

# 第 0 章 · 一页速览

TarsCine-Y001（命名取自《星际穿越》的 TARS）是一个**以 Web 为核心的多智能体影视自动化工作台**，跑在家用 NAS 上，把「产生观影念头 → 片单策展 → 资源搜源 → 下载调度 → 刮削归档 → 播放器就绪」整条链路自动化。

| 维度 | 事实 |
|---|---|
| 代码规模 | 后端 4,780 行 / 前端 4,596 行 ≈ **9,376 行** |
| 运行形态 | **单容器单端口**：FastAPI:8000 直接托管前端 `dist/`，无 CORS、无独立反代依赖 |
| 镜像 | `latest` **265MB**（core 8 包，已能跑 LLM）/ `llm` 721MB（仅非兼容协议需要） |
| 平台 | 终态 x86-64（飞牛OS / 自建 Linux）；极空间 Z2Pro(arm64) 仅过渡 |
| 已完成 | 一期 M1–M4 主线 + 模块 A + 模块 C + LLM 编排 + P0-4 实时推送 + 模块 G1 移动端 |
| 未完成 | 模块 D / E / F / B / G2 + P2 五项加固（详见 `TARSCINE-TODO.md`） |
| 真实联调 | ✅ 主链路已用真实 Prowlarr + qB + Aria2 + Jellyfin 跑通；自检 `passed=6 failed=0 skipped=1` |
| 累计修复缺陷 | **18 条**（全是「演示模式下看不出来」的静默失败） |

**三条不可动摇的结论**（踩过坑才写下的）：

1. **接口返回 200 不等于功能生效**。推理模型吃光 `max_tokens` → content 恒空 → 静默回退规则路由；qB 5.x 改名 `stop/start` → 暂停恒 4xx 但接口仍返回 `ok:true`。判断"有没有真生效"只能看可观测字段（`meta.llm.classify_source`）或真实外部系统状态。
2. **不伪造状态**。外部服务未配置 → `demo=True` / `skipped`；适配器失败 → 接口返回 `ok:false` 且不改库。任何"界面上看着成功"的假成功都算缺陷。
3. **契约里写的表结构有三张是不存在的**。以实现为准，迁移脚本对不存在的表直接跳过，禁止照文档建空表（详见第 10 章）。

---

# 第 1 章 · 文档体系与版本演进

```
AGENTS.md (v2.1)                 一期契约【冻结】
   │ M1–M4 主线：M1 骨架 / M2 对话流与机器人 / M3 下载归档 / M4 部署
   ▼
AGENTS_ADDENDUM.md (v2.6-Delta)  二期契约【冻结】
   │ 模块 A–G + NAS 轻量化红线 + 派发模板
   ▼
TARSCINE-DESIGN.md (v3.0-merged) 合并版总设计，日常派发用
   │ 契约文本 + 实现状态标注 + 差异清单
   ▼
TARSCINE-DEV-DOC.md (v3.1-full)  本文：再叠加实测修正、缺陷台账、环境坑
```

**为什么要三层而不是直接改原契约**：原始契约是"需求基线"，改它等于篡改历史，后面无法追溯"是需求变了还是实现走偏了"。所以契约冻结、差异单列（第 10 章）、缺陷单列（第 13 章）。

**对 AI 助手的派发方式（铁律）**：每次**只派发一个模块**，把 `TARSCINE-DEV-DOC.md` 一并给出。跨模块并行是本项目最容易产生回归的方式（例如改 SSE 协议会同时打断四条机器人链路）。

---

# 第 2 章 · 项目定位与部署目标

## 2.1 端侧覆盖

| 端 | 形态 | 状态 |
|---|---|:--:|
| Windows / macOS | 桌面三栏：左导航+机器人列表 / 中富媒体对话流 / 右任务管线与 NAS 监控抽屉 | ✅ |
| iOS / Android | PWA（manifest + Service Worker），可添加主屏全屏运行；<768px 切底部悬浮玻璃胶囊导航 | ✅ |

## 2.2 部署目标（2026-11-21 锁定）

- **终态**：飞牛 OS（fnOS）或自建 Linux，**x86-64**（参考 AMD Ryzen 7 H 255 / Zen4 8C16T）
- **过渡**：极空间 Z2Pro（RK3568 Cortex-A55 / 4GB / **arm64**）—— 仅过渡，arm64 支持已从「阻塞项」降为「可选」
- **形态**：单容器单端口，FastAPI 托管 `dist/`，规避 CORS 与反代配置

> 这条决策的由来：原本按极空间 arm64 规划，实测该机型 4GB 内存 + RK3568 算力无法承载后续 AI 能力，且官网规格已核实，故改锁定 x86-64。云服务器 amd64 部署即终态预演。

## 2.3 NAS 轻量化红线（原始契约第 1 章，不得破坏）

1. **主镜像严禁** PyTorch / 本地 Whisper 大模型 / Chromium 无头浏览器 → 重算力走外部 API 或 Sidecar
2. **核心表向后兼容**：不删列、不改名，新增必须带 `DEFAULT`
3. **协议锁定**：`<GlassSurface>` 视觉规范、z-0~z-50 层级、卡片协议与 `GenericActionCard` 的 `actions` 回调机制冻结
4. **主链路零侵入**：新机器人一律走 `plugins/` 自动注册，禁止在 TARS 核心路由硬编码 `if/else`

---

# 第 3 章 · 技术栈（锁定）与实测体积

## 3.1 前端

| 能力 | 选型 | 状态 |
|---|---|:--:|
| 构建 | Vite 6 | ✅ |
| 框架 | React 19 + TypeScript 严格模式 | ✅ |
| 样式 | Tailwind CSS v4（Lightning CSS 管线） | ✅ ⚠️见 §11.2 |
| 动效 | Framer Motion（spring 300/30） | ✅ |
| 图标 / 路由 / 状态 | Lucide / TanStack Router / Zustand | ✅ |

## 3.2 后端

| 能力 | 选型 | 状态 |
|---|---|:--:|
| 运行时 | Python 3.11 + FastAPI + Uvicorn | ✅ |
| 数据库 | SQLite + SQLModel，强制 `journal_mode=WAL` + `synchronous=NORMAL` | ✅ |
| 迁移 | 自研增量迁移 `core/migrations.py`，只 `ADD COLUMN ... DEFAULT` | ✅ |
| 编排 | **确定性规则路由默认** + LLM 结构化分类可选增强 | ✅ |
| 调度 | APScheduler + SQLite JobStore（`/config/jobs.db`），重启断点续传 | ✅ |
| 网络 | `httpx` 全异步 + SSE | ✅ |
| 文件名解析 | `guessit` | ✅ |

## 3.3 双镜像模式

`Dockerfile` 用 `ARG LLM_EXTRAS=core|llm` 切换依赖集：

| 镜像 | 内容 | 体积 | 实测 |
|---|---|---|:--:|
| `tarscine-y001:latest` | 核心 8 包，无 litellm | **265MB** | ✅ |
| `tarscine-y001:llm` | + litellm 1.110.2 与约 80 个依赖 | 721MB | ✅ |

> **重要结论**：填了 `llm.base_url` 就走 httpx 直连，**265MB 的 core 镜像已经能完整使用 LLM**。721MB 那个只在接入"非 OpenAI 兼容协议"时才需要。这条结论把交付体积砍掉了 63%。

---

# 第 4 章 · 仓库结构与代码资产

```text
TarsCine-Y001/
├── AGENTS.md                      # 一期契约 v2.1（冻结）
├── AGENTS_ADDENDUM.md             # 二期契约 v2.6-Delta（冻结）
├── docs/
│   ├── TARSCINE-DESIGN.md         # 合并版总设计 v3.0
│   ├── TARSCINE-DEV-DOC.md        # 【本文】总开发文档 v3.1
│   ├── TARSCINE-DELIVERABLES.md   # 产出物清单
│   ├── TARSCINE-TODO.md           # 后续待完成
│   └── TARSCINE-BACKLOG.md        # 待办 + 缺陷台账（历史）
├── Dockerfile / docker-compose.yml
├── deploy/
│   ├── cloud/                     # Caddy + Basic Auth + 导出/部署脚本
│   ├── lab/                       # 本地真实服务实验台
│   └── verify/                    # 28 张验收截图
├── frontend/src/
│   ├── pages/                     # Workbench / Agents / Settings
│   ├── components/
│   │   ├── glass/                 # GlassSurface / AmbientBackground
│   │   ├── chat/ + chat/cards/    # 对话流 + 五类卡片
│   │   ├── layout/                # AppShell / Sidebar / MobileTabBar / TaskPipeline
│   │   ├── onboarding/            # 模块 A
│   │   ├── settings/              # 模块 C RecommendMatrix
│   │   └── ui/                    # StatusBeacon / GlassButton / AgentIcon
│   ├── store/ lib/                # Zustand / api / types / hooks
│   └── globals.css                # CSS 变量与全局关键帧
└── backend/app/
    ├── agents/                    # base / registry / orchestrator / cards
    │   ├── builtin/               # curator / hunter / downloader / organizer
    │   └── plugins/               # 自动扫描目录
    ├── adapters/                  # qBittorrent / Aria2 / Jellyfin / Telegram
    ├── services/                  # llm / recommend / indexer / tmdb / net
    │                              # / archive / nfo / path_mapping
    │                              # / config_service / task_events
    ├── scheduler/engine.py
    ├── models/                    # 4 张真实表
    ├── api/routes/                # health / system / agents / chat / tasks
    │                              # / auto_lists / settings / images
    └── core/                      # config / database / migrations
```

> **顶层目录严禁随意更改**。文件级行数清单见 `TARSCINE-DELIVERABLES.md` 第 2 章。

---

# 第 5 章 · 核心架构

## 5.1 TARS 总控（Orchestrator）

TARS **不是注册机器人**，而是消息入口的调度器：

1. 解析 `@Tag` 显式指定（`classify_source=explicit`）
2. 未指定时按**确定性关键词规则**路由（默认路径，零依赖零延迟，`classify_source=rules`）
3. 可选 LLM 结构化分类增强（`classify_source=llm`）；**未配置/未安装/调用失败一律回退规则**
4. 处理卡片按钮回调 `POST /api/chat/action`，实现「点卡片 → 自动接力下一步」

**三类来源必须可观测**：`meta.llm.classify_source` ∈ `explicit | llm | rules`。这是判断"LLM 到底有没有生效"的唯一可靠依据，不要靠看回复内容猜。

## 5.2 四大内置机器人

| 机器人 | id / tag | 能力 | 输出卡片 |
|---|---|---|---|
| 片单策展 | `curator` / `@CuratorBot` | 模糊语义 → TMDB 检索 → 过滤已入库 → 结合画像推荐；支持「今日/本周/本月推荐」 | `poster_grid` |
| 资源猎手 | `hunter` / `@HunterBot` | Prowlarr 聚合检索 → `guessit` 解析 → 按 `4K>1080p`、中文字幕、做种数打分 | `torrent_list` |
| 下载调度 | `downloader` / `@DownloaderBot` | 策略模式：qB（BT/PT，打 `TarsCine` 标签）/ Aria2（HTTP/网盘直链） | `download_progress` |
| 刮削归档 | `organizer` / `@OrganizerBot` | 路径映射 + `st_dev` 硬链接校验（跨区降级复制）+ NFO/海报 + 字幕转 UTF-8 + Jellyfin 刷新 | `media_ready` |

## 5.3 归档规范

```
/media/movies/电影名 (年份)/
    ├── 电影名 (年份) - 2160p.mkv
    ├── 电影名 (年份).nfo
    ├── poster.jpg
    ├── fanart.jpg
    └── 电影名 (年份).zh.srt        # UTF-8
```

硬链接优先（不破坏原文件做种）；`st_dev` 不同则降级复制并记警告。

## 5.4 扩展机制（**严禁前端硬编码机器人列表**）

`GET /api/agents` 统一返回 `{id, tag, name, description, icon, color_accent, enabled, type}`，`type ∈ builtin|plugin|custom`。左侧导航、`@` 联想、消息头全部从此取数 —— 新增/启停机器人**不需要改任何前端布局代码**。

| 路径 | 机制 | 适用 |
|---|---|---|
| 代码插件 | `agents/plugins/` 自动扫描 + `@register_agent` | 开发者 / AI 扩展 |
| 可视化零代码 | SQLite `custom_agents` + 设置中心表单（System Prompt / @Tag / Webhook） | 终端用户 |

## 5.5 卡片协议

| 类型 | 用途 | 状态 |
|---|---|:--:|
| `poster_grid` | 海报网格（评分/年份/简介/【选定并搜源】） | ✅ |
| `torrent_list` | 资源对比（分辨率/体积/做种/字幕/【下载】） | ✅ |
| `download_progress` | 实时进度（引擎/速度/进度条/剩余/暂停继续） | ✅ 已接 SSE |
| `media_ready` | 归档就绪（路径/播放器刷新状态） | ✅ |
| `generic_action_card` | 通用扩展：Markdown + `actions:[{label, callback_id, payload}]` | ✅ |
| `wrapped_report_card` | 年度报告（模块 D） | ⬜ |

扩展与自定义机器人**一律**复用 `generic_action_card`，保证视觉与交互一致。

---

# 第 6 章 · 数据层（以真实表为准）

## 6.1 真实存在的表（4 张）

| 表 | 用途 | 关键字段 |
|---|---|---|
| `custom_agents` | 用户自定义机器人 | id / tag / name / system_prompt / webhook_url / enabled |
| `download_tasks` | 下载任务与入库历史 | remote_id / engine / name / title / year / resolution / status / progress / message / source_path |
| `app_settings` | **KV 键值表**（非列式） | key / value |
| `auto_lists` | 榜单巡航（模块 C 新增） | name / source_type / media_type / min_rating / max_items_per_run / max_size_gb / cron_expr / enabled |
| `jobs.db` | APScheduler JobStore（独立文件） | — |

**迁移铁律**：只允许 `ADD COLUMN ... DEFAULT`；表不存在**直接跳过**；禁止为兼容文档误建空表。当前 `SCHEMA_VERSION = 2`。

**首启状态**：`is_initialized` 存在 `app_settings` 的 `system.*` KV 键，**不新增列、不 ALTER TABLE**。

## 6.2 REST 路由全清单

```text
# 系统
GET    /api/health                     # 健康 + 版本 + 调度状态 + agents 数
GET    /api/system/status              # 首启状态 / 平台 / 迁移摘要 / LLM 状态
POST   /api/system/initialize          # 标记首启引导完成
POST   /api/system/check-hardlink      # /downloads 与 /media 是否同分区 + 磁盘余量
POST   /api/system/test-connection     # 外部服务连通与延迟

# 机器人
GET    /api/agents                     # Registry 全量（前端唯一数据源）
GET    /api/agents/{agent_id}
PATCH  /api/agents/{agent_id}/enabled
GET|POST /api/custom-agents
PATCH|DELETE /api/custom-agents/{agent_id}

# 对话（SSE 冻结协议）
POST   /api/chat/stream
POST   /api/chat/action                # 卡片按钮回调

# 任务
GET    /api/tasks
POST   /api/tasks
POST   /api/tasks/{task_id}/pause
GET    /api/tasks/stream               # 独立 SSE：下载进度推送

# 推荐矩阵（模块 C）
GET    /api/recommend/profile          # 观影画像 + 磁盘状态
GET    /api/recommend/library          # 本地库存清单
GET    /api/recommend/period/{period}  # day / week / month
GET|POST /api/auto-lists
PATCH|DELETE /api/auto-lists/{list_id}
POST   /api/auto-lists/{list_id}/run   # 立即执行（强制熔断检查）

# 配置
GET|PUT /api/settings
POST   /api/settings/test
GET    /api/scheduler
GET    /api/images/tmdb                # 海报后端转发（绕过 IP 层封禁）
```

## 6.3 SSE 协议 A：对话流（**冻结，不得改动**）

```
data: {"type":"meta",  ...}   # 会话元信息（agents / llm 状态 / classify_source）
data: {"type":"token", ...}   # 流式文本增量
data: {"type":"card",  ...}   # 富媒体卡片（可多次）
data: {"type":"error", ...}   # 错误（明确降级原因，不静默）
data: {"type":"done",  ...}   # 结束
```

实测事件数：普通对话 18–26 个，盲盒 8 个，action 接力 22 个。

## 6.4 SSE 协议 B：下载进度 `GET /api/tasks/stream`（独立端点）

```
event: snapshot   data: {"tasks":[...]}   # 连接建立时全量快照，供首屏
event: task       data: {...DownloadTask} # 单任务增量帧，每次轮询写入后广播
: ping                                    # 15s 心跳（注释行，不产生事件）
```

设计要点（改之前必须读完）：

- 广播在**进程内**完成：`services/task_events.py` 用 `asyncio.Queue`（容量 32，满则丢最旧）。**前提是单 worker** —— 将来上 `--workers N` 必须换 Redis pub/sub。
- 慢客户端不能拖累调度器：队列满时丢最旧帧，绝不阻塞 `publish()`。
- 前端 `useTaskStream()` 指数退避 1s→30s；**服务端正常关闭时退避归零**（否则重连会越来越慢）。
- 卡片认领推送帧靠 `taskId` 精确匹配；缺失时才回退 `demo===false && name` 匹配。
- 发布必须在 Session 内取值，否则 `DetachedInstanceError`。

---

# 第 7 章 · 运维速查

## 7.1 启动方式

```bash
# 容器（宿主默认 8000，被占用改 TARSCINE_HTTP_PORT）
docker compose up -d --build

# 本地开发：后端 + 前端分离
backend/.venv/Scripts/python -m uvicorn app.main:app --port 8000
cd frontend && npm run dev            # 5173，proxy /api → 8000

# 单端口预演（验证交付形态）
TARSCINE_STATIC_DIR=frontend/dist uvicorn app.main:app --port 8001
```

## 7.2 挂载与关键环境变量

```yaml
ports: ["${TARSCINE_HTTP_PORT:-8000}:8000"]
environment:
  TARSCINE_MEDIA_DIR: /media          # 必须显式指定
```

> 不指定 `TARSCINE_MEDIA_DIR` 会默认成 `/app/data/media` → 归档落进容器可写层，**重启即丢**。

其余变量见 `.env.example`（LLM / TMDB / Prowlarr / qB / Aria2 / Jellyfin / 路径映射 / Telegram / 调度）：

| 组 | 变量 |
|---|---|
| LLM | `TARSCINE_LLM_{PROVIDER,MODEL,API_KEY,BASE_URL}`（也可在设置中心可视化配置） |
| 元数据/搜源 | `TARSCINE_METADATA_{TMDB_API_KEY,PROWLARR_URL,PROWLARR_API_KEY}` |
| 下载 | `TARSCINE_DOWNLOADER_{QB_URL,QB_USERNAME,QB_PASSWORD,ARIA2_RPC_URL,ARIA2_SECRET,DEFAULT_ENGINE,DOWNLOAD_DIR}` |
| 播放器 | `TARSCINE_PLAYER_{JELLYFIN_URL,JELLYFIN_API_KEY}` |
| 路径 | `TARSCINE_PATHS_{MEDIA_DIR,PATH_MAPPINGS}` |
| 通知/调度 | `TARSCINE_NOTIFY_TG_*` / `TARSCINE_SCHEDULER_{ENABLED,POLL_INTERVAL}` |

## 7.3 云主机部署（验证环境）

`deploy/cloud/` 含 `Caddyfile` + `docker-compose.yml` + `.env.example` + `export-image.sh` + `deploy.sh`：

- **HTTPS**：Caddy 反代 + Let's Encrypt 真证书；纯 IP 无 SNI 会 `000`，故用 `<IP>.sslip.io` + 双 issuer
- **鉴权**：Caddy `basic_auth`（⚠️ 仅边界防护，应用内无鉴权）
- **SSE 透传**：`reverse_proxy` 必须 `flush_interval -1`，否则流式被缓冲
- **证书持久化**：`caddy_data` 具名卷；`--force-recreate` 不清证书，不触发 ACME 限流
- **离线交付**：`docker save` → 59MB bundle

## 7.4 实验台（真实联调用）

`deploy/lab/` 起 Prowlarr(9696) + qBittorrent(8080) + Aria2(6800) + Jellyfin(8096)，数据根 `E:/TarsCineLab`。

- Prowlarr 容器内 `thepiratebay.org` 被 DNS 污染 → 需 `LAB_HTTP_PROXY` + `host.docker.internal:host-gateway`
- 索引器失败会被临时拉黑，网络恢复后需 `POST /api/v1/indexer/testall` 才重新可用
- 聚合搜索用 **GET**；POST 需带 `indexerIds`，否则 400

---

# 第 8 章 · 出网层（国内网络必需）

`api.themoviedb.org` 的 A 记录被污染成 Facebook/Twitter 的 IP，但**真实 IP 未被封**；`doh.pub`（腾讯 DoH）国内可直连。

`services/net.py` 统一接管所有外网请求，顺序固定：

```
① 配置了 network.https_proxy → 走代理
② 启用 network.doh_bypass → DoH 解析真实 IP → 用 IP + sni_hostname 建连（不依赖代理）
③ 直连兜底
```

实测：未配代理时 TMDB 0.9–1.5s 可用。内网服务（qB / Aria2 / Jellyfin / Prowlarr）一律 `trust_env=False`，避免被宿主机遗留的 `HTTP_PROXY` 带偏。

**海报是另一回事**：`image.tmdb.org` → Bunny CDN `143.244.50.x` 是 **IP 层被封**，DoH 无解。

- 开 `network.image_proxy=true` → 前端 `posterUrl` 变成本后端 `/api/images/tmdb`
- 后端有代理时带代理拉图；无代理该接口返回 502，前端 `PosterArt` 自动回落渐变占位（功能不受影响）

> **未闭环**：至今没有可用代理地址，海报仍靠渐变占位。这是当前唯一影响观感的外部依赖。

---

# 第 9 章 · LLM 编排层（两通道）

## 9.1 唯一入口

`services/llm.py` 是**唯一** LLM 调用入口，orchestrator / recommend / 自检全部走它。**禁止各处自行 `import litellm`。**

```python
def transport() -> str                 # "http-openai" | "litellm" | "none"
def completion_url(base_url: str) -> str   # 归一，自动补 /chat/completions
async def chat(messages, *, temperature=0, max_tokens=512, timeout=20) -> str
```

通道选择**配置驱动**，不看 provider 字段：

| 条件 | 通道 | 依赖 | 适用 |
|---|---|---|---|
| `llm.base_url` 非空 | **http-openai**（httpx 直连，`trust_env=False`） | 仅 httpx（core 已含） | 千帆 Token Plan / 百炼 / DeepSeek / 智谱 / 硅基流动 / Ollama |
| `llm.base_url` 为空 | litellm | 需 `LLM_EXTRAS=llm` 镜像 | 非 OpenAI 兼容协议 |

`base_url` 自动归一（各家粒度不同，统一补 `/chat/completions`）：

```
https://qianfan.baidubce.com/v2/tokenplan/personal  →  .../personal/chat/completions
https://api.deepseek.com/v1                          →  .../v1/chat/completions
```

## 9.2 推理模型的关键坑（真机踩过，勿回退）

`DeepSeek-V4.1-Flash` 这类模型先输出 `reasoning_content`（思维链）再给 `content`，**思维链也计入 `max_tokens`**。

原实现：分类 16 / 推荐语 80 / 自检 **1** → HTTP **200 但 content 恒为空** → 静默回退规则路由，看起来"配了 LLM 却毫无变化"。

现固定：分类 512、推荐语 512、自检 512；配置项 `llm.max_tokens` 默认 1024。空正文时 `llm.last_error()` 写明 reasoning 长度，并通过 `meta.llm.last_error` 暴露。

## 9.3 百度千帆 Token Plan 实测事实（避免重复踩坑）

| 项 | 事实 |
|---|---|
| 正确模型 ID | **`deepseek-v4.1-flash`**（全小写、无日期后缀） |
| 错误写法 | `DeepSeek-V4.1-Flash` / `DeepSeek-V4.1-Flash-1017` → **403** `token_plan_person_model_not_supported`（注意：此时认证**已通过**，是模型名错而非 Key 错） |
| 端点 | 只能打 `.../v2/tokenplan/personal/chat/completions`；打标准 `/v2/chat/completions` → **401** `token_plan_person_api_key_not_allowed` |
| 返回结构 | 含 `reasoning_content`；`usage.completion_tokens_details.reasoning_tokens` 可查思维链消耗 |
| 延迟 | 单次分类 2.2–3.0s（推理模型固有），ping 1.15s |

> ⚠️ **合规提醒**：阿里云 / 百度两家 Token Plan 个人版条款均写明「仅限订阅人本人在编程工具与智能体工具中使用，**禁止用于自动化脚本、自定义应用后端或非交互式批量调用**」。TarsCine 后端属"自定义应用后端"，用订阅额度属违规。自用调试可接受，**正式部署建议换按量付费 Key**（单次约 600 in / 80 out，成本 ≈ ¥0.0006/次，月均约 ¥2）。

## 9.4 可观测字段

| 字段 | 位置 | 含义 |
|---|---|---|
| `meta.llm.classify_source` | SSE `meta` | `explicit` / `llm` / `rules` |
| `meta.llm.transport` | SSE `meta` | `http-openai` / `litellm` / `none` |
| `meta.llm.last_error` | SSE `meta` | 失败原因（含 reasoning 字符数） |
| `llm_backend_status()` | `GET /api/system/status` | 聚合状态，`engine` 可能是 `llm+rules` |

---

# 第 10 章 · 契约与实现的差异清单（必读）

原始契约撰写时预设的部分结构**从未落地**。以实现为准，**不要照文档去建表或改代码**。

| # | 契约描述 | 实际实现 | 处理原则 |
|---|---|---|---|
| 1 | 表：`sessions`、`messages`、`media_tasks` | 实际为 `custom_agents`、`download_tasks`、`app_settings`(KV)、`auto_lists` | 迁移脚本对不存在的表**直接跳过**，禁止误建空表 |
| 2 | `settings` 表新增 `is_initialized` 列 | 存在 `app_settings` 的 `system.*` KV 键 | **不 ALTER TABLE** |
| 3 | 编排用 PydanticAI 或 LiteLLM | 规则路由兜底；LLM 增强走 `services/llm.py` 双通道 | 未配置/失败一律回退规则，不静默 |
| 4 | 模块 C 画像读 `media_tasks` | 读 `download_tasks` 已入库记录 | 语义等价，直接复用 |
| 5 | 多架构 `amd64` + `arm64` | 仅 `amd64` 实测 | 终态锁定 x86-64，arm64 降为可选 |
| 6 | 主题自定义模块 | CSS 变量入口已预留，UI 未做 | 无阻塞，P2 |
| 7 | 契约要求"4 种内置卡片协议冻结" | 实际 **5 种**（多加 `generic_action_card`） | 是追加非破坏，向后兼容 |
| 8 | 模块 A「`/help` 走对话流」 | **纯前端本地生成**，不进 SSE、不改 orchestrator | 有意为之，符合"零侵入"红线 |
| 9 | 模块 G 是单一模块 | 拆成 G1（移动端，✅）与 G2（追剧洗版，⬜） | G1 收益快、风险低，先独立交付 |
| 10 | 段位提示：`deploy.sh` 里"镜像 59MB" | 59MB 是 **bundle 压缩包**，镜像解压后 265MB | 对外表述以镜像实际体积为准 |

---

# 第 11 章 · 视觉规范（Apple Liquid Glass）

## 11.1 空间底层与玻璃组件

- 底层渐变 `#070A12 → #0D121F`，**禁止死板纯黑**；内置缓慢流动的 Ambient Orbs 弥散光晕
- `<GlassSurface>` 统一参数：`rgba(255,255,255,0.06)` / `blur(24px) saturate(180%)` / `1px solid rgba(255,255,255,0.14)` / 内高光 + 深投影
- 所有面板、侧栏、气泡、卡片**必须复用**该组件；参数走 `globals.css` 的 CSS 变量（`--glass-bg` / `--glass-blur` / `--glass-saturate` / `--glass-border` / `--glass-shadow` / `--accent-primary`）
- 性能降级：`--glass-blur` 24px → 8px，背景不透明度提高；滚动列表高频卡片加 `will-change: transform`
- Framer Motion spring `{stiffness:300, damping:30}`；层级冻结 `z-0` 背景 / `z-10` 内容 / `z-20` 侧栏 / `z-30` 抽屉 / `z-40` 底部导航 / `z-50` 模态向导

## 11.2 ⚠️ backdrop-filter 构建陷阱（真机验证踩过）

> **只写标准 `backdrop-filter`，绝不手写 `-webkit-backdrop-filter` 双写。**

Tailwind v4 的 Lightning CSS 会把双写**合并成只留 `-webkit-` 版本**，而现代 Chrome 已不认 `-webkit-backdrop-filter`（`CSS.supports` 实测 `false`）→ 毛玻璃模糊**静默失效**，只剩半透明底色 + 高光边框，肉眼极易误判为「已生效」。

**验证方式**：查 `dist/assets/*.css` 产物 + 浏览器 `getComputedStyle().backdropFilter`（修复前 `none`，修复后 `blur(24px) saturate(1.8)`，实测 19 个元素生效）。

## 11.3 ⚠️ Tailwind display 工具类优先级

`StatusBeacon` 基类含 `inline-flex`，外部再传 `hidden` **不生效** —— Tailwind 的 display 工具类同优先级，谁生效取决于产物生成顺序，**与 class 书写顺序无关**。

**规则**：组件内部的 display 类一旦固定，显隐一律由**外层容器** `<span className="hidden lg:block">` 控制。

---

# 第 12 章 · 移动端适配（模块 G1，已交付）

手机 PWA 是主要使用场景，三条约束必须同时成立。

## 12.1 视口贴合（键盘不遮挡）

`index.html` 的 viewport 带 `interactive-widget=resizes-content`（Android Chrome 让布局视口随键盘收缩）。iOS Safari **不支持**该参数，只压缩 `visualViewport` —— 由 `lib/useViewport.ts::useViewportFit()` 兜底：

```
visualViewport.resize → --app-h = vv.height（仅 < 1024px 时写）
                      → html[data-kb="1"]（inset > 120px 判定键盘弹起）
```

- `#root { height: var(--app-h, 100%) }` —— 整壳跟可视视口收，输入框永远在键盘上方
- 键盘态：底部导航淡出（已被遮住）、`.composer-pad` 留白 96px → 8px
- `useVisualViewportHeight()` 供对话流做依赖：键盘收放后重新贴底

## 12.2 呼吸灯状态角标

`lib/useActivity.ts` 从 `backendTasks` 汇总 `downloading / seeding / paused`；`ui/StatusBeacon.tsx` 渲染呼吸光晕 + 计数（`--beacon` RGB 变量驱动 `beacon-halo` / `beacon-ring` 关键帧）。

展示位**收敛为两处**（原本做了三处，截图发现同屏重复）：移动端底部导航「管线」图标、桌面端左栏状态区。

> 字幕计数在模块 E 落地前**恒为 0 且不渲染** —— 不填假数字。

## 12.3 长会话分页与智能贴底

`ChatStream` 只挂最近 **50 条**，触顶自动补页（400ms 去抖）+ 顶部「加载更早的 N 条」按钮。补页前后用「距底距离」锚定（`anchorRef` + `useLayoutEffect`），滚动条不跳。

距底 < 96px 才跟随新内容；上滚读历史时不拽回，改出「回到最新」浮标；流式增量用 `auto` 避免逐 token 抖动。

## 12.4 ⚠️ 移动端验收必须用 CDP

Chrome 的 `--window-size=390,844 --screenshot` 会被**窗口最小宽度钳制**（实际布局 460+px），随后按请求尺寸裁剪 → 产出「元素被右边界裁掉」的**假截图**。曾据此误判为横向溢出。

正确做法：

```js
Emulation.setDeviceMetricsOverride({ width: 390, height: 844, deviceScaleFactor: 2, mobile: true })
→ Page.captureScreenshot
→ 判溢出：document.documentElement.scrollWidth === document.documentElement.clientWidth
```

Node 22 内置 WebSocket 可直连 CDP，**不需要 playwright**。本次实测 390/390，无横向溢出。

---

# 第 13 章 · 缺陷台账（18 条，全部已修）

全部是「演示模式下看不出来」的静默失败。**这张表的价值大于任何功能文档**：同类问题会重复出现。

## 13.1 主链路联调（7 条）

| # | 问题 | 影响 | 修复位置 |
|---|---|---|---|
| 1 | TMDB 域名被 DNS 污染，直连必超时 | 真实片单永远退化成演示数据 | `services/net.py` DoH + SNI 绕过 |
| 2 | `image.tmdb.org`（Bunny）IP 层被封 | 海报全部加载失败 | `/api/images/tmdb` 后端转发 + 前端渐变兜底 |
| 3 | qB **5.x** 登录返回 `204` 空响应，代码按 4.x 的 `"Ok."` 判断 | 登录永远失败 → 下载永远降级成演示卡 | `adapters/qbittorrent.py::_login` |
| 4 | 资源卡片不返回磁力链接 | 点【下载】只拿到片名，无法建任务 | `hunter.py` 卡片带 `magnet` + 前端透传 |
| 5 | 下载记录存磁力串而非 info-hash | 轮询永远匹配不上，进度卡 0% | `infohash_of()` + `_find_added()` |
| 6 | `source_path` 存下载目录 | 完成后找不到文件，自动归档形同虚设 | 轮询补 `content_path` / `files[0].path` |
| 7 | Jellyfin 12.x 不再接受 `X-Emby-Token` | 媒体库刷新一直 401 | `jellyfin.py::_headers` 改 MediaBrowser 授权头 |

## 13.2 LLM 接入（5 条）

| # | 问题 | 影响 | 修复位置 |
|---|---|---|---|
| 8 | `max_tokens` 分别 16 / 80 / **1** | 思维链吃光配额，HTTP 200 但 content 恒空 → 静默回退 | `services/llm.py` 统一 512 |
| 9 | 所有 LLM 调用硬依赖 litellm | core 镜像用不了 LLM，为一次分类被迫撑到 720MB | 新增 http-openai 通道，仅用 httpx |
| 10 | `is_secret()` 把 `max_tokens` 当密钥脱敏 | 前端显示 `***configured***` 无法编辑，保存可能写回脏值 | `_NON_SECRET_KEYS` 白名单 |
| 11 | 检索词清洗只剥开头指令词 | 「搜一下 The Matrix 的种子」→ 检索词含「的种子」→ Prowlarr 恒 0 | `hunter.py::clean_query()` 双向清洗 + @提及兜底 |
| 12 | 搜源失败文案合并「未配置」与「没搜到」 | 用户误以为 Key 写错，反复回去改配置 | 按 `demo_mode` / `last_error` 分三种提示 |

## 13.3 实时推送（4 条）

| # | 问题 | 影响 | 修复位置 |
|---|---|---|---|
| 13 | SSE 手写 `json.dumps` 不认 `datetime` | 首帧直接 500（`/api/tasks` 走 FastAPI 编码器所以从没暴露） | `tasks.py::_sse` 加 `default=str` |
| 14 | qB 5.x 把 `pause`/`resume` 改名 `stop`/`start` | 暂停恒 4xx，但接口仍返回 `ok:true` | `set_paused` 按新→旧顺序尝试 |
| 15 | 状态映射只认 `paused`，5.x 暂停态是 `stoppedDL` | 暂停后推送仍 `downloading` | `_map_state` 增 `stopped` 分支 |
| 16 | 暂停接口不看适配器结果，直接改库返 `ok:true` | 界面「已暂停」而下载器还在跑 —— **伪造状态** | 适配器失败返 `ok:false` 且不改库 |

## 13.4 移动端（2 条）

| # | 问题 | 影响 | 修复位置 |
|---|---|---|---|
| 17 | 卡片「暂停」只改本地状态，**从不调** `POST /api/tasks/{id}/pause` | 真实任务点暂停后 5 秒被 SSE 帧改回下载中 | `api.ts::setTaskPaused` + `store::toggleCardPause`（乐观更新 + 失败回滚 + pending） |
| 18 | 卡片统计行 `flex justify-between` 无换行 | 390px 下「速度/剩余/节点」被裁出卡片 | `DownloadProgressCard` 改 `flex-wrap` + 剩余 `truncate` |

## 13.5 另修 4 个体验/健壮性问题

- 搜源超时 10s 太短（聚合多索引器需 20s+）
- `@CuratorBot 片名` 把 `@提及` 带进检索词
- orchestrator 用硬编码「星际穿越」兜底片名（**伪造数据**）
- 速度固定 `:.1f MB/s` 把 20KB/s 显示成 `0.0 MB/s`（用户误以为卡死）→ `base.py::format_speed` 三档自适应
- 下载卡片硬编码「星际穿越」，且真实任务也提供「直接置为完成」按钮（造假）→ 改 `card.name` 并隐藏
- 任务 `name` 为空时把「创建下载任务：」当文件名 → 磁力 `dn` 参数兜底

---

# 第 14 章 · 工程红线（任何改动前必须确认）

1. **单容器单端口**：不引入 CORS、不引入独立反代依赖（云上 Caddy 属可选外挂）
2. **backdrop-filter 只写标准属性**，不手写 `-webkit-` 双写
3. **机器人列表严禁前端硬编码**，一律来自 `GET /api/agents`
4. **SSE 对话协议冻结**：`meta / token / card / error / done`；新推送另开端点
5. **表结构只增不改**：只 `ADD COLUMN DEFAULT`，不删列、不改名、不误建空表
6. **主镜像严禁** PyTorch / 本地 Whisper / Chromium，重算力走外部 API 或 Sidecar
7. **主链路零侵入**：新机器人走 `plugins/` 自动注册，禁止 orchestrator 硬编码 `if/else`
8. **不伪造成功**：未配置明确 `demo` / `skipped`；适配器失败不返回 `ok:true`
9. **路径白名单**：任何删除/移动 `/media` 文件的操作必须先校验路径前缀
10. **每次只派发一个模块**（AI 协作铁律），避免跨模块回归
11. **可观测优先**：新增分支必须带状态字段，便于"判断是否真生效"，不靠肉眼

---

# 第 15 章 · 环境与工具坑（本机实测）

| 类别 | 现象 | 处理 |
|---|---|---|
| Docker | Windows 10 IoT LTSC 2024 不在 Docker 官方支持列表；WSL 3.0.5 + Docker Desktop 4.100 / Engine 29.9.3 | 可用，但需注意下一条 |
| Docker | 由工具会话启动 Docker Desktop 会因黑名单拦截 `wsl.exe`，引擎随即退出 | **必须用户从桌面双击启动** |
| 网络 | Docker Hub 直连不通；daocloud ≈81KB/s、1ms.run ≈190KB/s | 配 3 个加速器；构建带 `NPM_REGISTRY=npmmirror` + `PIP_INDEX_URL=aliyun` |
| pip | 清华源 403 不可用 | 用阿里云源 |
| 构建 | BuildKit 按 digest 复用缓存 | 重建远快于首次，不必 `--no-cache` |
| 文件锁 | `vite build` 清 `dist/` 被 Windows 文件锁拦（8013 实例正 serve 该目录） | **构建前先停实例** |
| 验收 | Chrome `--window-size` 被最小宽度钳制，产出假裁剪截图 | 用 CDP `Emulation.setDeviceMetricsOverride` |
| 验收 | 截图看不出 `backdrop-filter` 是否生效（半透明底色 + 高光框看着像生效） | 读 `getComputedStyle().backdropFilter` |
| 网络 | Prowlarr 索引器失败后被临时拉黑，网络恢复仍搜不到 | `POST /api/v1/indexer/testall` 刷新 |
| 配置 | 配置误写入 `backend/data` 而非 `config` | 走 `PUT /api/settings`，不要直连 sqlite 写 |

---

# 第 16 章 · 当前交付状态

| 项 | 状态 |
|---|---|
| 一期 M1–M4 主线 | ✅ 全部完成并实测 |
| 模块 A 首启引导 | ✅ 后端 4 接口 + 四步玻璃向导 + 4 快捷胶囊 + `/help` 动态卡 |
| 模块 C 推荐矩阵 | ✅ 三档推荐 + 画像三档降级 + 防爆盘熔断 + 库存盲盒 |
| 模块 G1 移动端 | ✅ 呼吸灯角标 / 50 条分页 / 键盘不遮挡，390px 实测无溢出 |
| LLM 编排 | ✅ 千帆 `deepseek-v4.1-flash`，`classify_source=llm` 实证，`transport=http-openai` |
| 下载进度推送 | ✅ `/api/tasks/stream`，实测 `downloading→paused→paused→downloading` |
| 外部服务联调 | ✅ `passed=6 failed=0 skipped=1`（仅 Telegram 未配） |
| 镜像 | `latest` 265MB（**已能跑 LLM**）/ `llm` 721MB |
| 云主机 | 已部署模块 A 版；模块 C + LLM 版 bundle **待上传**（P0-1，用户自行操作） |
| 本地实验台 | `deploy/lab/` 主链路已真实跑通；**当前已按用户要求关闭** |
| 公网链路 | 401 鉴权 / Let's Encrypt 真证书 / 308→HTTPS / SSE 透传 全绿 |
| 版本控制 | ❌ **不是 git 仓库** —— 建议尽快 `git init` 建基线 |
| 后续待办 | 见 `docs/TARSCINE-TODO.md` |

---

# 第 17 章 · 变更日志

| 日期 | 内容 | 关键产出 |
|---|---|---|
| 2026-11-20 | 一期 M1–M2：骨架、玻璃视觉体系、对话流、四机器人、卡片协议 | 首个可跑的桌面三栏工作台 |
| 2026-11-21 上午 | M3–M4：下载归档链路、Docker 多阶段构建、单容器单端口 | 265MB 镜像 |
| 2026-11-21 下午 | 云主机部署：Caddy HTTPS + Basic Auth + 离线 bundle | 公网可访问 |
| 2026-11-21 晚 | 模块 A：增量迁移底座 + 4 自检接口 + 四步向导 + `/help` | 首启零门槛 |
| 2026-11-21 深夜 | 模块 C：画像三档降级 + 三档推荐 + 榜单巡航熔断 + 盲盒 | `auto_lists` 表、`/api/recommend/*` |
| 2026-11-22 上午 | **P0-3 真实服务联调**：lab 实验台 + 主链路首次真跑通 + 修 7 缺陷 | 硬链接 nlink=3、真实片单 |
| 2026-11-22 傍晚 | **LLM 接入**：`services/llm.py` 双通道 + 千帆实测 + 修 5 缺陷 | core 镜像即可用 LLM |
| 2026-11-22 深夜 | **P0-4 实时推送**：`task_events` + `/api/tasks/stream` + 修 4 缺陷 | 状态序列实测一致 |
| 2026-11-22 深夜 | **P1-5 模块 G1 移动端**：三个子项 + 修 2 缺陷 + CDP 验收 | 390px 无溢出 |

---

# 第 18 章 · 已知风险与未闭环项

| 风险 | 等级 | 说明 / 缓解 |
|---|:--:|---|
| 无版本控制 | **高** | 不是 git 仓库，改错无法回滚。建议 `git init` + 首次提交 + 忽略 `node_modules` / `.venv` / `dist` / `*.db` / `*.tar.gz` / `.env` |
| 海报 CDN 被封 | 中 | `image.tmdb.org` 在 IP 层被封，DoH 无解；需代理或自建反代。当前靠渐变占位，功能不受影响 |
| Token Plan 合规 | 中 | 个人版条款禁止后端自动化调用；正式部署换按量付费 Key（≈¥2/月） |
| 单 worker 假设 | 中 | `task_events` 是进程内队列；上 `--workers N` 必须换 Redis，否则推送错乱 |
| 应用内无鉴权 | 中 | 仅 Caddy Basic Auth 边界防护；P2-11 |
| PWA 离线不全 | 低 | Basic Auth 下 `/sw.js` `/manifest` 401；P2-12 |
| arm64 未实测 | 低 | 终态锁定 x86-64，过渡机不跑也无影响 |
| 无自动化测试 | 中 | 全靠人工端到端；18 条缺陷说明回归风险真实存在；P2-15 |
| 删除类操作 | **高（未实现）** | 模块 G2 涉及删文件，必须路径白名单 + 二次确认，单独派发 |

---

**文档结束。** 后续任务清单见 `TARSCINE-TODO.md`；文件与产物位置见 `TARSCINE-DELIVERABLES.md`。
