# Codeg iOS 重构方案：原生布局、底部 Tabs 与交互逻辑

> v1.0 · 2026-06-10 · 基于代码库全量勘察（48 个 Swift 文件，约 7,500 行）与 2026 年主流 AI App 移动端对标

---

## 0. 一页纸结论（TL;DR）

**产品定位**：codeg-ios 不是通用聊天 App，而是 **coding agent 的远程驾驶舱**。同类标杆是 2026-05 发布的 ChatGPT 内 Codex mobile（"remote control, not a mobile IDE"）和 Claude 的 Remote Control。移动端用户只做五件事：**看进度、收审批、给指令、开任务、读结果**。整个方案围绕这五件事组织。

**五个结构性决策**：

1. **底部 Tabs 重组**：`会话 / 项目 / 活动 / 设置` + 系统分离式搜索 Tab（`role: .search`）。「服务器」从一级 Tab 降级为会话页标题菜单 + 设置内管理页。
2. **全局「正在运行」状态条**：用 iOS 26 `tabViewBottomAccessory`（Apple Music now-playing 范式）常驻展示运行中的 agent 任务，一键直达。这是本方案的招牌交互。
3. **补齐三条缺失的主流程**：新建会话（目前完全没有）、审批响应（协议里有 `permission_request` 事件但无 UI）、全局活动视图。
4. **会话详情升级**：按工具类型分化的卡片（diff 卡、终端卡、审批卡）、智能滚动、会话信息面板。
5. **架构地基**：路由枚举化（支撑深链/推送）、App 级 `SessionHub` 单 socket 多路复用（支撑活动页/角标/状态条）、磁盘缓存 stale-while-revalidate（秒开）。

**节奏**：M0–M5 六个里程碑，每个独立可交付；M0–M2 不依赖任何服务端改动，M3 起有少量服务端确认项（见 §11）。

---

## 1. 对标：2026 年主流 AI App 怎么做

| 产品 | 移动端信息架构（2026 年中） | 对 codeg 的启示 |
|---|---|---|
| **ChatGPT**（2026-04 改版） | 聊天为中心 + 左侧抽屉；Images / Codex / Pulse / Apps 收进抽屉顶部横向栏 | 通用聊天用抽屉做沉浸；但「计划级功能」被提升为横向入口——功能分区仍在回归 |
| **Codex mobile**（2026-05，ChatGPT 内） | 任务列表 + 实时进度/终端输出 + **审批/拒绝命令** + diff 审阅 + 新任务发起 + 推送 | 与 codeg 几乎同构。证明 agent 远程控制的核心循环 = 监控 → 审批 → 转向 → 新任务 |
| **Claude**（Remote Control） | 手机接管本地 Claude Code 会话，继续对话 | 「会话连续性」是底线能力；官方未做的 session 管理 UI 正是 codeg 的空间 |
| **Gemini**（2026-05 改版） | 沉浸式深色 + 粒子动效 + 导航抽屉 | 深色沉浸是行业审美共识；现有 dark + Liquid Glass 方向正确 |
| **Perplexity / Comet** | 主 App 走 Tab + 侧栏；Comet iOS 用 Safari 式底栏 + Liquid Glass 形变 | 拥抱 iOS 26 系统材质和系统导航范式的产品观感最"原生" |
| **第三方 Claude Code 客户端**（Happy、Claude Remote 等） | 会话列表 + 推送通知 + 聊天式转向 | 推送/通知是这一品类的灵魂；纯只读监控也有市场，但「可操作」是分水岭 |

**三条结论**：

1. 通用聊天 App（单会话沉浸）用抽屉；**agent 监控类产品全部收敛到「任务列表 + 状态 + 审批」结构**。codeg 是多会话并行监控工具 → 底部 Tabs 是正确形态（iOS 也没有原生抽屉组件，HIG 推荐 tab bar 做顶级分区）。
2. **审批 + 推送是 agent 移动端的核心卖点**（Codex mobile 的头条功能就是"approve or deny pending commands without returning to your desk"）。codeg 协议已有 `permission_request` 事件，客户端却没有任何 UI——这是最大的产品缺口。
3. 视觉上深色沉浸 + 玻璃材质是共识，现有设计系统保留，做增量不做推翻。

参考源：[9to5Mac：Codex 登陆 ChatGPT 移动端](https://9to5mac.com/2026/05/14/openai-brings-codex-control-to-chatgpt-for-iphone-and-android/) · [MacRumors](https://www.macrumors.com/2026/05/15/openai-brings-codex-chatgpt-mobile-app/) · [ChatGPT 更新日志](https://releasebot.io/updates/openai/chatgpt) · [9to5Google：Gemini 全面改版](https://9to5google.com/2026/05/03/gemini-full-redesign/) · [MacStories：Comet for iOS](https://www.macstories.net/news/comet-is-the-first-agentic-browser-for-ios-worth-trying/) · [Claude Code 移动客户端横评](https://nimbalyst.com/blog/best-mobile-apps-for-claude-code-2026/)

---

## 2. 现状盘点

### 2.1 做得好的（全部保留）

- 现代架构：`@MainActor @Observable` 视图模型、值类型 `CodegClient`、无单例污染。
- 流式渲染性能：`LiveTurn` 分段可观察对象、流式期 plain 渲染 + 完成后一次性 Markdown 解析。
- 协议正确性：attach-before-prompt、`connection_gone` 单次重试、camelCase/snake_case 双向编码。
- iOS 26 原生观感：Liquid Glass 组件族、scroll-edge 自动磨砂、大标题折叠（见 memory `ios26-nav-chrome`）。
- 设计系统已 token 化（`Theme.swift`），改版可从令牌层传播。

### 2.2 关键差距（重构靶心）

| # | 差距 | 证据 | 后果 |
|---|---|---|---|
| 1 | **无法新建会话** | 全 App 无任何「新建」入口；`acp_connect` 仅用于续连 | 只能回复服务端已存在的会话，半成品体验 |
| 2 | **无审批 UI** | `AcpEvent.swift` 不解码 `permission_request`；无任何 approval 视图 | agent 卡在等待授权时，手机端完全不可见、不可操作 |
| 3 | **无全局活动视图** | 状态只在单个会话详情内可见 | 多任务并行时必须逐个点开查看 |
| 4 | **无持久 WebSocket** | `EventStream` 每次发送新建、turn 结束即关（`SessionDetailViewModel.swift`） | 别端发起的任务、后台任务的事件全部丢失；图片能力无法预判（`Attachment.swift:9-17` 已自注） |
| 5 | **Tab 结构服务器中心** | `RootView.swift:101-113`：Servers / Threads / Settings | 服务器是环境配置不是日常目的地；首跳浪费在选服务器上 |
| 6 | **Settings 是空壳且 iPad 不可达** | `SettingsView.swift` 仅静态卡片；iPad split view 无 Settings 入口 | 平台一致性缺陷 |
| 7 | **无缓存、无深链、无推送、无 App Icon** | 启动全量拉取；无 `onOpenURL`/`CFBundleURLTypes`；`project.yml:35` 显式去掉了 AppIcon | 冷启动白屏等待；通知体系无落点；不可上架 |

| 8 | **工具调用全部扁平化为文本** | `ContentBlockView.swift`：通用 chip + 等宽文本 | diff、命令执行、文件读写没有差异化呈现，读结果效率低 |

---

## 3. 设计原则

1. **Glanceable first**：打开 App 3 秒内知道所有 agent 在干嘛、有没有在等我。
2. **一次注意力完成一个动作**：审批从状态条 / 活动页 /（将来）锁屏通知一步完成，不强制进入会话上下文。
3. **系统组件优先**：Tab role `.search`、`tabViewBottomAccessory`、`tabBarMinimizeBehavior`、swipe actions、context menu、`.searchable`(regular)——能用系统的不自造（现有自制沉浸式搜索将被系统搜索 Tab 取代）。
4. **内容即结构**：assistant 输出全宽 transcript；工具调用按 kind 分化为结构化卡片；diff 是一等公民。
5. **离线可读，在线可控**：缓存先渲染，网络到达后刷新；断连永远有明确状态与恢复路径。
6. **协议诚实**：UI 只承诺 API 能兑现的（`set_mode` 是 enqueue 不是 applied；探针选项是默认值不是会话态——既有 memory 中的契约规则全部沿用）。

---

## 4. 信息架构

### 4.1 Before → After

```
Before (iPhone)                        After (iPhone)
┌──────────────────────────┐           ┌────────────────────────────────────────┐
│ Servers │ Threads │ 设置 │           │ 会话 │ 项目 │ 活动(角标) │ 设置 │ 🔍  │
└──────────────────────────┘           └────────────────────────────────────────┘
  选服务器 → 跳 Threads → 进会话            ▲ tabViewBottomAccessory：正在运行条
                                       默认落在「会话」，服务器切换在标题菜单
```

```
TabView (iPhone, compact)
├─ 会话 Chats（默认）   NavigationStack: ConversationListView → SessionDetailView
├─ 项目 Projects        NavigationStack: ProjectListView → ProjectDetailView → SessionDetailView
├─ 活动 Activity        NavigationStack: ActivityView → SessionDetailView（角标 = 待审批数）
├─ 设置 Settings        NavigationStack: SettingsView → ServersManageView / 其他子页
└─ 搜索（Tab role: .search，玻璃分离式放大镜）SearchView → SessionDetailView
─ .tabViewBottomAccessory { NowRunningBar() }
─ .tabBarMinimizeBehavior(.onScrollDown)
```

### 4.2 为什么是这四个 Tab + 搜索

- **会话**是高频主场（等价 ChatGPT 的 chat 列表），必须是默认 Tab。
- **项目**对应「在哪个仓库干活」的心智。`list_all_folder_details` 返回的 `color / git_branch / default_agent_type / last_opened_at` 字段目前只被用作一个过滤下拉——升为一级页面后这些数据全部派上用场，且是「新建任务」最自然的入口（Codex mobile 同样以 repo/任务为组织单位）。
- **活动**是 agent 远程控制的差异化核心：审批收件箱 + 运行中 + 最近完成。通用聊天 App 没有这个 Tab，但 codeg 的用户每天最关心的就是它。
- **设置**吸收服务器管理，并修复 iPad 不可达问题。
- **搜索**用 iOS 26 的 `Tab(value:role: .search)`：系统自动把它渲染成 tab bar 右侧分离的玻璃圆钮（Apple Music/Podcasts 范式）。此前 memory 记录过 iPhone 上 `.searchable` 只能落在底部、被迫自造顶部搜索栏的问题——搜索 Tab 是 iOS 26 的正解，自造的约 120 行沉浸式搜索代码可以删除。

**为什么不是抽屉**（ChatGPT/Gemini 路线）：抽屉服务「单会话沉浸 + 偶尔翻历史」；codeg 是「多会话并行监控」，活动/项目/会话需要并列的常驻入口。且 iOS 无原生抽屉组件，自造抽屉与 Liquid Glass 导航体系冲突。

**为什么不是 5 个内容 Tab**：「新建」不占 Tab（它是动作不是地点，用 toolbar「+」+ 项目页大按钮承载）；「服务器」不占 Tab（见下）。

### 4.3 服务器的降级处理

- **会话页大标题 = 当前服务器名**，标题右侧 chevron 下拉菜单：切换服务器（带状态点）/「管理服务器…」。范式同 ChatGPT 顶部账号切换、Slack workspace 切换。
- **设置 → 服务器**：完整管理页（现 `ServerListView` 改造，去掉 `onSelect` 跳转职责，只留增删改/测试连接）。
- **首启无服务器** → 全屏 Onboarding（图标 + 一句话价值主张 + 「添加服务器」表单 + 测试连接），完成后落在会话 Tab。取代现在「空 Servers 列表」的冷启动。
- 多服务器是少数派场景：默认单服务器用户感知不到这层，多服务器用户两次点击可切换。

### 4.4 NowRunningBar（tabViewBottomAccessory）

「正在运行」状态条，悬浮于 tab bar 之上，随 `tabBarMinimizeBehavior` 形变（用 `@Environment(\.tabViewBottomAccessoryPlacement)` 适配 inline/expanded 两态）：

- **0 个运行中**：不显示。
- **1 个**：`[agent 图标] 修复登录超时 · Editing AuthService.swift…`，尾部 LivePulse；tap → 直达该会话。
- **多个**：`[图标叠放] 3 个任务运行中`；tap → 活动 Tab。
- **有待审批**：变 amber 强调色，`⚠ 等待批准 · npm install …`；tap → 直达审批卡片。
- 状态动词来自事件流（`tool_call.title` / `status_changed`），与会话详情内 compose 上方的状态行同源。

这是「agent 在后台干活」心智的最佳原生表达——同 Apple Music 的 now-playing 条。演进：M1 先基于「本设备发起的流式会话」（现有状态提升到 App 级即可），M4 SessionHub 上线后覆盖所有端发起的任务。

### 4.5 路由统一

所有入口（列表、搜索、活动、状态条、深链、将来的推送）汇聚到同一路由：

```swift
enum Route: Hashable {
    case conversation(id: Int)
    case project(id: Int)
    case approval(requestID: String)   // 落到所属会话并滚动到审批卡
    case serversManage, settings(SettingsPage)
}
```

URL scheme `codeg://`（`CFBundleURLTypes` 进 `project.yml`）：`codeg://conversation/123`、`codeg://approvals`。M0 就位，为推送/快捷指令/Widget 铺路。

---

## 5. 各 Tab 详细设计

### 5.1 会话 Chats

- **导航栏**：大标题 = 服务器名（menu 见 §4.3）；`topBarTrailing` =「+」新建（主 CTA，`.glassProminent`）。
- **列表结构**：
  - Section「正在运行」（status == in_progress）：置顶，行内 LivePulse + 实时状态动词。
  - 其余按 `updated_at` 倒序；现有 folder 玻璃下拉保留（位于 `safeAreaInset(edge: .top)`，规则见 memory：不加 `.bar` 背景）。
- **会话行**（现 `SessionRow` 升级）：`AgentIcon`（带 per-agent accent 描边）+ 标题（`title` 为空时回退首条用户消息截断，再回退 "New session"）+ 第二行 `项目名 · branch` + 尾部状态徽章/相对时间；本地未读点（read-state 存 UserDefaults，服务端无此概念）。
- **交互**：swipe leading = pin（本地置顶）；swipe trailing = 归档/删除（**依赖服务端路由，见 §11-B④，路由确认前不上**）；长按 context menu（pin / 在项目中查看 / 复制标题）+ 预览（最近 3 条消息摘要）。
- **空态**：无会话 → 居中插画 +「开始第一个任务」按钮；服务器不可达 → 现有 `ColumnPlaceholder` 风格 + 重试。
- **删除项**：自制沉浸式搜索（搜索图标、全宽玻璃搜索场、`searchPresented` 状态机）整体移除，全局搜索归搜索 Tab。

### 5.2 项目 Projects

- **数据**：`list_all_folder_details`（已有 API，零服务端改动）。
- **项目行**：`color` 圆点 + 名称 + 路径末段（`~/work/codeg`）+ `git_branch` 徽章 + 默认 agent 小图标 + 运行中数量角标；按 `sort_order`，次序 `last_opened_at`。
- **项目详情**：头部卡（完整路径 / branch / 默认 agent）→ **「新建任务」大按钮**（带默认 agent 图标，直接进入 §6.1 流程且项目预选）→ 该项目会话列表（`listConversations(folderIds: [id])`，复用会话行组件）。
- **空态**：服务端无 folder → 说明文案「在桌面端 codeg 中添加项目后即可在此发起任务」。

### 5.3 活动 Activity

agent 远程控制的核心页。三个 Section：

1. **待审批**（角标来源）：`ApprovalRow` = agent 图标 + 工具名 + 命令/路径预览（等宽字体，2 行截断）+ 内联 `允许 / 拒绝` 按钮（更多选项进会话）。行 tap → 会话内审批卡。
2. **正在运行**：会话行 + 实时状态动词 + 已运行时长；尾部「停止」context 动作（`acp_cancel`）。
3. **最近**（24h）：完成（✓ + duration + token 用量）/ 失败（红，error message 首行）/ 已取消。tap 进会话。

- **空态**：「所有 agent 空闲」+ 上次活动时间。
- **演进**：M1–M3 降级版 = 本设备发起的活动 + 下拉刷新（`listConversations(status: in_progress)` 轮询）；M4 SessionHub 后全量实时。Tab 角标用 `.badge(pendingApprovals.count)`。

### 5.4 设置 Settings

Sections：**服务器**（管理页入口，行内显示当前服务器与状态点）/ **通知**（M5：审批提醒、完成提醒开关）/ **默认值**（默认项目、默认 agent——新建表单的初值）/ **外观**（暂只深色，预留）/ **数据**（缓存占用 + 清空）/ **关于**（版本、服务端版本 via `health`、开源链接）。

iPad：sidebar 底部 gear 打开 Settings sheet（修复不可达）。

### 5.5 搜索 Search（Tab role .search）

- 进入即聚焦系统搜索场；**服务端搜索**：`listConversations(search:)` 防抖 300ms（服务端已支持，现在完全没用上），结果按项目分组；scope 条：全部 / 当前项目；本地「最近搜索」（UserDefaults，10 条）。
- 搜索结果行复用会话行组件，命中词高亮。
- 空态：最近搜索 + 快捷过滤 chips（按 agent / 按状态）。

---

## 6. 核心交互流程（协议级）

### 6.1 新建会话（目前完全缺失，最高优先级流程）

入口：会话页「+」/ 项目详情大按钮 / 空态 CTA / 搜索空态。

```
NewSessionSheet（.medium → .large detent）
├─ 项目选择器（横向卡片，last_opened_at 排序，预选最近/来源项目）
├─ Agent 选择器（6 个品牌图标横排，预选 folder.default_agent_type ?? 设置默认值）
└─ 消息输入框（复用 ComposeBar 精简版：文本 + 附件）
   [开始任务] →
   1. acp_connect{agentType, workingDir: folder.path}        → connectionId
   2. EventStream attach，等 snapshot 确认（attach-before-prompt 既有规则）
   3. acp_prompt{connectionId, blocks, folderId}
   4. 立即 push SessionDetail（乐观 UI：pending user turn + 流式）
   5. 收 conversation_linked{conversation_id} → 绑定正式 ID，会话列表插入新行
```

要点：**不在此流程调用 `acp_describe_agent_options`**（探针要起一个一次性 agent，最长 60s）——模式/配置调整留在会话内的现有 Options sheet；`title` 为 null 期间显示首条消息截断。失败路径：connect 失败 → sheet 内错误不关闭；attach 无确认 → 按既有规则判定挂死、不盲发。

### 6.2 审批（核心新流程）

```
WS: permission_request{request_id, tool_call, options}
 ├─ ① 会话 transcript 内插入 ApprovalCard（流式中内联出现）
 ├─ ② 活动 Tab 收件箱 + Tab 角标 + ③ NowRunningBar 变 amber
 └─ （M5）④ 可操作推送通知（允许/拒绝按钮，锁屏直接处理）

ApprovalCard
┌──────────────────────────────────────┐
│ ⚠ Claude Code 请求执行                  │
│ ▸ Bash · npm install lodash           │  ← kind 图标 + title + raw_input 预览
│   （可展开完整命令 / diff）              │
│ [ 仅本次允许 ] [ 始终允许 ] [ 拒绝 ]      │  ← 按钮组直接映射 options[].option_id
└──────────────────────────────────────┘
```

- 用户选择 → 调用审批响应路由（**路由名待确认**，见 §11-A①；web 客户端已实现，照抄其语义）→ 卡片折叠为结果行（「已允许 · 刚刚」）。
- 响应失败 → 卡片恢复可点 + 错误 toast；他端已处理 → 收到后续 `tool_call_update` 自动消解；会话被 cancel → 清理队列。
- 多个 pending 顺序排队，活动页可批量逐个处理。

### 6.3 发送 / 流式 / 重连（现有机制的增量）

- 保留：generation 计数防串流、乐观 user turn、`turn_complete` 后 `refreshAfterTurn` 校正、stop 按钮。
- 新增：**near-bottom 检测**——只有用户在底部附近才自动跟随滚动；上翻阅读时不打断，改为右下角「↓ 新消息」玻璃 pill（点击回到底部）。这是现有 `scrollTick` 无条件滚底的最大体验问题。
- 新增：`turn_complete` 成功触感（`.success`）、error 触感、审批出现 `.warning` 触感。
- 断连横幅统一组件：「连接已断开，正在重连…」（amber）→「已恢复」（绿，2s 自动消失）。

### 6.4 跨入口一致性

推送 / URL / 状态条 / 搜索 / 活动，全部走 `Route.conversation(id)` 落到同一详情；`Route.approval` 额外滚动定位到审批卡。深链在目标 Tab 的 `NavigationPath` 上重建栈（如 `codeg://conversation/123` → 切到会话 Tab + push 详情），保证返回手势语义正确。

---

## 7. 会话详情 v2

布局骨架不变（transcript + `safeAreaInset` compose bar，修饰符顺序遵守 memory 中的既定规则），内容层升级：

### 7.1 Transcript 渲染

- **角色样式保留现状**（业界一致）：user = 右对齐玻璃气泡；assistant = 全宽 transcript 列 + 头像；system = 居中弱化。
- **ToolCallCard 取代通用 ToolChip**，按 ACP `kind` 分化：

| kind | 呈现 |
|---|---|
| `read` / `search` / `fetch` | 单行紧凑 chip：图标 + 「读取 main.swift」+ 状态点（保持现有密度） |
| `edit` | **DiffCard**：文件名 + 语言图标 + `+12 −3` 统计；tap → 全屏 DiffViewerSheet（行号、红/绿行底色、等宽、横向滚动、多文件切换、字号调节）。数据优先取 `tool_call.content` 的结构化 diff（**待确认**，§11-A②），缺失时回退 raw_input/output 文本 |
| `execute` | **TerminalCard**：深色圆角块、`$ 命令` 头行、输出等宽流式追加（`raw_output_append`）、退出码徽章、>280 字符折叠（沿用现有 More/Less） |
| 其他/未知 | 现有通用 chip 兜底 |

- **连续工具调用聚合**：≥3 个连续完成的工具卡折叠为「执行了 N 个操作 ▸」组（默认收起，流式中的最后一个始终展开）。解决长 agent 回合的滚动疲劳。
- thinking 折叠块、图片内联、Markdown 渲染均保留。

### 7.2 头部与信息面板

- inline 标题 = 会话名；副标题行 = `项目 · branch · agent`。
- **tap 标题 → SessionInfoSheet**：项目/路径、agent + model + 当前 mode（来自 live snapshot，探针仅兜底——既有规则）、**上下文用量环**（`sessionStats.contextWindowUsagePercent`，数据已有但现在只是页脚小字）、token 统计、导出 Markdown、断开连接（cancel）。
- 重命名/删除入口预留，依赖 §11-B④ 路由。

### 7.3 Compose

现有能力全保留（+ 菜单：附件/快捷消息/专家/斜杠命令；agent options；stop）。增量：

- M4 Hub 后用实时 `prompt_capabilities` 预先禁用图片按钮（解决 `Attachment.swift:9-17` 自注的 deferred 问题）。
- 待审批时 compose 上方插入一条 amber 提示行（tap 滚到审批卡），输入仍可用（用户可边回复边决定）。

### 7.4 长会话性能

`get_folder_conversation` 无分页（§11-B⑤）。客户端先行方案：初始只渲染最近 50 turns，顶部「加载更早 ▸」按钮逐段展开；`LazyVStack` + 既有 bottomAnchor 滚动机制不变。

---

## 8. iPad / 多窗口

- **保留三栏 `NavigationSplitView`**（已验证形态，符合 iPad 规范），但 sidebar 重定义——从「服务器列表」改为**来源列表**：
  - 顶部：服务器切换 menu（同 iPhone 标题菜单）
  - Section 列表：会话（全部/运行中）、项目（逐项列出，带运行角标）、活动（带审批角标）
  - 顶部 `.searchable`（regular 宽度下系统放在正确的顶部位置，无需自造）
  - 底部：gear → Settings sheet
- content 列 = 所选来源的列表；detail = 会话详情。`ColumnPlaceholder` 风格保留。
- 键盘：`⌘N` 新任务、`⌘F` 搜索、`⌘1–4` 切来源、`↑↓` 列表移动、`⌘.` 停止当前 turn。pointer hover 行高亮。
- 多窗口：`UIApplicationSupportsMultipleScenes` 已开启；会话拖出独立窗口作为 M5 可选项（scene 级路由）。

---

## 9. 视觉与设计系统

**保留**：深色基调、mint accent、`CodegBackground` 双色光晕、玻璃组件族、6 个 agent 品牌矢量图标、scroll-edge 自动磨砂规则集（memory `ios26-nav-chrome` 全部沿用）。

**Token 增量**（`Theme.swift`）：

- 状态色族：`running`（accent）、`waiting`（amber `#FFB454` 系）、`failed`（danger）、`succeeded`（secondary 绿）。
- Diff 色：加行 `rgba(46,160,67,0.18)` 底 + 绿字，删行红系同理（深色底调校）。
- per-agent accent（`AgentType.accent` 已有）扩展到行描边、审批卡左缘条。

**新组件清单**：`NowRunningBar` / `ApprovalCard` / `DiffCard` + `DiffViewerSheet` / `TerminalCard` / `ToolCallGroup`（聚合折叠）/ `ScrollToBottomPill` / `ConnectionBanner` / `OnboardingView` / `ServerSwitcherMenu` / `ContextUsageRing`。

**App Icon（生产阻塞项）**：目前没有图标（`project.yml:35` 显式置空）。需设计深色玻璃风 codeg 字形图标，提供 default / dark / tinted 三变体，恢复 `ASSETCATALOG_COMPILER_APPICON_NAME`。

**动效**：系统弹簧默认值；`LivePulse` / shimmer / 状态条形变全部尊重 Reduce Motion。

**浅色模式**：非本期目标（入口 `CodegiOSApp.swift` 硬编码 `.preferredColorScheme(.dark)`）。token 已语义化，将来成本可控；主流 AI App 同样深色主打。

---

## 10. 技术架构重构

### 10.1 路由层（M0）

```
App/
├─ CodegiOSApp.swift      + .onOpenURL → NavModel.handle(url)
├─ NavModel.swift          @Observable：selectedTab、每 Tab 一条 NavigationPath、route(to:)、handle(url:)
├─ Route.swift             §4.5 的枚举 + URL 双向转换
└─ RootView.swift          五 Tab 壳 + .navigationDestination(for: Route.self) 集中注册
```

`AppModel` 拆分：导航态入 `NavModel`；`selectedServerID` / `serverStore` 保留为 `AppEnvironment`（后续 `SessionHub`、`CacheStore` 同挂于此，经 `.environment()` 注入）。现有 `.id("server|conversation|token")` 防串流重建策略保留。

### 10.2 SessionHub（M4，最大架构变更）

```swift
@MainActor @Observable
final class SessionHub {
    // 每服务器一条持久 EventStream，多路 attach（subscription_id ↔ connection_id）
    private(set) var liveStates: [Int: LiveSessionState]   // conversationID → 流式态（LiveTurn 提升至此）
    private(set) var approvals: [ApprovalRequest]           // 全局审批队列
    private(set) var running: [Int]                         // 运行中会话（有序）
    var connection: ConnectionPhase                         // idle/connecting/live/degraded

    func attach(conversationID: Int) async   // 引用计数：详情页在看 / 状态条 / 活动页共享同一订阅
    func detach(conversationID: Int)
    func send(prompt:to:) / cancel(_) / respond(approval:option:)
}
```

- 生命周期：`scenePhase` background → 保留内存态、关 socket；foreground → 重连 + `attach(since_seq:)` 续传（事件缝隙由 replay 补齐）。`connection_gone` 重连一次、`lagged` 走 re-attach-with-since_seq——既有契约规则照搬。
- **`SessionDetailViewModel` 瘦身**：654 行中约 300 行流式状态机迁入 Hub，VM 只剩加载/发送编排/UI 态。
- **渐进迁移**：第一步 Hub 仅托管详情页 socket（行为等价、mock 回归）；第二步开放多路给活动页/状态条/角标。两步分开提测。

### 10.3 缓存层（M5，可提前）

`CacheStore`（actor）：`Application Support/codeg-cache/<serverID>/` 下 `conversations.json` + `transcripts/<id>.json`（LRU 50）。列表与详情 stale-while-revalidate：缓存命中先渲染（标「更新中」微状态）→ 网络返回后 diff 刷新。模型均 `Codable`，不引入 SwiftData（将来要全文搜索再评估）。设置页可查看占用/清空。

### 10.4 质量基建

- 并发：`SWIFT_STRICT_CONCURRENCY: minimal` → `targeted`（M4 同步进行，Hub 是最受益方）。
- 测试（现状为零）：① `JSONCoding` / `AcpEvent` / `WireRequests` 解码 fixture 单测（双 casing 与 unknown 兜底是高危区）；② `EventStream` attach 状态机测试（ready→attach→snapshot→event→detached 各分支）；③ mock server + `SIMCTL_CHILD` seed 截图脚本（memory `keychain-and-verification` 中已有基础）跑关键路径冒烟。
- 本地化：硬编码英文文案收敛进 String Catalog（en + zh-Hans）。
- 可达性：审批按钮/工具卡 VoiceOver 标签、Dynamic Type 全档检查（等宽预览块允许 `.monospaced` 缩放）、Reduce Motion。

---

## 11. 服务端配套清单

**A · M3 前必须确认**（去 `github.com/xintaofei/codeg` `src-tauri/src/web/` 对照 web 客户端）：

1. `permission_request` 的**响应路由**（名称/参数/语义，web 端已实现审批，必然存在）。
2. `tool_call.content` 是否携带**结构化 diff**（ACP 规范支持 `{type:"diff", path, oldText, newText}`）；决定 DiffCard 数据源还是回退文本。
3. 新建流程中 `conversation_linked` 的时序保证（prompt 后多快到达；到达前列表如何占位）。

**B · 建议服务端新增**（不阻塞 M0–M4）：

4. 会话 rename / archive / delete 路由（解锁 swipe 动作与信息面板入口）。
5. `get_folder_conversation` 分页（长会话；客户端折叠方案先顶）。
6. 推送通道：自托管场景 APNs 不易（需开发者账号+签名），备选 ntfy/webhook → 本地中继；或服务端轮询令牌方案。审批推送是 Codex mobile 级体验的最后一块。
7. 全局活动查询（`since` 参数的事件历史），避免活动页冷启动靠轮询拼装。

**C · 已确认可用（零改动）**：列表 `search`/`status`/`folderIds` 参数、folder 全字段、live/by-conversation snapshot、quick messages / experts / slash commands。

---

## 12. 里程碑

依赖关系：`M0 → M1 → {M2 ∥ M3} → M4 → M5`。每个里程碑独立可合、可演示、可回归（mock server 冒烟脚本随 M0 建立）。

| | 范围 | 主要触点 | 体量 | 验收标准 |
|---|---|---|---|---|
| **M0 路由地基** | `Route`/`NavModel`/五 Tab 枚举、`codeg://` scheme、mock 冒烟脚本 | `App/*`（~240 行重写）、`project.yml` | S | 现功能零回归；`codeg://conversation/N` 冷启动直达；xcodegen 后构建过 |
| **M1 新 Tab 壳** | 五 Tab + 搜索 role、服务器降级（标题菜单 + 设置内管理 + Onboarding）、Settings 充实 + iPad 可达、移除自制搜索、NowRunningBar v1（本设备流式会话） | `RootView`、`SessionListView`（-120 行搜索）、`ServerListView`、`SettingsView`、新 `SearchView`/`OnboardingView`/`NowRunningBar` | M | 首启 → 添加服务器 → 会话列表 ≤3 步；切服务器 2 tap；iPad Settings 可达；搜索走系统 Tab 且服务端 search 生效 |
| **M2 新建会话 + 项目** | `NewSessionSheet`、`ProjectListView`/`ProjectDetailView`、默认值设置项 | 新 Feature 目录 ×2、`CodegClient` 微调 | M | 冷启动 → 发出新任务 ≤4 tap；connect/attach 失败路径有明确错误；`conversation_linked` 后列表出现新行 |
| **M3 会话详情 v2** | ToolCallCard 族（Diff/Terminal/聚合）、ApprovalCard（路由确认后接通）、near-bottom 滚动 + pill、SessionInfoSheet、上下文环 | `ContentBlockView`/`TurnView`/`LiveTurnView`/`TranscriptView`、新组件 ×6 | L | diff 可全屏审阅；审批从出现到响应 ≤2 tap；流式中上翻不被拽回底部 |
| **M4 SessionHub** | 持久 socket 多路复用、活动 Tab 实时化、角标、NowRunningBar 全量、前后台生命周期、strict concurrency targeted | `EventStream`、新 `SessionHub`、`SessionDetailViewModel` 瘦身、`ActivityView` | L（最高风险） | 两会话并行各自实时；锁屏 1 分钟回来自动续传无缝隙；断网横幅 + 自动恢复；他端发起的任务出现在活动页与状态条 |
| **M5 生产打磨** | App Icon、缓存层、触感、a11y pass、String Catalog、推送（视 §11-B⑥）、Live Activity（可选）、长会话折叠 | 全局 | M | 冷启动缓存先渲染 <500ms 出内容；VoiceOver 走通审批流；图标/上架材料齐备 |

---

## 13. 风险与权衡

1. **SessionHub 重写流式管线**（最高风险）：现有 654 行状态机经过实战调校（generation 防串流、乐观 turn 校正）。缓解：两步等价迁移（§10.2）、迁移前补 `EventStream` 状态机测试、mock 回归脚本先行（M0 交付物）。
2. **审批响应路由未确认**：§11-A① 是 M3 的硬前置。缓解：ApprovalCard 做成 options 驱动的协议无关组件，路由层一处接入；最坏情况降级为「跳转桌面端处理」的提示卡（仍比现状的不可见强）。
3. **iOS 26 新 API 成熟度**（`tabViewBottomAccessory`、`Tab role .search`、accessory 与 `.toolbar(.hidden, for: .tabBar)` 在 push 详情时的共存行为）：M1 第一周真机/模拟器验证清单先行，发现坑有降级方案（accessory → 列表页顶部内联条）。
4. **单 socket 多 attach 的服务端承载**：订阅数 = 运行中会话数，量级很小，但 `lagged` 处理路径要按契约实现。灰度：详情订阅优先，活动页订阅上限 10。
5. **范围蔓延**：本方案触达约 1/3 代码。约束手段就是里程碑切分——每个 M 独立合入，任何时点停下都是一个更完整的 App。

## 14. Non-goals

- 移动端 IDE / 文件树浏览 / 直接改代码（Codex mobile 同样明确不做——"remote control, not a mobile IDE"）。
- 通用 LLM 聊天（人格、跨设备账号体系、消息云同步——codeg-server 是单一事实源）。
- Android / 跨平台。
- 本期不做：浅色模式、watchOS、桌面 Widget（Live Activity 在 M5 作为可选项评估）。
