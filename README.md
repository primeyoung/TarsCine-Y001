<div align="center">

# 🎬 TarsCine-Y001
**Your Multi-Agent AI Cinema Concierge for NAS**  
*灵感源自《星际穿越》TARS 机器人 × Apple Liquid Glass 空间视觉美学*

[![Next.js](https://shields.io)](https://nextjs.org)
[![FastAPI](https://shields.io)](https://tiangolo.com)
[![Docker](https://shields.io)](https://docker.com)
[![PWA](https://shields.io)](https://web.dev)

</div>

---

## ✨ 项目简介

**TarsCine-Y001** 是一款专为私人 NAS 打造的**跨端多智能体（Multi-Agent）影视自动化管家**。

不同于传统的表单式媒体管理工具，TarsCine-Y001 借鉴了 **OpenManus** 的多智能体协作交互体验，并采用极具通透感的 **Apple Liquid Glass（苹果液态玻璃）** 界面设计。你只需在统一的对话工作台里说出想看的电影，四个专职 AI 机器人便会接力完成**“意图推荐 → 全网搜源 → 调度下载 → 硬链接刮削入库”**的全流程，让你在手机、电脑上随时随地指挥 NAS 干活。

---

## 🤖 核心特性：四大专职 AI 机器人矩阵

系统由 **TARS 主控调度器** 统一协调，支持在对话流中 `@呼叫指定机器人` 分步确认，也支持开启“全自动托管模式”：

| 机器人代号 | 职责定位 | 核心能力与外部生态对接 |
| :--- | :--- | :--- |
| **`@CuratorBot`**<br>*(片单推荐官)* | 模糊需求理解与选片 | 对接 **TMDB API**，根据心情、类型或相似电影生成交互式海报卡片，支持一键加入任务流。 |
| **`@HunterBot`**<br>*(资源猎手)* | 全网/PT站搜源与洗版决策 | 对接 **Prowlarr / Jackett**，根据自定义画质策略（如 `4K HDR > 1080P`、中文字幕、做种数）智能打分筛选最优种子。 |
| **`@DownloaderBot`**<br>*(下载调度员)* | 双引擎下载队列管控 | 同时支持 **qBittorrent**（PT/BT 种子磁力）与 **Aria2**（网盘直链/HTTP），在对话内嵌入实时动态下载进度卡片。 |
| **`@OrganizerBot`**<br>*(刮削归档管家)* | 硬链接整理与全播放器兼容 | 秒级硬链接归档（不占双倍空间、不影响做种）；自动生成标准 **`.nfo` 元数据与高清海报**，完美兼容 **极空间影视、飞牛影视 (fnOS)** 目录刮削，并主动触发 **Jellyfin** 媒体库刷新。 |

### 🌟 更多亮点
* **Apple Liquid Glass 空间视觉**：深邃流光暗色背景配合多层毛玻璃折射（`backdrop-blur`）、1px 高光边缘与苹果弹簧阻尼动效；内置可扩展 CSS 变量架构，支持自定义主题。
* **全平台无缝覆盖**：桌面端（Windows / macOS）采用三栏式空间看板；移动端（iOS / Android）深度适配 **PWA**，支持“添加到主屏幕”获得全屏原生 App 体验与底部悬浮胶囊导航。
* **Telegram Bot 消息推送（已预留）**：支持在资源就绪、下载完成、入库可播时通过 Telegram 推送带海报通知及快捷确认按钮。

---

## 🛠️ 技术栈与项目结构

* **前端**：Next.js 15 (App Router) · React 19 · TypeScript · Tailwind CSS · Framer Motion · Lucide Icons
* **后端**：Python 3.11+ · FastAPI · SQLite (SQLModel) · SSE (Server-Sent Events)
* **部署**：Docker & Docker Compose (`linux/amd64` & `linux/arm64`)

```text
tarscine-y001/
├── AGENTS.md                 # 跨 AI 编程工具统一开发契约
├── docker-compose.yml        # NAS Docker 一键部署模板
├── frontend/                 # Next.js 前端与 Liquid Glass 组件库
└── backend/                  # FastAPI 多智能体编排与适配器服务
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
      - "8321:3000"   # Web UI 访问端口
      - "8322:8000"   # FastAPI 后端端口
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

- [x] **Milestone 1**：初始化前后端工程脚手架，构建 `Apple Liquid Glass` 核心玻璃组件库与 PWA 配置。
- [ ] **Milestone 2**：完成桌面三栏布局、移动端底部胶囊导航及四大机器人富媒体交互卡片（Mock 闭环）。
- [ ] **Milestone 3**：接入大模型路由与 `qBittorrent`、`Aria2`、`Jellyfin`、`NFO刮削`、`Telegram Bot` 适配器。
- [ ] **Milestone 4**：完善可视化设置面板、主题自定义模块及多架构 Docker 镜像发布。

---

## 📄 开源协议

本项目基于 [MIT License](LICENSE) 开源，欢迎提交 Issue 或 Pull Request 共同完善你的私人 TARS 影院管家。

