# AGENTS_ADDENDUM.md — TarsCine-Y001 二期增量开发补充总纲 (v2.3-Delta)

> **增量协作铁律**：本项目 v2.2 核心主流程（推荐 -> 搜源 -> 下载 -> 刮削入库）已跑通并稳定运行。任何 AI 助手在执行本补充文档中的任务时，**严禁重构或破坏现有的核心数据库表结构、SSE 流式协议与内置四大机器人主链路**。所有新功能必须通过“插件注册中心”或“增量迁移”方式平滑接入。

---

## 1. 现有资产保护红线 (Do Not Break)
1. **核心表向后兼容**：不得删除或重命名 `sessions`、`messages`、`media_tasks`、`custom_agents` 的现有字段；如需新增字段，必须提供默认值（`DEFAULT`）并编写自动迁移检查脚本（启动时 `ALTER TABLE` 兼容老库）。
2. **协议锁定**：前端 `<GlassSurface>` 视觉规范、Z-Index 层级（`z-0` 至 `z-50`）、4 种内置卡片协议与 `GenericActionCard` 协议保持冻结，新增交互优先复用 `GenericActionCard` 的 `actions` 回调机制。
3. **主链路零侵入**：新增的自动化任务或机器人，必须通过 `backend/app/agents/plugins/` 自动注册机制挂载，禁止在 `TARS` 核心路由中硬编码 `if/else`。

---

## 2. 二期追加功能模块清单 (建议按顺序逐个派发)

### 追加模块 A：首启初始化引导（Onboarding）与对话流使用指南 [优先体验升级]
* **业务目标**：降低首次部署的配置门槛（自动检测路径与服务连通性），并提供零学习成本的对话交互引导，同时不增加额外的复杂菜单。
* **后端接口与状态**：
  * `settings` 表/配置中新增布尔标志 `is_initialized`（默认 `false`）。
  * 新增连通性测试与防呆自检接口：
    * `POST /api/system/test-connection`：支持传入 `type`（`llm` | `tmdb` | `qbittorrent` | `aria2` | `jellyfin`）与临时配置参数，返回连通状态与响应延迟（ms）。
    * `POST /api/system/check-hardlink`：接收配置的下载目录与 `/media` 目录，调用 `os.stat().st_dev` 实时校验是否处于同一文件系统，返回是否支持硬链接及具体路径诊断建议。
  * TARS 总控增加 `/help` 或 `@TARS 帮助` 意图识别：动态读取 `AgentRegistry` 中所有已激活机器人的 `tag` 与 `description`，渲染结构化的 `generic_action_card` 使用说明书（含 `@呼叫` 语法说明与一键触发【重新运行配置向导】回调）。
* **前端交互设计（严格复用 `<GlassSurface>`）**：
  * **四步极简初始化向导（Glass Onboarding Modal，Z-Index: `z-50`）**：当 `is_initialized == false`（或核心必填项为空）时自动弹出，支持点击“稍后配置”跳过，并在设置中心保留【重新运行配置向导】按钮：
    1. **Step 1 — 大脑激活**：配置 LLM（提供 OpenAI / DeepSeek / 本地 Ollama 一键预设模板），内置【测试连接】按钮。
    2. **Step 2 — 影视大脑**：填写 TMDB API Key（附带申请指引与可选代理地址），一键测试元数据拉取。
    3. **Step 3 — 下载与播放**：选择 qBittorrent 或 Aria2，填写地址与账密并测试连通；可选填 Jellyfin 地址。
    4. **Step 4 — NAS 路径与硬链接自检**：填写下载器远端路径与容器映射路径，实时展示 `check-hardlink` 自检结果（绿灯“支持秒级硬链接” / 黄灯“跨分区将降级为复制模式”）。完成后写入 `is_initialized = true`。
  * **空会话欢迎态（Empty State）**：新建对话且消息为空时，中间工作台居中展示 TARS 欢迎玻璃卡片，内置 **4 个可点击的快捷示例指令胶囊**（如 *“周末想看高分烧脑科幻片”*、*“帮我找《沙丘2》4K中字资源”*、*“查看当前下载进度”*、*“有哪些可用的机器人？”*），点击直接填入输入框或一键发送。

---

### 追加模块 B：智能追剧与洗版订阅机器人 (`@WatcherBot` — Plugin)
* **业务目标**：针对未完结连载剧集或当前仅有枪版（TC/CAM）的电影，提供自动追更与画质升级（洗版）能力。
* **后端实现**：
  * 新增插件：`backend/app/agents/plugins/watcher_bot.py`（继承 `BaseAgent`，使用 `@register_agent` 注册）。
  * 新增数据表 `subscriptions`：`id`, `tmdb_id`, `media_type`, `title`, `target_quality`（如 `2160p`/`1080p`）, `current_episode`（剧集进度）, `auto_upgrade`（是否允许洗版）, `status`（`active` | `completed`）, `last_checked_at`。
  * 调度接入：复用现有 `APScheduler`，新增每 6 小时执行一次的 `check_subscriptions_job`，发现新集数或更高画质资源时，自动调用 `@HunterBot` 与 `@DownloaderBot` 静默执行，并通过 `TelegramNotifier` 推送通知。
* **前端交互**：
  * 对话流中 `@WatcherBot` 返回 `generic_action_card`，展示当前订阅状态及【开启/取消自动洗版】按钮。
  * 在“设置与运维中心”新增 **“追剧与洗版订阅”** Tab，支持查看与手动删除订阅项。

---

### 追加模块 C：本地媒体库盘点与重复片源清理 (`@LibrarianBot` — Plugin)
* **业务目标**：让用户能用自然语言查询 NAS 里“已经有哪些片”、“有哪些占空间的双版本重复电影”，并安全清理低画质旧版本。
* **后端实现**：
  * 新增插件：`backend/app/agents/plugins/librarian_bot.py`。
  * 扫描服务：异步扫描 `/media/movies` 与 `/media/tv` 目录，读取现有的 `.nfo` 文件建立轻量索引，支持按演员、年份、分辨率检索本地库。
  * 安全清理机制：当用户确认删除某个低画质副本时，**严格校验路径 `resolve()` 后位于 `/media` 白名单下**，仅删除目标视频与关联 `.nfo`/海报，绝不误删父级文件夹或其他影片。
* **前端交互**：
  * 返回 `generic_action_card`，列出本地匹配影片的分辨率、文件大小及路径，提供【保留4K并清理1080P旧版】二次确认按钮。

---

### 追加模块 D：外挂字幕自动精调与补全 (`@SubtitleBot` — Plugin)
* **业务目标**：解决部分冷门电影或生肉资源下载后缺少中文字幕的问题。
* **后端实现**：
  * 新增插件：`backend/app/agents/plugins/subtitle_bot.py`。
  * 能力集成：对接开源字幕源（如 OpenSubtitles API / Zimuku 适配器），支持自动按视频文件哈希或文件名匹配下载 `.srt`/`.ass` 字幕，统一转码为 `UTF-8` 并重命名为与视频同名的 `.zh.srt` 放入对应媒体目录，随后触发 Jellyfin 刷新。
* **前端交互**：
  * 支持在 `@OrganizerBot` 归档完成后作为可选后置动作，或由用户随时 `@SubtitleBot 帮我给《XXX》找中文字幕` 触发。

---

### 追加模块 E：移动端 PWA 快捷指令与全局状态角标优化
* **业务目标**：提升手机端添加到主屏幕后的日常使用与长对话流畅度。
* **前端实现**：
  * 在顶部状态栏或悬浮胶囊导航旁，增加一个微型液态玻璃呼吸灯角标，实时显示当前 `下载中任务数` 与 `订阅监控数`（点击直接滑出右侧任务抽屉）。
  * 优化对话流长列表分页加载（默认加载最近 50 条消息，向上滚动加载历史），防止长期使用后 DOM 节点过多导致手机端掉帧。

---

## 3. 增量开发指令模板 (供向 AI 派发任务时复制)

> **使用方式**：每次只向 AI 派发一个模块，把下面这段话连同 `AGENTS.md` 与本文件一起发给 AI 即可：
> 
> “请严格阅读 `AGENTS.md`（基础契约）和 `AGENTS_ADDENDUM.md`（增量规范）。现在请仅实现 **【追加模块 A：首启初始化引导与对话流使用指南】**。要求：1. 不破坏现有任何接口与表结构；2. 前端模态框与欢迎卡片严格复用 `<GlassSurface>`、锁定 Z-Index 层级及 `GenericActionCard`；3. 完整实现后端路径硬链接同分区自检与服务连通性测试接口。”
