# 🎬 TarsCine-Y001
**Your Multi-Agent AI Cinema Concierge for NAS**  
*灵感源自《星际穿越》TARS 机器人 × Apple Liquid Glass 空间视觉美学*

## ✨ 项目简介
**TarsCine-Y001** 是一款专为私人 NAS 打造的**跨端多智能体（Multi-Agent）影视自动化管家**。其融合了 **OpenManus** 的多智能体协作机制与 **Apple Liquid Glass** 空间视觉设计，支持在统一的对话框中通过四大内置 AI 机器人（支持插件扩展）完成“意图推荐 → 全网搜源 → 调度下载 → 硬链接刮削入库”的全流程。

## 🏗️ 系统架构与工作流
系统采用前后端合并单容器交付，通过 SSE（Server-Sent Events）实现多智能体协作状态的实时流式推送。核心架构包含：
* **跨端交互层**：Apple Liquid Glass UI / PWA 统一对话工作台。
* **单容器后端**：FastAPI + Vite SPA，集成 TARS 主控调度器、Agent Registry 矩阵及 SQLite (WAL) 状态库。
* **专职智能体矩阵**：
  * `@CuratorBot`（片单推荐官）对接 TMDB API。
  * `@HunterBot`（资源猎手）对接 Prowlarr / Jackett。
  * `@DownloaderBot`（下载调度员）对接 qBittorrent / Aria2。
  * `@OrganizerBot`（刮削归档管家）支持硬链接与极空间、飞牛、Jellyfin 联动。

*(详细的架构图、全自动观影协作时序图及界面预览效果可参考原文档中的 Mermaid 图表与占位图布局)*

## 🤖 核心特性
* **四大内置机器人与自定义扩展**：支持零代码可视化新建专属 `@自定义机器人`，或在 `plugins/` 目录通过 `@register_agent` 装饰器挂载新技能。
* **Apple Liquid Glass 视觉体验**：深邃流光暗色背景、毛玻璃折射（`backdrop-blur`）、1px 高光边缘与动效，兼顾视觉与低功耗降级。
* **全平台覆盖与 PWA**：前后端单端口交付，移动端深度适配 iOS/Android PWA 全屏原生体验。
* **消息推送**：预留 Telegram Bot 通知支持。

## 🛠️ 技术栈与项目结构
* **前端**：Vite 6 · React 19 · TypeScript · TanStack Router · Zustand · Tailwind CSS
* **后端**：Python 3.11+ · FastAPI · PydanticAI/LiteLLM · SQLite (WAL) · APScheduler · SSE
* **部署**：Docker Multi-stage Build (`linux/amd64` & `linux/arm64`)

```text
tarscine-y001/
├── AGENTS.md                 # 跨 AI 编程工具统一架构与设计契约 (v2.1)
├── Dockerfile                # 前后端合并多阶段构建镜像文件
├── docker-compose.yml        # NAS Docker 一键部署模板
├── docs/                     # 界面展示截图与文档目录
├── frontend/                 # Vite + React 前端与 Liquid Glass 组件库
└── backend/                  # FastAPI 多智能体编排与适配器服务
```

## 🚀 NAS Docker 快速部署
> ⚠️ **硬链接规范提示**：请将下载目录与媒体库目录放在同一个宿主机父级目录下统一映射为 `/media`，以支持秒级硬链接。

1. **编辑 `docker-compose.yml`**：
```yaml
version: "3.8"
services:
  tarscine-y001:
    image: tarscine-y001:latest
    container_name: tarscine-y001
    restart: unless-stopped
    ports:
      - "8321:8000"
    environment:
      - TZ=Asia/Shanghai
      - PUID=1000
      - PGID=1000
    volumes:
      - ./config:/config
      - /path/to/your/nas/Media:/media
```

2. **启动与访问**：
   * 执行 `docker-compose up -d` 启动容器。
   * 电脑端访问 `http://<你的NAS_IP>:8321`，移动端通过浏览器“添加到主屏幕”体验 PWA 模式。

## 🗺️ 开发路线图 & 开源协议
* 推进从基础脚手架、多端布局、大模型路由到可视化设置面板的 4 个 Milestone 演进。
* 本项目基于 [MIT License](LICENSE) 开源。
