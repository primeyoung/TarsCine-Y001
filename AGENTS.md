# AGENTS.md — TarsCine-Y001 (NAS 影视多智能体 AI 助手) 开发总纲 v2.1

> **通用协作契约**：本文档为跨 AI 编程工具（Codex / Workbuddy / Cursor 等）的统一架构与设计规范。任何 AI 助手在生成或修改代码前，必须严格遵守以下技术栈锁定、目录结构与视觉红线，确保多工具接力开发时风格与架构高度一致。

---

## 1. 项目概述与端侧目标
**TarsCine-Y001**（灵感源自《星际穿越》TARS 机器人）是一款以 **Web 为核心** 的多智能体（Multi-Agent）影视自动化工作台，交互形态借鉴 OpenManus。用户可在统一对话流中与多个专职 AI 机器人协作，实现从“产生观影念头”到“NAS 播放器就绪”的全链路自动化。

* **全平台覆盖**：
  * **Windows / macOS**：桌面端三栏式空间布局（左侧导航与动态机器人列表、中间多智能体富媒体对话流、右侧任务管线与 NAS 监控抽屉）。
  * **iOS / Android**：支持 PWA（`manifest.json` + Service Worker），可“添加到主屏幕”全屏运行；移动端自动切换为底部悬浮玻璃胶囊导航栏（Floating Glass Tab Bar）。
* **部署规范**：
  * **单容器极简交付**：前端打包为静态资源直接由 FastAPI 托管，单端口对外服务，彻底规避跨域（CORS）与反向代理配置难题。
  * **多架构支持**：兼容 `linux/amd64` 与 `linux/arm64`，适配飞牛OS (fnOS)、极空间、群晖等 NAS 设备，统一挂载 `/config` 与 `/media`（推荐单一根目录挂载以保障硬链接同分区）。

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
  * 影视解析与网络：`httpx`（全异步请求）、`guessit` / `PTN`（种子文件名智能解析）、SSE（Server-Sent Events 流式输出）
* **容器化**：Docker Multi-stage Build（阶段一 Node 构建前端 `dist/`，阶段二 Python 合并静态文件打包）+ Docker Compose

### 2.2 标准仓库结构（严禁随意更改顶层目录）
```text
tarscine-y001/
├── AGENTS.md                 # 本架构与设计契约文档
├── Dockerfile                # 前后端合并多阶段构建镜像文件
├── docker-compose.yml        # NAS 部署编排文件
├── frontend/                 # Vite + React 前端工程
│   ├── public/               # PWA manifest.json 与图标资源
│   ├── src/
│   │   ├── pages/            # 主工作台、机器人管理与设置中心等页面视图
│   │   ├── components/
│   │   │   ├── glass/        # 核心液态玻璃基础组件 (GlassSurface, AmbientBg 等)
│   │   │   ├── chat/         # 对话流、四大专属卡片与通用扩展卡片 (GenericActionCard)
│   │   │   └── layout/       # 桌面三栏与移动端底部悬浮胶囊导航
│   │   ├── store/            # Zustand 状态管理 (对话流、动态 Agent 列表、任务队列、主题配置)
│   │   └── lib/              # API/SSE 请求封装、类型定义、主题 CSS 变量映射
├── backend/                  # FastAPI 后端工程
│   ├── app/
│   │   ├── agents/           # TARS 总控调度、注册中心 (registry.py) 与内置四大机器人
│   │   │   └── plugins/      # 开发者代码级扩展机器人自动扫描目录
│   │   ├── adapters/         # 外部工具适配器 (qBittorrent, Aria2, Jellyfin, Telegram, Webhook)
│   │   ├── services/         # NFO 元数据生成、硬链接归档、路径映射、字幕匹配
│   │   ├── scheduler/        # APScheduler 持久化后台轮询与自动接力引擎
│   │   ├── models/           # SQLite 数据模型 (含 custom_agents 表) 与配置 Schema
│   │   └── api/              # RESTful、SSE 流式接口、Agent 管理接口与前端静态文件挂载
```

---

## 3. 核心业务架构与 NAS 工程红线

### 3.1 四大内置专职 AI 机器人矩阵
系统由 **TARS 主控调度器（Orchestrator）** 与 **可独立 `@呼叫` 或自动接力的子机器人** 组成，关键节点支持“人工点击确认”或“全自动托管”双模式。以下为默认内置的 4 个核心机器人：

1. **片单推荐机器人 (`@CuratorBot`)**
   * **能力**：解析用户模糊语义，调用 TMDB API 获取影片信息，自动过滤已入库影片。
   * **对话流卡片**：渲染横向/网格电影海报玻璃卡片（`poster_grid`，含海报、评分、年份、简介、一键【选定并搜源】按钮）。

2. **资源获取机器人 (`@HunterBot`)**
   * **能力**：对接 Prowlarr / Jackett API 检索资源，结合 `guessit` 解析种子文件名，按预设策略（如 `4K/2160p > 1080p`、`含中文字幕`、`做种数 > 5`）打分排序。
   * **对话流卡片**：输出结构化资源对比列表卡片（`torrent_list`），供用户一键确认或自动放行。

3. **下载调度机器人 (`@DownloaderBot`)**
   * **双引擎适配器（策略模式）**：
     * `QBittorrentAdapter`：对接 qBittorrent WebUI API，负责 BT/PT 种子与磁力下载、打标签（`TarsCine`）及做种状态追踪。
     * `Aria2Adapter`：对接 Aria2 JSON-RPC API，负责网盘直链/HTTP 高速下载任务调度。
   * **持久化追踪**：交由 `APScheduler` 每 5 秒轮询进度并推送至前端，下载达 100% 时自动触发 `@OrganizerBot`。
   * **对话流卡片**：实时更新的下载进度玻璃卡片（`download_progress`，含引擎标识、实时速度、进度条、剩余时间、暂停/继续控制）。

4. **刮削管理机器人 (`@OrganizerBot`)**
   * **NAS 路径映射与硬链接防呆**：
     * 支持配置下载器远端路径到本容器路径的替换规则（`Path Mapping`）。
     * 执行硬链接前自动校验 `st_dev` 是否处于同一文件系统；若跨分区则记录警告并降级为复制模式。
   * **全播放器兼容归档策略**：
     * 硬链接归档至 `/media/movies` 标准目录（`电影名 (年份)/电影名 (年份) - 分辨率.mkv`），不破坏原文件做种。
     * **适配极空间影视 & 飞牛影视**：自动下载 `poster.jpg`、`fanart.jpg`，生成标准 Kodi/Emby 兼容的 **`.nfo` 元数据文件**与外挂中文字幕（自动转换编码为 UTF-8），确保国产 NAS 影视墙通过目录监控秒级精准刮削。
     * **适配 Jellyfin**：归档完成后主动调用 **Jellyfin API (`/Library/Refresh`)** 触发媒体库即时刷新。
   * **对话流卡片**：输出媒体库就绪卡片（`media_ready`，含海报、归档路径、播放器刷新状态）。
   * **通知预留 (`TelegramNotifier`)**：
     * 预留 Telegram Bot 接口（`TG_BOT_TOKEN` & `TG_CHAT_ID`），支持在资源待确认、下载完成、入库就绪时推送带海报的消息通知（预留 Inline Keyboard 回调接口）。

### 3.2 插件化与自定义机器人扩展机制 (Agent Registry)
为支持后期无缝新增专职机器人（如 `@SubtitleBot` 字幕精调、`@WashBot` 自动洗版、`@CinemaModeBot` 观影灯光联动）及用户自定义 Agent，系统必须遵循 **注册中心模式（Registry Pattern）**：

1. **动态元数据驱动 UI（严禁前端硬编码机器人列表）**：
   * 后端提供 `GET /api/agents` 接口，统一返回所有已激活机器人的元数据清单（`id`, `tag`, `name`, `description`, `icon`, `color_accent`, `enabled`, `type: builtin | plugin | custom`）。
   * 前端的左侧导航栏、输入框 `@` 联想弹窗、对话消息头均从该接口动态获取，新增或启用/禁用机器人无需修改任何前端布局代码。
2. **双轨制扩展路径**：
   * **代码插件扩展（开发者 / AI 编码扩展）**：`backend/app/agents/plugins/` 为自动扫描目录。任何继承 `BaseAgent` 并使用 `@register_agent` 装饰器的 Python 模块，在应用启动时自动注册进 `AgentRegistry` 与 TARS 总控路由表。
   * **可视化零代码扩展（用户自定义配置）**：SQLite `custom_agents` 表支持用户通过设置中心创建自定义机器人，支持配置专属 `System Prompt`、触发词（`@Tag`）及外部 **Webhook / HTTP Tool**（用于联动 HomeAssistant、自定义通知或外部脚本）。
3. **统一卡片输出协议（Card Protocol）**：
   * 内置机器人可指定专属前端卡片类型（`poster_grid`, `torrent_list`, `download_progress`, `media_ready`）。
   * 扩展与自定义机器人统一返回 `generic_action_card`（支持 Markdown 富文本 + 动态按钮列表 `actions: [{label, callback_id, payload}]`），复用 `<GlassSurface>` 渲染，保持视觉与交互闭环高度一致。

---

## 4. 视觉红线：Apple Liquid Glass（苹果液态玻璃风格）

所有涉及前端 UI 的任务必须严格执行以下设计规范：

1. **空间底层背景**：禁止使用死板纯黑。底层采用深邃流光渐变（`#070A12` 至 `#0D121F`），并内置缓慢流动的柔和弥散光晕（Ambient Orbs），以凸显前景玻璃的折射通透感。
2. **统一玻璃组件 (`<GlassSurface>`)**：所有面板、侧边栏、对话气泡、任务卡片必须复用此组件，核心参数：
   * `background: rgba(255, 255, 255, 0.06)`
   * `backdrop-filter: blur(24px) saturate(180%)`
   * `border: 1px solid rgba(255, 255, 255, 0.14)`
   * `box-shadow: inset 0 1px 0 0 rgba(255, 255, 255, 0.25), 0 20px 40px -15px rgba(0, 0, 0, 0.5)`
3. **GPU 渲染与低功耗性能降级**：
   * 滚动列表中的高频玻璃卡片必须添加 `will-change: transform` 开启独立 GPU 合成层。
   * 内置“性能模式 / 低功耗降级”开关（默认支持自动检测或手动切换）：开启后将全局 `--glass-blur` 从 `24px` 降至 `8px` 并提高背景不透明度，确保老旧核显与移动端 60fps 流畅滑动。
4. **动态微交互**：使用 Framer Motion 实现苹果标志性弹簧阻尼曲线（`type: "spring", stiffness: 300, damping: 30`）。
5. **主题自定义扩展预留**：所有颜色、透明度、模糊半径均抽离至 `globals.css` 的 CSS Variables（`--glass-bg`, `--glass-blur`, `--accent-primary`），为后期“主题自定义模块”预留统一入口。

---

## 5. 标准化分步开发里程碑 (Milestones)

切换不同 AI 工具开发时，请按以下顺序逐个里程碑推进：
* **Milestone 1（基础框架与液态玻璃组件）**：初始化 Vite 前端与 FastAPI 后端目录、配置 SQLite WAL 模式、搭建 `AgentRegistry` 基础接口框架、配置 CSS 主题变量、完成 `GlassSurface`（含性能降级）/ `AmbientBackground` 基础组件与 PWA 配置。
* **Milestone 2（多智能体对话工作台 UI）**：实现桌面三栏布局、移动端底部悬浮胶囊导航、动态 `@机器人` 快捷呼出菜单，以及 4 种专属富媒体卡片 + 1 种通用扩展卡片 `GenericActionCard`（先用 Mock 数据跑通完整前端交互闭环）。
* **Milestone 3（后端 Agent 引擎、插件加载与适配器）**：实现 PydanticAI/LiteLLM 意图编排、插件目录自动扫描与自定义机器人 CRUD、SSE 流式对话路由、`APScheduler` 后台下载监控任务，完成 `QBittorrentAdapter`、`Aria2Adapter`、`JellyfinAdapter`、`NFOGenerator`（含路径映射与硬链接检测）与 `TelegramNotifier` 模块。
* **Milestone 4（设置中心、机器人管理与单容器 Docker 打包）**：开发可视化设置页（LLM 模型配置、自定义机器人新建/Webhook 配置、路径映射规则、下载器/播放器配置、主题与玻璃性能预览）及前后端合并多阶段 `Dockerfile` / `docker-compose.yml`。
