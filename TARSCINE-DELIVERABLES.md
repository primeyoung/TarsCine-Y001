# TarsCine-Y001 · 重要产出物清单

> **用途**：留存备份 / 交接索引。回答「这个项目到底产出了什么、在哪、怎么验证」。
> **生成时间**：2026-11-22 22:55 ｜ **对应代码**：镜像 `tarscine-y001:latest`（265MB，commit 时点 22:35 构建）
> **配套**：`TARSCINE-DEV-DOC.md`（总开发文档）、`TARSCINE-TODO.md`（后续待完成）
>
> ⚠️ **本项目当前不是 git 仓库**（`git status` 返回 `not a git repository`）。所有版本历史只存在于本机文件与 AI 会话记录中，**建议尽快 `git init` 建立基线**，否则无法回滚。

---

## 1. 可运行交付物

| # | 产出物 | 位置 | 体积 | 状态 | 说明 |
|---|---|---|---|:--:|---|
| 1 | 主镜像 `tarscine-y001:latest` | 本机 Docker | **265MB** | ✅ 运行中（`127.0.0.1:8000`） | core 依赖 8 包，**已可跑 LLM**（httpx 直连） |
| 2 | 备选镜像 `tarscine-y001:llm` | 本机 Docker | 721MB | ✅ 可用 | 含 litellm，仅非 OpenAI 兼容协议才需要 |
| 3 | 云交付包 | `tarscine-cloud-bundle.tar.gz` | **59MB** | ✅ 已生成 | `docker save` 产物，云主机 `docker load` 即用 |
| 4 | 云交付包副本 | `deploy/cloud/tarscine.tar.gz` | 59MB | ✅ | 同上，随 `deploy/cloud/` 一起上传 |
| 5 | 前端产物 `dist/` | `frontend/dist/` | ~0.6MB | ✅ 已内置于镜像 | 6 个文件，见下表 |

**前端 bundle 指纹**（换版本时用于确认「云上到底跑的是哪一版」）：

| 文件 | 体积 | 说明 |
|---|---|---|
| `index-RAYodupv.js` | 108,790 B | 应用入口（G1 移动端版本） |
| `index-chn6zVNt.css` | 25,793 B | 含 3 处标准 `backdrop-filter`、`composer-pad`、`beacon-halo`、`data-kb` |
| `vendor-CZxjjC-b.js` | 226,240 B | React 等三方依赖 |
| `motion-Bhyx75J-.js` | 126,679 B | Framer Motion |
| `router-BGzyftuj.js` | 73,880 B | TanStack Router |
| `icons-RSz4rwkn.js` | 26,983 B | Lucide 图标 |

**启动方式速查**：

```bash
# 主服务（本机，容器）
docker compose up -d                       # 宿主 8000；改端口用 TARSCINE_HTTP_PORT
# 本地开发（无容器）
backend/.venv/Scripts/python -m uvicorn app.main:app --port 8000
cd frontend && npm run dev                 # 5173，proxy /api → 8000
# 单端口预演（模拟交付形态）
TARSCINE_STATIC_DIR=frontend/dist uvicorn app.main:app --port 8001
```

---

## 2. 代码资产

### 2.1 后端（`backend/app/`，**4,780 行 / 54 个 .py**）

| 模块 | 文件 | 行数 | 职责 | 状态 |
|---|---|---:|---|:--:|
| 入口 | `main.py` | 78 | FastAPI 装配、静态托管、调度器生命周期 | ✅ |
| 核心 | `core/config.py` / `database.py` / `migrations.py` | 37 / 34 / 116 | 配置、WAL 连接、增量迁移（`SCHEMA_VERSION=2`） | ✅ |
| 模型 | `models/agent.py` / `download.py` / `settings.py` / `auto_list.py` | 25/31/17/36 | `custom_agents` / `download_tasks` / `app_settings`(KV) / `auto_lists` | ✅ |
| 机器人基座 | `agents/base.py` / `registry.py` / `bootstrap.py` | 38/88/31 | `@register_agent`、Registry、内置注册 | ✅ |
| 总控 | `agents/orchestrator.py` | 173 | 规则路由兜底 + LLM 分类增强、卡片回调接力 | ✅ |
| 内置机器人 | `builtin/curator.py` | 169 | 片单策展 / 今日本周本月推荐 | ✅ |
| | `builtin/hunter.py` | 93 | Prowlarr 搜源 + 检索词清洗 + 评分排序 | ✅ |
| | `builtin/downloader.py` | 90 | qB / Aria2 双引擎派发 | ✅ |
| | `builtin/organizer.py` | 59 | 硬链接归档 + NFO + Jellyfin 刷新 | ✅ |
| 插件 | `plugins/example_subtitle_bot.py` | 37 | 模块 E 骨架示例（未启用） | 🚧 |
| 卡片 | `agents/cards.py` | 77 | 五类卡片协议构造 | ✅ |
| 自定义 | `agents/custom.py` | 76 | 零代码机器人 CRUD | ✅ |
| 适配器 | `adapters/qbittorrent.py` | 188 | qB 5.x（`stop/start`、`stoppedDL`） | ✅ |
| | `adapters/aria2.py` / `jellyfin.py` / `telegram.py` / `base.py` | 83/87/65/57 | 其余外部服务 + `format_speed` | ✅ |
| 服务 | `services/llm.py` | **225** | **唯一 LLM 入口**，双通道（http-openai / litellm） | ✅ |
| | `services/recommend.py` | 374 | 画像三档降级、推荐矩阵、盲盒、熔断 | ✅ |
| | `services/indexer.py` / `tmdb.py` / `net.py` | 163/170/178 | 搜源聚合 / TMDB / DoH 出网层 | ✅ |
| | `services/archive.py` / `nfo.py` / `path_mapping.py` | 118/36/85 | 归档 / NFO / 路径映射 | ✅ |
| | `services/task_events.py` | 55 | 进程内 pub/sub（SSE 广播） | ✅ |
| | `services/config_service.py` | 247 | 配置读写、密钥脱敏、`llm_backend_status` | ✅ |
| 调度 | `scheduler/engine.py` | 375 | 下载轮询、榜单巡航、熔断、自动接力 | ✅ |
| 路由 | `api/routes/system.py` | 323 | 首启状态 / 初始化 / 硬链接 / 连通性自检 | ✅ |
| | `api/routes/auto_lists.py` | 190 | 榜单 CRUD + 立即执行 | ✅ |
| | `api/routes/tasks.py` | 124 | 任务列表 / 建任务 / 暂停 / **SSE 流** | ✅ |
| | `api/routes/custom_agents.py` / `agents.py` / `chat.py` / `settings.py` / `images.py` / `health.py` | 100/46/60/62/41/23 | 其余端点 | ✅ |

### 2.2 前端（`frontend/src/`，**4,596 行 / 45 个文件**）

| 层 | 关键文件 | 行数 | 说明 |
|---|---|---:|---|
| 状态 | `store/useAppStore.ts` | 563 | 对话流、任务、Agent、卡片回调、`toggleCardPause` |
| | `store/mock.ts` | 162 | 演示数据（仅兜底用） |
| 类型 | `lib/types.ts` | 346 | 卡片协议 / Agent / Task 全部类型 |
| 网络 | `lib/api.ts` | 242 | REST + `streamChat` + `streamTaskEvents` + `setTaskPaused` |
| 移动端 | `lib/useViewport.ts` | 88 | `--app-h` / `data-kb` / `visualViewport` 双 hook |
| | `lib/useActivity.ts` | 32 | 呼吸灯计数聚合 |
| | `lib/useTaskStream.ts` | 54 | SSE 订阅 + 指数退避重连 |
| 对话 | `components/chat/ChatStream.tsx` | 144 | 50 条分页 + 距底锚定 + 智能贴底 + 回到最新 |
| | `ChatComposer` / `MessageBubble` / `AgentMentionPopover` / `RichText` | 129/70/43/36 | 输入、气泡、@ 联想、Markdown |
| 卡片 | `cards/DownloadProgressCard.tsx` | 108 | 真接口暂停 + `taskId` 认领 + 自适应单位 |
| | `cards/PosterGridCard` / `TorrentListCard` / `MediaReadyCard` / `GenericActionCard` / `PosterArt` | 59/73/50/37/51 | 其余四类 + 海报渐变兜底 |
| 玻璃 | `glass/GlassSurface.tsx` / `AmbientBackground.tsx` | 47/27 | 视觉红线的唯一实现处 |
| 布局 | `layout/TaskPipeline.tsx` / `Sidebar.tsx` / `MobileTabBar.tsx` / `AppShell.tsx` | 178/143/77/37 | 三栏 + 底部胶囊导航 |
| 页面 | `pages/SettingsPage.tsx` / `AgentsPage.tsx` / `WorkbenchPage.tsx` | 366/202/45 | 设置中心 / 机器人管理 / 工作台 |
| 模块 A | `onboarding/OnboardingWizard.tsx` / `WelcomePanel.tsx` / `commands.ts` | 319/53/28 | 四步向导 + 快捷胶囊 |
| 模块 C | `settings/RecommendMatrix.tsx` | 368 | 推荐矩阵 + 榜单 + 盲盒配置 |
| G1 | `ui/StatusBeacon.tsx` | 68 | 呼吸光晕角标 |
| 样式 | `globals.css` | 125 | CSS 变量、`#root` 高度、键盘态、呼吸灯关键帧 |

**合计：后端 4,780 + 前端 4,596 ≈ 9,376 行**（不含依赖与构建产物）。

---

## 3. 文档资产

| 文件 | 体积 | 性质 | 说明 |
|---|---|---|---|
| `AGENTS.md` | 12.5KB | **原始契约 v2.1（冻结）** | M1–M4 主线定义，不再修改 |
| `AGENTS_ADDENDUM.md` | 11.6KB | **二期契约 v2.6-Delta（冻结）** | 模块 A–G + 派发模板 |
| `docs/TARSCINE-DESIGN.md` | 31.5KB | 合并版总设计 v3.0-merged | 契约 + 实现状态合并，日常派发用这份 |
| `docs/TARSCINE-BACKLOG.md` | 20.3KB | 待办 + **缺陷台账（18 条）** | 含联调修复清单、环境坑 |
| `docs/TARSCINE-DELIVERABLES.md` | 本文 | 产出物索引 | 备份清单 |
| `docs/TARSCINE-DEV-DOC.md` | 待生成 | **总开发文档 v3.1** | 含追加与全部修正，自包含可交接 |
| `docs/TARSCINE-TODO.md` | 待生成 | 后续待完成 | 剩余项 + 时间窗 + 验收 |
| `.env.example` | 2.9KB | 配置模板 | 全量环境变量注释 |
| `docker-compose.yml` / `Dockerfile` / `.dockerignore` | 2.7KB / 2.0KB / 392B | 构建与部署 | 多阶段构建、`LLM_EXTRAS` 开关 |
| `deploy/cloud/` | Caddyfile / deploy.sh / export-image.sh / compose / .env | 云主机一键包 | ⚠️ `.env` 含真实密钥，勿外传 |
| `deploy/lab/docker-compose.yml` + `.env` | 3.4KB / 528B | 本地实验台 | Prowlarr + qB + Aria2 + Jellyfin，主链路验证用 |
| `.workbuddy/memory/2026-11-20/21/22.md` | 13/36/10KB | 逐日工作日志 | 决策过程与踩坑原始记录 |
| `.workbuddy/memory/MEMORY.md` | 9.1KB | 项目长期笔记 | 红线、决策、未实测项 |

---

## 4. 验收证据（`deploy/verify/`，28 张真实截图）

| 分组 | 文件 | 证明什么 |
|---|---|---|
| **首启引导（模块 A）** | `onboarding-step1/2/3.png` | 四步玻璃向导的分步渲染 |
| | `welcome-panel.png`、`mobile-welcome.png`、`mobile-onboarding.png` | 空会话 4 胶囊 + 移动端向导 |
| | `help-card.png` | `/help` 动态读 Registry 生成使用说明卡 |
| **推荐矩阵（模块 C）** | `module-c-today.png`、`module-c-matrix.png`、`module-c-newform.png` | 今日/本周/本月三档 + 榜单配置表单 |
| | `module-c-blindbox.png`、`module-c-reroll.png` | 库存盲盒 + 换一个 |
| | `module-c-fused.png` | 熔断红框告警（阈值 999999GB 实测） |
| | `module-c-mobile.png` | 移动端矩阵布局 |
| **LLM / 设置** | `llm-settings.png` | 6 个 LLM 字段可编辑（含 `max_tokens` 不再被脱敏） |
| | `lab-settings.png`、`lab-today.png` | 真实服务配置页 + 真实片单 |
| | `desktop.png`、`desktop-fixed.png` | `backdrop-filter` 失效 → 修复后的桌面对比 |
| **移动端（G1）** | `g1-cdp-390.png` | **CDP 390×844 真机尺寸**，无横向溢出 |
| | `g1-mobile-final.png`、`g1-mobile-390.png`、`g1-mobile-card.png` | 底部导航 + 呼吸灯角标 + 卡片换行 |
| | `g1-390-paged.png`、`g1-390-normal.png` | 分页「加载更早的 N 条」前后 |
| | `g1-desktop.png`、`g1-desktop-final.png` | 桌面档角标收敛为两处 |
| | `mobile.png`、`mobile-onboarding.png` | G1 之前的移动端基线（对照用） |

> 说明：`g1-mobile-card.png` 等早期图有「元素被裁」假象，源于 Chrome `--window-size` 被最小宽度钳制（详见 DEV-DOC 环境坑）。以 `g1-cdp-390.png` 为准。

---

## 5. 运行数据与配置（备份取舍）

| 路径 | 内容 | 是否需备份 | 说明 |
|---|---|:--:|---|
| `config/tarscine.db` + `-wal` + `-shm` | 主库（4 表） | **是** | 约 165KB；含下载历史、榜单、KV 配置 |
| `config/jobs.db` | APScheduler JobStore | 建议 | 丢失只会重建定时任务，不丢数据 |
| `backend/.labcfg/tarscine.db` | 联调实验台配置 | 否 | 含真实服务地址与 Key，仅本机调试用 |
| `backend/data/` | 早期默认数据目录 | 否 | 已被 `config/` 取代，可归档 |
| `downloads/`、`media/` | 下载与归档 | 视情况 | 归档库才是真正的「资产」 |
| `deploy/cloud/.env` | 云主机密钥（域名/密码） | **是（加密保存）** | 明文含 Basic Auth 与 API Key |
| `frontend/node_modules`、`backend/.venv` | 依赖 | 否 | 可重装 |
| `_a.txt` `_b.txt` `_bb.txt` `_m.txt` `_s2.txt` | 调试中间产物（根目录 5 个） | 否 | **可删**，非源码 |

---

## 6. 验收基线（复现用命令）

```bash
# 1. 全量外部服务自检（应在 passed=6 failed=0 skipped=1，仅 Telegram 未配）
curl -s -X POST http://127.0.0.1:8000/api/system/test-connection \
  -H 'Content-Type: application/json' -d '{"probe_llm":true}' | python -m json.tool

# 2. 首启状态 / 平台 / 迁移版本
curl -s http://127.0.0.1:8000/api/system/status | python -m json.tool

# 3. 硬链接可用性（st_dev 一致性 + 磁盘余量）
curl -s -X POST http://127.0.0.1:8000/api/system/check-hardlink | python -m json.tool

# 4. 下载进度 SSE（独立端点，看 snapshot / task / ping）
curl -N http://127.0.0.1:8000/api/tasks/stream

# 5. 机器人列表（必须来自 Registry，非前端硬编码）
curl -s http://127.0.0.1:8000/api/agents | python -m json.tool
```

**实测历史值**（2026-11-22，实验台全开时）：
`tmdb 4822ms`（DoH 冷启动）/ `prowlarr 9.5ms` / `qb 34.8ms` / `aria2 3.4ms` / `jellyfin 4.8ms` / `llm 2275ms (http-openai)`。
