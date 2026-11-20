<div align="center">

# 🎬 TarsCine-Y001
**Your Multi-Agent AI Cinema Concierge for NAS**  
*灵感源自《星际穿越》TARS 机器人 × Apple Liquid Glass 空间视觉美学*

[![Vite](https://shields.io)](https://vite.dev)
[![FastAPI](https://shields.io)](https://fastapi.tiangolo.com)
[![Docker](https://shields.io)](https://docker.com)
[![PWA](https://shields.io)](https://web.dev)

</div>

---

## ✨ 项目简介

**TarsCine-Y001** 是一款专为私人 NAS 打造的**跨端多智能体（Multi-Agent）影视自动化管家**。

不同于传统的表单式媒体管理工具，TarsCine-Y001 借鉴了 **OpenManus** 的多智能体协作交互体验，并采用极具通透感的 **Apple Liquid Glass（苹果液态玻璃）** 界面设计。你只需在统一的对话工作台里说出想看的电影，内置的四大专职 AI 机器人（以及你自定义扩展的机器人）便会接力完成**“意图推荐 → 全网搜源 → 调度下载 → 硬链接刮削入库”**的全流程，让你在手机、电脑上随时随地指挥 NAS 干活。

---

## 🤖 核心特性：四大内置机器人 + 插件化扩展矩阵

系统由 **TARS 主控调度器** 统一协调，支持在对话流中 `@呼叫指定机器人` 分步确认，也支持开启“全自动托管模式”：

| 机器人代号 | 职责定位 | 核心能力与外部生态对接 |
| :--- | :--- | :--- |
| **`@CuratorBot`**<br>*(片单推荐官)* | 模糊需求理解与选片 | 对接 **TMDB API**，根据心情、类型或相似电影生成交互式海报卡片，自动过滤已入库影片。 |
| **`@HunterBot`**<br>*(资源猎手)* | 全网/PT站搜源与洗版决策 | 对接 **Prowlarr / Jackett**，结合 `guessit` 智能解析种子文件名，按自定义策略（如 `4K HDR > 1080P`、中文字幕、做种数）打分排序。 |
| **`@DownloaderBot`**<br>*(下载调度员)* | 双引擎下载队列管控 | 同时支持 **qBittorrent**（PT/BT 种子磁力）与 **Aria2**（网盘直链/HTTP），后台持久化轮询并在对话内嵌入实时下载进度卡片。 |
| **`@OrganizerBot`**<br>*(刮削归档管家)* | 硬链接整理与全播放器兼容 | 秒级硬链接归档（内置同分区检测与路径映射）；自动生成标准 **`.nfo` 元数据、高清海报与 UTF-8 字幕**，完美兼容 **极空间影视、飞牛影视 (fnOS)** 目录刮削，并主动触发 **Jellyfin** 媒体库刷新。 |

### 🌟 更多工程与体验亮点
* **🧩 插件化与自定义机器人扩展（Agent Registry）**：
  * **可视化零代码新建**：支持在设置中心创建专属 `@自定义机器人`，配置独立 `System Prompt` 与外部 **Webhook**（如联动 HomeAssistant 一键开启客厅影院模式）。
  * **开发者插件目录**：支持在 `plugins/` 目录通过 `@register_agent` 装饰器无缝挂载新技能机器人，前端列表与 `@` 快捷菜单自动动态识别。
* **🪟 Apple Liquid Glass 空间视觉与性能自适应**：深邃流光暗色背景配合多层毛玻璃折射（`backdrop-blur`）、1px 高光边缘与苹果弹簧阻尼动效；内置**低功耗降级开关**与可扩展 CSS 变量架构，兼顾旗舰视觉与老旧核显/移动端 60fps 流畅度。
* **📱 全平台无缝覆盖（单容器极简交付）**：前后端合并单端口交付，零跨域配置；桌面端采用三栏式空间看板，移动端（iOS / Android）深度适配 **PWA**，支持“添加到主屏幕”获得全屏原生 App 体验与底部悬浮胶囊导航。
* **✈️ Telegram Bot 消息推送（已预留）**：支持在资源待确认、下载完成、入库就绪时通过 Telegram 推送带海报通知及快捷确认按钮。

---

## 🛠️ 技术栈与项目结构

* **前端（纯 SPA 高性能架构）**：Vite 6 · React 19 · TypeScript · TanStack Router · Zustand · Tailwind CSS · Framer Motion
* **后端（纯 Python 高稳定异步架构）**：Python 3.11+ · FastAPI · PydanticAI / LiteLLM · SQLite (WAL 模式) · APScheduler · SSE
* **部署**：Docker Multi-stage Build & Docker Compose (`linux/amd64` & `linux/arm64`)

```text
tarscine-y001/
├── AGENTS.md                 # 跨 AI 编程工具统一架构与设计契约 (v2.1)
├── Dockerfile                # 前后端合并多阶段构建镜像文件
├── docker-compose.yml        # NAS Docker 一键部署模板
├── frontend/                 # Vite + React 前端与 Liquid Glass 组件库
└── backend/                  # FastAPI 多智能体编排、插件系统与适配器服务
```

---

## 🚀 NAS Docker 快速部署

> ⚠️ **硬链接目录规范提示**：在飞牛OS、极空间或群晖部署时，请务必将**下载目录**与**媒体库目录**放在同一个宿主机父级目录下统一映射为 `/media`，以确保文件归档时实现不占额外空间的秒级硬链接（Hardlink）。

1. **创建配置目录并编辑 `docker-compose.yml`**：

```yaml
version: "3.8"

services:
  tarscine-y001:
    image: tarscine-y001:latest
    container_name: tarscine-y001
    restart: unless-stopped
    ports:
      - "8321:8000"   # 单端口统一访问 Web UI 与 API
    environment:
      - TZ=Asia/Shanghai
      - PUID=1000     # 替换为你的 NAS 用户 UID
      - PGID=1000     # 替换为你的 NAS 用户 GID
    volumes:
      - ./config:/config
      # 宿主机 /Media 下建议包含 downloads/ 与 movies/ 两个子目录
      - /path/to/your/nas/Media:/media
```

2. **启动容器**：

```bash
docker-compose up -d
```

3. **访问与移动端安装**：
   * **电脑端**：浏览器打开 `http://<你的NAS_IP>:8321`
   * **iOS / Android**：使用 Safari 或 Chrome 访问上述地址，点击浏览器的 **“分享 -> 添加到主屏幕”** 即可作为原生全屏 App 使用。

---

## 🗺️ 开发路线图 (Roadmap)

- [ ] **Milestone 1**：初始化 Vite + FastAPI 工程脚手架，配置 SQLite WAL 模式、`AgentRegistry` 基础接口与 `Apple Liquid Glass` 核心组件库。
- [ ] **Milestone 2**：完成桌面三栏布局、移动端底部胶囊导航、动态 `@机器人` 菜单及 4+1 种富媒体交互卡片（Mock 闭环）。
- [ ] **Milestone 3**：接入 PydanticAI/LiteLLM 大模型路由、插件扫描与自定义机器人管理、`APScheduler` 持久化轮询及各外部工具适配器。
- [ ] **Milestone 4**：完善可视化设置面板（LLM/路径映射/自定义机器人/主题调节）及前后端合并多架构 Docker 镜像发布。

---

## 📄 开源协议

本项目基于 [MIT License](LICENSE) 开源，欢迎提交 Issue 或 Pull Request 共同完善你的私人 TARS 影院管家。
