# AGENTS_ADDENDUM.md — TarsCine-Y001 二期增量开发补充总纲 (v2.4-Delta)

> **增量协作铁律**：本项目 v2.2 核心主流程（推荐 -> 搜源 -> 下载 -> 刮削入库）已跑通并稳定运行。任何 AI 助手在执行本补充文档中的任务时，**严禁重构或破坏现有的核心数据库表结构、SSE 流式协议与内置四大机器人主链路**。所有新功能必须通过“插件注册中心”或“增量迁移”方式平滑接入。

---

## 1. 现有资产保护红线 (Do Not Break)
1. **核心表向后兼容**：不得删除或重命名 `sessions`、`messages`、`media_tasks`、`custom_agents` 的现有字段；如需新增字段，必须提供默认值（`DEFAULT`）并编写自动迁移检查脚本（启动时 `ALTER TABLE` 兼容老库）。
2. **协议锁定**：前端 `<GlassSurface>` 视觉规范、Z-Index 层级（`z-0` 至 `z-50`）、4 种内置卡片协议与 `GenericActionCard` 协议保持冻结，新增交互优先复用 `GenericActionCard` 的 `actions` 回调机制。
3. **主链路零侵入**：新增的自动化任务或机器人，必须通过 `backend/app/agents/plugins/` 自动注册机制挂载，禁止在 `TARS` 核心路由中硬编码 `if/else`。

---

## 2. 二期追加功能模块清单 (按推荐开发顺序排列)

### 追加模块 A：首启初始化引导（Onboarding）与对话流使用指南 [基础体验]
* **业务目标**：降低首次部署的配置门槛（自动检测路径与服务连通性），并提供零学习成本的对话交互引导。
* **后端接口与状态**：
  * `settings` 表新增布尔标志 `is_initialized`（默认 `false`）。
  * 新增连通性测试与防呆自检接口：
    * `POST /api/system/test-connection`：支持传入 `type`（`llm` | `tmdb` | `qbittorrent` | `aria2` | `jellyfin`）与临时配置参数，返回连通状态与延迟（ms）。
    * `POST /api/system/check-hardlink`：接收配置的下载目录与 `/media` 目录，调用 `os.stat().st_dev` 实时校验是否处于同一文件系统，返回是否支持硬链接及路径诊断建议。
  * TARS 总控增加 `/help` 或 `@TARS 帮助` 意图识别：动态读取 `AgentRegistry` 所有已激活机器人，渲染结构化 `generic_action_card` 使用说明书与【重新运行配置向导】回调按钮。
* **前端交互设计（严格复用 `<GlassSurface>`）**：
  * **四步极简初始化向导（Glass Onboarding Modal，Z-Index: `z-50`）**：当 `is_initialized == false` 时自动弹出，支持“稍后配置”跳过，设置中心保留【重新运行配置向导】入口（涵盖 Step 1 LLM 预设与测速、Step 2 TMDB Key 测试、Step 3 下载器与播放器连通测试、Step 4 NAS 路径硬链接同分区自检）。
  * **空会话欢迎态（Empty State）**：新建空对话时，中间工作台居中展示 TARS 欢迎玻璃卡片，内置 **4 个可点击的快捷示例指令胶囊**。

---

### 追加模块 B：多模态“截图 / 海报”以图搜片 [高爽感 AI 原生能力]
* **业务目标**：支持用户将刷短视频看到的电影截图、无字海报直接拖入或粘贴进对话框，由多模态视觉模型识别片名并无缝衔接搜源下载。
* **前端实现**：
  * **输入框多模态扩展**：支持拖拽图片、点击图库图标选择图片、或直接在输入框 `Ctrl+V / Cmd+V` 粘贴剪贴板截图；上传前在前端进行轻量压缩（限制最大边长 `1024px`，JPEG 质量 `0.85`）并展示微型玻璃缩略图预览及移除按钮。
  * **消息持久化**：发送带图消息时，将图片存储至 `/config/uploads/`（避免 Base64 撑爆 SQLite 数据库），`messages` 表仅保存相对路径（如 `attachments: ["/uploads/xxx.jpg"]`）。
* **后端实现**：
  * 新增静态资源挂载 `/uploads`（受全局访问锁鉴权保护）与图片上传接口 `POST /api/chat/upload`。
  * **TARS 视觉识别管线**：当用户消息包含图片附件时，TARS 自动将其封装为 LiteLLM / OpenAI 标准多模态 `image_url` 格式发给模型，提取核心影片信息（标准中英文片名、推测年份、识别置信度与简短依据），随后自动移交 `@CuratorBot` 匹配 TMDB 并输出标准的 `poster_grid` 卡片（附带一键【选定并搜源】按钮）。
  * **容错提示**：若当前配置的 LLM 模型不支持视觉输入（如纯文本小模型），捕获异常并返回友好玻璃提示卡片，引导用户在设置中切换支持 Vision 的模型（如 GPT-4o / Gemini / Qwen-VL）。

---

### 追加模块 C：自动榜单巡航与“今晚看啥”本地库存盲盒 [高粘性自动化]
* **业务目标**：解决“不知道看什么”的片荒痛点，实现后台全自动攒片，以及基于本地未看库存的趣味交互式抽片。
* **后端实现**：
  * **榜单巡航引擎（`AutoList` 调度任务）**：
    * 新增数据表 `auto_lists`：`id`, `name`, `source_type`（`tmdb_trending` | `douban_hot` | `imdb_top` | `custom_rss`）, `media_type`（`movie` | `tv`）, `min_rating`（最低评分过滤）, `max_items_per_run`（单次最多抓取数）, `cron_expr`（如每周五下午执行）, `enabled`。
    * 复用 `APScheduler` 注册 `run_auto_lists_job`，拉取榜单后自动过滤 `media_tasks` 中已存在及本地已入库影片，自动触发 `@HunterBot` + `@DownloaderBot` 静默下载，并通过 `TelegramNotifier` 发送“周末自动攒片报告”。
  * **本地库存“盲盒抽片”技能（扩展 `@CuratorBot`）**：
    * 当用户发送 `@CuratorBot 今晚看啥`、`抽盲盒` 或 `随便推荐一部库里的片` 时，优先扫描本地 `/media` 已入库影片（若配置了 Jellyfin，则调用 Jellyfin API 过滤掉 `Played == True` 的已看影片），随机抽取 1~3 部高分库存生成盲盒推荐理由。
* **前端交互**：
  * **盲盒抽片卡片**：复用 `generic_action_card`（展示电影海报、本地分辨率标签、AI 推荐语），底部提供【就看这部（跳转播放器）】与【换一个盲盒】交互按钮。
  * **榜单配置页**：在“设置与运维中心”新增 **“自动榜单巡航”** 管理面板，支持一键开启预设榜单（如 *TMDB 每周热门高分电影*、*豆瓣近期高分剧集*）及查看上次抓取日志。

---

### 追加模块 D：智能追剧与洗版订阅机器人 (`@WatcherBot` — Plugin)
* **业务目标**：针对未完结连载剧集或当前仅有枪版（TC/CAM）的电影，提供自动追更与画质升级（洗版）能力。
* **后端实现**：
  * 新增插件：`backend/app/agents/plugins/watcher_bot.py`（继承 `BaseAgent`，使用 `@register_agent` 注册）。
  * 新增数据表 `subscriptions`：`id`, `tmdb_id`, `media_type`, `title`, `target_quality`（如 `2160p`/`1080p`）, `current_episode`, `auto_upgrade`（是否允许洗版）, `status`（`active` | `completed`）, `last_checked_at`。
  * 调度接入：复用现有 `APScheduler`，每 6 小时执行 `check_subscriptions_job`，发现新集或更高画质时静默触发下载并通知。
* **前端交互**：对话流返回 `generic_action_card`（含【开启/取消自动洗版】按钮）；设置中心新增 **“追剧与洗版订阅”** Tab。

---

### 追加模块 E：本地媒体库盘点与重复片源清理 (`@LibrarianBot` — Plugin)
* **业务目标**：支持用自然语言查询本地片库（如按导演、分辨率、占用空间查询），并安全清理低画质旧版本。
* **后端实现**：
  * 新增插件：`backend/app/agents/plugins/librarian_bot.py`。异步扫描 `/media` 下的 `.nfo` 文件建立轻量索引。
  * **安全清理红线**：确认删除低画质副本时，**严格校验路径 `resolve()` 后位于 `/media` 白名单下**，仅删除目标视频与关联 `.nfo`/海报，绝不误删父级文件夹。
* **前端交互**：返回 `generic_action_card` 列出匹配影片及大小，提供【保留4K并清理1080P旧版】二次确认按钮。

---

### 追加模块 F：外挂字幕自动精调与补全 (`@SubtitleBot` — Plugin)
* **业务目标**：自动为冷门或生肉资源匹配下载中文字幕。
* **后端实现**：新增 `backend/app/agents/plugins/subtitle_bot.py`，对接 OpenSubtitles / Zimuku 适配器，匹配下载 `.srt`/`.ass` 后统一转码为 `UTF-8` 并重命名为 `.zh.srt` 放入媒体目录，随后触发 Jellyfin 刷新。

---

## 3. 增量开发指令模板 (供向 AI 派发任务时复制)

> **使用方式**：每次只向 AI 派发一个模块，把下面这段话连同 `AGENTS.md` 与本文件一起发给 AI 即可：
> 
> “请严格阅读 `AGENTS.md`（基础契约）和 `AGENTS_ADDENDUM.md`（增量规范）。现在请仅实现 **【追加模块 B：多模态“截图 / 海报”以图搜片】**。要求：1. 不破坏现有任何接口与表结构；2. 图片存本地 `/config/uploads/` 不存 Base64 进数据库；3. 识别出影片后无缝衔接 `@CuratorBot` 渲染 `poster_grid` 卡片。”
