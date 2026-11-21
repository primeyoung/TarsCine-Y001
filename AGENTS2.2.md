# AGENTS.md — TarsCine-Y001 (NAS 影视多智能体 AI 助手) 开发总纲 v2.2

> **通用协作契约**：本文档为跨 AI 编程工具（Codex / Workbuddy / Cursor 等）的统一架构与设计规范。任何 AI 助手在生成或修改代码前，必须严格遵守以下技术栈锁定、目录结构、工程安全红线与视觉规范，确保多工具接力开发时风格与架构高度一致。

---

## 1. 项目概述与端侧目标
**TarsCine-Y001**（灵感源自《星际穿越》TARS 机器人）是一款以 **Web 为核心** 的多智能体（Multi-Agent）影视自动化工作台，交互形态借鉴 OpenManus。用户可在统一对话流中与多个专职 AI 机器人协作，实现从“产生观影念头”到“NAS 播放器就绪”的全链路自动化。

* **全平台覆盖**：
  * **Windows / macOS**：桌面端三栏式空间布局（左侧导航、会话历史与动态机器人列表；中间多智能体富媒体对话流；右侧任务管线与 NAS 监控抽屉）。
  * **iOS / Android**：支持 PWA（`manifest.json` + Service Worker），可“添加到主屏幕”全屏运行；移动端自动切换为底部悬浮玻璃胶囊导航栏（Floating Glass Tab Bar）。
* **部署与 NAS 权限规范**：
  * **单容器极简交付**：前端打包为静态资源直接由 FastAPI 托管，单端口对外服务，彻底规避跨域（CORS）与反向代理配置难题。
  * **多架构与权限映射**：兼容 `linux/amd64` 与 `linux/arm64`，适配飞牛OS (fnOS)、极空间、群晖等 NAS 设备。容器入口脚本（`entrypoint.sh`）必须支持 `PUID`、`PGID`（默认 `1000:1000`）与 `TZ=Asia/Shanghai` 环境变量映射，确保生成的硬链接与 `.nfo` 文件与宿主机影视套件权限互通。
  * **存储挂载**：统一挂载 `/config` 与 `/media`（推荐单一根目录挂载以保障硬链接同分区）。

---

## 2. 技术栈与目录结构锁定

### 2.1 锁定技术栈
* **前端（纯 SPA 高性能架构）**：
  * Vite 6 + React 19 + TypeScript + Tailwind CSS + Framer Motion + Lucide Icons
  * 路由与状态：TanStack Router + Zustand（轻量全局状态与流式卡片状态管理）
* **后端（纯 Python 高稳定异步架构）**：
  * Python 3.11+ + FastAPI + Uvicorn
  * 数据库：**SQLite + SQLModel**（强制开启 `PRAGMA journal_mode=WAL;` 与 `synchronous=NORMAL` 以支持多 Agent 高并发读写不锁库）
  * AI 编排：**PydanticAI** 或 **LiteLLM**（强类型结构化输出，统一兼容 OpenAI / DeepSeek / 本地 Ollama 模型）
  * 任务调度：**APScheduler**（使用 SQLite JobStore 持久化后台监控任务，确保容器重启后下载进度追踪与自动刮削断点续传）
  * 影视解析与网络：`httpx`（全异步请求）、`guessit` / `PTN`（电影与剧集季/集智能解析）、SSE（Server-Sent Events 流式输出）
* **容器化**：Docker Multi-stage Build（阶段一 Node 构建前端 `dist/`，阶段二 Python 合并静态文件与 `entrypoint.sh` 打包）+ Docker Compose

### 2.2 标准仓库结构（严禁随意更改顶层目录）
```text
tarscine-y001/
├── AGENTS.md                 # 本架构与设计契约文档
├── Dockerfile                # 前后端合并多阶段构建镜像文件
├── entrypoint.sh             # PUID/PGID 权限动态映射与启动脚本
├── docker-compose.yml        # NAS 部署编排文件
├── frontend/                 # Vite + React 前端工程
│   ├── public/               # PWA manifest.json 与图标资源
│   ├── src/
│   │   ├── pages/            # 主工作台、设置与运维中心（含Agent管理/任务审计/日志）
│   │   ├── components/
│   │   │   ├── glass/        # 核心液态玻璃基础组件 (GlassSurface, AmbientBg 等)
│   │   │   ├── chat/         # 对话流、四大专属卡片与通用扩展卡片 (GenericActionCard)
│   │   │   └── layout/       # 桌面三栏与移动端底部悬浮胶囊导航
│   │   ├── store/            # Zustand 状态管理 (会话/对话流、动态 Agent 列表、任务队列、主题配置)
│   │   └── lib/              # API/SSE 请求封装、类型定义、Z-Index 规范与主题 CSS 变量映射
├── backend/                  # FastAPI 后端工程
│   ├── app/
│   │   ├── agents/           # TARS 总控调度、注册中心 (registry.py) 与内置四大机器人
│   │   │   └── plugins/      # 开发者代码级扩展机器人自动扫描目录
│   │   ├── adapters/         # 外部工具适配器 (qBittorrent, Aria2, Jellyfin, Telegram, Webhook)
│   │   ├── services/         # NFO 元数据生成(电影/剧集)、硬链接归档、路径映射与路径安全校验
│   │   ├── scheduler/        # APScheduler 持久化后台轮询与自动接力引擎
│   │   ├── models/           # SQLite 数据模型 (sessions, messages, media_tasks, custom_agents)
│   │   └── api/              # RESTful、SSE 流式接口、运维诊断接口、鉴权中间件与静态文件挂载
```

### 2.3 核心数据库表结构锁定 (`backend/app/models/`)
为保障多轮对话记忆、跨设备刷新卡片状态不丢失以及下载防重，SQLite 必须包含以下核心表：
1. `sessions`：会话元数据（`id`, `title`, `created_at`, `updated_at`）。
2. `messages`：对话消息与卡片快照（`id`, `session_id`, `role`, `agent_id`, `content`, `card_type`, `card_payload` [JSON], `action_state` [记录按钮是否已确认/禁用], `created_at`）。
3. `media_tasks`：全链路任务审计与幂等去重表（`id`, `tmdb_id`, `media_type` [movie|tv], `season`, `info_hash` [唯一索引防重复下载], `title`, `status` [searching|downloading|organizing|completed|failed], `downloader`, `save_path`, `target_path`, `error_msg`, `updated_at`）。
4. `custom_agents`：用户自定义零代码机器人配置表。

---

## 3. 核心业务架构与 NAS 工程红线

### 3.1 四大内置专职 AI 机器人矩阵
系统由 **TARS 主控调度器（Orchestrator）** 与 **可独立 `@呼叫` 或自动接力的子机器人** 组成，关键节点支持“人工点击确认”或“全自动托管”双模式。默认内置 4 个核心机器人：

1. **片单推荐机器人 (`@CuratorBot`)**
   * **能力**：解析用户模糊语义，区分电影与剧集，调用 TMDB API 获取详情与分季信息，自动过滤本地已入库资源。
   * **对话流卡片**：渲染横向/网格海报玻璃卡片（`poster_grid`，含海报、类型标识、评分、年份、简介、一键【选定并搜源】按钮）。

2. **资源获取机器人 (`@HunterBot`)**
   * **能力**：对接 Prowlarr / Jackett API 检索资源，结合 `guessit` 解析种子文件名（识别分辨率、编码、字幕组、剧集季/集范围），按预设策略（如 `4K/2160p > 1080p`、`含中文字幕`、`做种数 > 5`）打分排序。
   * **对话流卡片**：输出结构化资源对比列表卡片（`torrent_list`），支持按钮点击后持久化锁定状态（防止刷新后重复点击）。

3. **下载调度机器人 (`@DownloaderBot`)**
   * **双引擎适配器（策略模式）**：
     * `QBittorrentAdapter`：对接 qBittorrent WebUI API，负责 BT/PT 种子与磁力下载、打标签（`TarsCine`）及做种状态追踪。
     * `Aria2Adapter`：对接 Aria2 JSON-RPC API，负责网盘直链/HTTP 高速下载任务调度。
   * **幂等防重与持久化追踪**：派发前校验 `media_tasks` 表 `info_hash` 是否已存在；交由 `APScheduler` 每 5 秒轮询进度并推送至前端，下载达 100% 时自动触发 `@OrganizerBot`。
   * **对话流卡片**：实时更新的下载进度玻璃卡片（`download_progress`，含引擎标识、实时速度、进度条、剩余时间、暂停/继续控制）。

4. **刮削管理机器人 (`@OrganizerBot`)**
   * **NAS 路径映射与硬链接防呆**：
     * 支持配置下载器远端路径到本容器路径的替换规则（`Path Mapping`）。
     * 执行硬链接前自动校验 `st_dev` 是否处于同一文件系统；若跨分区则记录警告并降级为复制模式。
   * **全播放器兼容归档策略（电影 + 剧集双支持）**：
     * **电影归档**：硬链接至 `/media/movies/{电影名 (年份)}/{电影名 (年份)} - {分辨率}.{ext}`。
     * **剧集归档**：自动识别单集与全季打包种子，硬链接至 `/media/tv/{剧名 (年份)}/Season {XX}/{剧名 (年份)} - S{XX}E{YY} - {分辨率}.{ext}`。
     * **适配极空间影视 & 飞牛影视**：自动下载 `poster.jpg`、`fanart.jpg`（剧集额外下载 `seasonXX-poster.jpg`），生成标准 Kodi/Emby 兼容的 **`.nfo` 元数据文件**（`movie.nfo` / `tvshow.nfo` / 单集 `.nfo`）与外挂中文字幕（自动转换编码为 UTF-8），确保国产 NAS 影视墙秒级精准刮削。
     * **适配 Jellyfin**：归档完成后主动调用 **Jellyfin API (`/Library/Refresh`)** 触发媒体库即时刷新。
   * **对话流卡片**：输出媒体库就绪卡片（`media_ready`，含海报、归档路径、播放器刷新状态）。
   * **通知预留 (`TelegramNotifier`)**：预留 Telegram Bot 接口（`TG_BOT_TOKEN` & `TG_CHAT_ID`），支持推送带海报的消息通知与 Inline Keyboard 回调。

### 3.2 插件化与自定义机器人扩展机制 (Agent Registry)
1. **动态元数据驱动 UI（严禁前端硬编码机器人列表）**：
   * 后端提供 `GET /api/agents` 接口，统一返回所有已激活机器人的元数据清单（`id`, `tag`, `name`, `description`, `icon`, `color_accent`, `enabled`, `type: builtin | plugin | custom`）。
   * 前端的左侧栏、输入框 `@` 联想弹窗、对话消息头均从该接口动态获取。
2. **双轨制扩展路径**：
   * **代码插件扩展**：`backend/app/agents/plugins/` 为自动扫描目录。任何继承 `BaseAgent` 并使用 `@register_agent` 装饰器的 Python 模块，启动时自动注册进 `AgentRegistry`。
   * **可视化零代码扩展**：通过 `custom_agents` 表支持用户在设置中心创建自定义机器人，配置专属 `System Prompt`、触发词（`@Tag`）及外部 **Webhook / HTTP Tool**。
3. **统一卡片输出协议（Card Protocol）**：
   * 内置机器人可指定专属卡片（`poster_grid`, `torrent_list`, `download_progress`, `media_ready`）。
   * 扩展与自定义机器人统一返回 `generic_action_card`（支持 Markdown 富文本 + 动态按钮列表 `actions: [{label, callback_id, payload}]`），复用 `<GlassSurface>` 渲染。

### 3.3 轻量级运维面板与系统安全红线（替代独立后台）
不拆分子域名或独立 Admin 后台，所有运维与管理能力统一收口至前端 **“设置与运维中心”** 页面：
1. **任务审计与人工干预台**：展示 `media_tasks` 历史记录，支持筛选失败任务、一键触发“重新刮削/硬链接重试”以及清理异常记录。
2. **实时系统日志与诊断导出**：提供 `/api/system/logs` 实时日志流查看器（支持按 `INFO/WARN/ERROR` 过滤，方便排查路径映射错误或 Tracker 连通性问题）及一键导出脱敏诊断包。
3. **安全与权限防线**：
   * **可选全局访问锁**：支持在设置中心开启单密码 / PIN 码访问保护（签发 HttpOnly Cookie 或 JWT），默认关闭以保证内网开箱即用，公网穿透时一键开启防裸奔。
   * **接口密钥脱敏**：`GET /api/settings` 返回给前端的所有 API Key、下载器密码、Webhook Secret 必须做掩码处理（如 `sk-****1234`），仅在用户主动修改时覆盖写入。
   * **文件操作路径白名单**：任何文件移动、硬链接或清理操作，后端必须强制校验路径 `resolve()` 后位于配置的 `/media` 或下载映射根目录内，严防路径穿越漏洞。

---

## 4. 视觉红线：Apple Liquid Glass（苹果液态玻璃风格）

所有涉及前端 UI 的任务必须严格执行以下设计规范：

1. **空间底层背景**：禁止死板纯黑。底层采用深邃流光渐变（`#070A12` 至 `#0D121F`），内置缓慢流动的柔和弥散光晕（Ambient Orbs）。
2. **统一玻璃组件 (`<GlassSurface>`)**：所有面板、侧边栏、对话气泡、任务卡片必须复用此组件，核心参数：
   * `background: rgba(255, 255, 255, 0.06)`
   * `backdrop-filter: blur(24px) saturate(180%)`
   * `border: 1px solid rgba(255, 255, 255, 0.14)`
   * `box-shadow: inset 0 1px 0 0 rgba(255, 255, 255, 0.25), 0 20px 40px -15px rgba(0, 0, 0, 0.5)`
3. **全局 Z-Index 层级锁定（防玻璃堆叠上下文遮挡）**：
   * `z-0`：底层流动弥散背景 (`AmbientBackground`)
   * `z-10`：主对话流与富媒体卡片 (`ChatStream`)
   * `z-20`：桌面端固定侧边栏与右侧监控抽屉
   * `z-30`：移动端底部悬浮玻璃胶囊导航栏 (`FloatingGlassTabBar`)
   * `z-40`：输入框 `@机器人` 快捷联想浮窗 (`MentionPopover`)
   * `z-50`：全局模态框、Toast 提示与锁屏验证层
4. **GPU 渲染与低功耗性能降级**：
   * 滚动列表中的高频玻璃卡片必须添加 `will-change: transform` 开启独立 GPU 合成层。
   * 内置“性能模式 / 低功耗降级”开关：开启后将全局 `--glass-blur` 从 `24px` 降至 `8px` 并提高背景不透明度，确保老旧核显与移动端 60fps 流畅滑动。
5. **动态微交互与主题变量**：
   * 使用 Framer Motion 实现苹果标志性弹簧阻尼曲线（`type: "spring", stiffness: 300, damping: 30`）。
   * 所有颜色、透明度、模糊半径抽离至 `globals.css` 的 CSS Variables（`--glass-bg`, `--glass-blur`, `--accent-primary`）。

---

## 5. 标准化分步开发里程碑 (Milestones)

切换不同 AI 工具开发时，请严格按以下顺序逐个里程碑推进：
* **Milestone 1（基础框架、数据库与液态玻璃组件）**：初始化 Vite 前端与 FastAPI 后端目录、配置 SQLite WAL 模式及 4 张核心表（`sessions`, `messages`, `media_tasks`, `custom_agents`）、搭建 `AgentRegistry` 基础接口框架、锁定 Z-Index 与 CSS 主题变量、完成 `GlassSurface`（含性能降级）/ `AmbientBackground` 基础组件与 PWA 配置。
* **Milestone 2（多智能体对话工作台 UI）**：实现桌面三栏布局（含历史会话切换）、移动端底部悬浮胶囊导航、动态 `@机器人` 快捷呼出菜单，以及 4 种专属富媒体卡片（支持按钮锁定态）+ 1 种通用扩展卡片 `GenericActionCard`（先用 Mock 数据跑通完整前端交互闭环）。
* **Milestone 3（后端 Agent 引擎、剧集/电影刮削与适配器）**：实现 PydanticAI/LiteLLM 意图编排、插件目录自动扫描与自定义机器人 CRUD、SSE 流式对话路由、`APScheduler` 后台下载监控与幂等防重任务，完成 `QBittorrentAdapter`、`Aria2Adapter`、`JellyfinAdapter`、`NFOGenerator`（电影 + 剧集分季命名、路径映射与路径白名单校验）与 `TelegramNotifier` 模块。
* **Milestone 4（设置与运维中心、安全鉴权及单容器 Docker 打包）**：开发可视化设置与运维页（LLM/下载器/播放器配置与密钥脱敏、自定义机器人配置、任务审计与重试台、实时系统日志查看器、全局访问锁开关、主题与玻璃性能预览），编写 `entrypoint.sh`（PUID/PGID 映射）及前后端合并多阶段 `Dockerfile` / `docker-compose.yml`。
