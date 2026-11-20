# AGENTS.md — TarsCine-Y001 (NAS 影视多智能体 AI 助手) 开发总纲

> **通用协作契约**：本文档为跨 AI 编程工具（Codex / Workbuddy / Cursor 等）的统一架构与设计规范。任何 AI 助手在生成或修改代码前，必须严格遵守以下技术栈锁定、目录结构与视觉红线，确保多工具接力开发时风格与架构高度一致。

---

## 1. 项目概述与端侧目标
**TarsCine-Y001**（灵感源自《星际穿越》TARS 机器人）是一款以 **Web 为核心** 的多智能体（Multi-Agent）影视自动化工作台，交互形态借鉴 OpenManus。用户可在统一对话流中与多个专职 AI 机器人协作，实现从“产生观影念头”到“NAS 播放器就绪”的全链路自动化。

* **全平台覆盖**：
  * **Windows / macOS**：桌面端三栏式空间布局（左侧导航与机器人列表、中间多智能体富媒体对话流、右侧任务管线与 NAS 监控抽屉）。
  * **iOS / Android**：支持 PWA（`manifest.json` + Service Worker），可“添加到主屏幕”全屏运行；移动端自动切换为底部悬浮玻璃胶囊导航栏（Floating Glass Tab Bar）。
* **部署规范**：支持 Docker 一键部署（兼容 `linux/amd64` 与 `linux/arm64`），适配飞牛OS (fnOS)、极空间、群晖等 NAS 设备，挂载 `/config` 与 `/media` 目录。

---

## 2. 技术栈与目录结构锁定

### 2.1 技术栈
* **前端**：Next.js 15 (App Router) + React 19 + TypeScript + Tailwind CSS + Framer Motion + Lucide Icons
* **后端**：Python 3.11+ + FastAPI + SQLite (SQLModel) + httpx (异步请求) + SSE (Server-Sent Events 流式输出)
* **容器化**：Docker + Docker Compose

### 2.2 标准仓库结构（严禁随意更改顶层目录）
```text
tarscine-y001/
├── AGENTS.md                 # 本架构与设计契约文档
├── docker-compose.yml        # NAS 部署编排文件
├── frontend/                 # Next.js 前端工程
│   ├── src/
│   │   ├── app/              # 页面路由与 globals.css (主题 CSS 变量)
│   │   ├── components/
│   │   │   ├── glass/        # 核心液态玻璃基础组件 (GlassSurface, AmbientBg 等)
│   │   │   ├── chat/         # 对话流与四大机器人专属富媒体卡片
│   │   │   └── layout/       # 桌面三栏与移动端底部悬浮胶囊导航
│   │   └── lib/              # API 请求封装、类型定义、主题配置
├── backend/                  # FastAPI 后端工程
│   ├── app/
│   │   ├── agents/           # 四大机器人 Agent 逻辑与总控路由
│   │   ├── adapters/         # 外部工具适配器 (qBittorrent, Aria2, Jellyfin, Telegram)
│   │   ├── services/         # NFO 元数据生成、硬链接归档、字幕匹配
│   │   ├── models/           # SQLite 数据模型与配置 Schema
│   │   └── api/              # RESTful 与 SSE 流式接口
```

---

## 3. 核心业务架构：四大专职 AI 机器人矩阵

系统由 **TARS 主控调度器（Orchestrator）** 与 **4 个可独立 `@呼叫` 或自动接力的子机器人** 组成，关键节点支持“人工点击确认”或“全自动托管”双模式：

1. **片单推荐机器人 (`@CuratorBot`)**
   * **能力**：解析用户模糊语义，调用 TMDB API 获取影片信息。
   * **对话流卡片**：渲染横向/网格电影海报玻璃卡片（含海报、评分、年份、简介、一键【选定并搜源】按钮）。

2. **资源获取机器人 (`@HunterBot`)**
   * **能力**：对接 Prowlarr / Jackett API 检索资源，按预设策略（如 `4K/2160p > 1080p`、`含中文字幕`、`做种数 > 5`）解析种子文件名并打分排序。
   * **对话流卡片**：输出结构化资源对比列表卡片，供用户一键确认或自动放行。

3. **下载调度机器人 (`@DownloaderBot`)**
   * **双引擎适配器（策略模式）**：
     * `QBittorrentAdapter`：对接 qBittorrent WebUI API，负责 BT/PT 种子与磁力下载、打标签（`TarsCine`）及做种状态追踪。
     * `Aria2Adapter`：对接 Aria2 JSON-RPC API，负责网盘直链/HTTP 高速下载任务调度。
   * **对话流卡片**：实时更新的下载进度玻璃卡片（引擎标识、实时速度、进度条、剩余时间、暂停/继续控制）。

4. **刮削管理机器人 (`@OrganizerBot`)**
   * **全播放器兼容归档策略**：
     * 下载完成后通过**硬链接（Hardlink）**归档至 `/media/movies` 标准目录（`电影名 (年份)/电影名 (年份) - 分辨率.mkv`），不破坏原文件做种。
     * **适配极空间影视 & 飞牛影视**：自动下载 `poster.jpg`、`fanart.jpg`，生成标准 Kodi/Emby 兼容的 **`.nfo` 元数据文件**与外挂中文字幕，确保国产 NAS 影视墙通过目录监控秒级精准刮削。
     * **适配 Jellyfin**：归档完成后主动调用 **Jellyfin API (`/Library/Refresh`)** 触发媒体库即时刷新。
   * **通知预留 (`TelegramNotifier`)**：
     * 预留 Telegram Bot 接口（`TG_BOT_TOKEN` & `TG_CHAT_ID`），支持在资源待确认、下载完成、入库就绪时推送带海报的消息通知（预留 Inline Keyboard 回调接口）。

---

## 4. 视觉红线：Apple Liquid Glass（苹果液态玻璃风格）

所有涉及前端 UI 的任务必须严格执行以下设计规范：

1. **空间底层背景**：禁止使用死板纯黑。底层采用深邃流光渐变（`#070A12` 至 `#0D121F`），并内置缓慢流动的柔和弥散光晕（Ambient Orbs），以凸显前景玻璃的折射通透感。
2. **统一玻璃组件 (`<GlassSurface>`)**：所有面板、侧边栏、对话气泡、任务卡片必须复用此组件，核心参数：
   * `background: rgba(255, 255, 255, 0.06)`
   * `backdrop-filter: blur(24px) saturate(180%)`
   * `border: 1px solid rgba(255, 255, 255, 0.14)`
   * `box-shadow: inset 0 1px 0 0 rgba(255, 255, 255, 0.25), 0 20px 40px -15px rgba(0, 0, 0, 0.5)`
3. **动态微交互**：使用 Framer Motion 实现苹果标志性弹簧阻尼曲线（`type: "spring", stiffness: 300, damping: 30`）。
4. **主题自定义扩展预留**：所有颜色、透明度、模糊半径均抽离至 `globals.css` 的 CSS Variables（`--glass-bg`, `--glass-blur`, `--accent-primary`），为后期“主题自定义模块”预留统一入口。

---

## 5. 标准化分步开发里程碑 (Milestones)

切换不同 AI 工具开发时，请按以下顺序逐个里程碑推进：
* **Milestone 1（基础框架与液态玻璃组件）**：初始化前后端目录、配置 CSS 主题变量、完成 `GlassSurface` / `AmbientBackground` 基础组件与 PWA 配置。
* **Milestone 2（多智能体对话工作台 UI）**：实现桌面三栏布局、移动端底部悬浮胶囊导航、`@机器人` 快捷呼出菜单，以及 4 种富媒体交互卡片（先用 Mock 数据跑通完整前端交互闭环）。
* **Milestone 3（后端 Agent 引擎与适配器）**：实现 FastAPI + SQLite 配置存储、SSE 流式对话路由，完成 `QBittorrentAdapter`、`Aria2Adapter`、`JellyfinAdapter`、`NFOGenerator` 与 `TelegramNotifier` 模块。
* **Milestone 4（设置中心与 NAS Docker 打包）**：开发可视化设置页（API Key、下载器/播放器配置、主题调节预览）及多架构 `Dockerfile` / `docker-compose.yml`。
