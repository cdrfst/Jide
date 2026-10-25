# Jide 设计文档

> 整理自 2026-10-25 的选型讨论。结论摘要见根目录 `CLAUDE.md`。
> 竞品参考：TickTick（滴答清单）。其为独立产品（2013 年推出），并非谷歌应用，仅在 Google Play 上架。
> 价格与免费版限制来自网络搜索摘要，未核对原文，仅作参考。

## 1. 产品定位

个人任务管理 + 到期提醒。核心价值：把事情记下来，到点提醒你。
后期通过 AI 接口支持一句话自然语言生成任务。

### TickTick 功能参考

| 模块 | 功能 |
|---|---|
| 任务管理 | 清单、子任务、标签、优先级、重复任务、提醒；自然语言时间识别 |
| 视图 | 列表、看板、时间线、四象限 |
| 日历 | 任务与日历事件同屏，可拖拽排时间，可订阅外部日历 |
| 专注 | 番茄钟、白噪音、专注统计 |
| 习惯 | 每日习惯打卡、连续天数 |
| 协作 | 共享清单（每清单最多 29 人）、指派、评论，无团队工作流 |
| 平台 | iOS、Android、Mac、Windows、Web、Apple Watch、浏览器插件 |
| 定价 | 免费版有清单/任务数量限制；高级版约 35.99 美元/年（各来源有出入） |

优势：功能全、价格低、任务与日历同屏。不足：协作弱、第三方集成少。

## 2. 功能模块与优先级

| 优先级 | 模块 |
|---|---|
| P0（MVP） | 任务与清单；重复任务与提醒；列表视图与智能清单 |
| P1 | 自然语言输入；日历视图；番茄钟；习惯打卡；看板与四象限 |
| P2 | 多端同步（离线可用）；共享协作；外部日历订阅；统计报表 |
| 后期 | AI 自然语言生成任务；语音输入；AI 任务拆解与重新排期 |

四象限不需要单独建表，是按 `priority` 与截止日期分组；看板只多一个 `column_id`。

## 3. 数据模型（草案）

```sql
list(id, user_id, name, color, sort_order, kind['task'|'note'], folder_id, updated_at, deleted)

task(
  id, user_id, list_id, parent_id,          -- 子任务用 parent_id 自关联
  title, content,
  priority,        -- 0无 1低 3中 5高
  status,          -- 0未完成 2已完成 -1放弃
  start_at, due_at, is_all_day, timezone,
  repeat_rule,     -- RRULE，如 FREQ=WEEKLY;BYDAY=MO
  repeat_from,     -- 'due' 按截止日 | 'completed' 按完成日
  reminders,       -- JSON，如 ["TRIGGER:-PT15M"]
  tags, column_id, sort_order, completed_at,
  version, updated_at, deleted
)

focus_session(id, user_id, task_id, start_at, end_at, duration_sec, type['pomo'|'stopwatch'], status)
habit(id, user_id, name, icon, goal_type['bool'|'count'], goal_value, frequency_rule, reminder_times)
habit_checkin(id, habit_id, date, value, status)   -- (habit_id, date) 唯一
```

## 4. 关键技术点

### 4.1 重复任务（最易出错）
- 规则使用 RFC 5545 RRULE，不自己设计；Dart 用 `rrule` 包。
- 完成时标记当前条并生成下一条，不预生成未来实例。
- 日历视图里的未来重复任务在前端临时展开，不入库。
- 边界：每月 31 号、夏令时、跳过某一次、"仅本次"与"本次及以后"。

### 4.2 自然语言解析
- 中文需自写规则解析器，例："下周三下午3点 提交报告 !高 #工作"
  - 时间词典：今天/明天/后天/下周X/X月X日/X点/上午下午/每天/每周X/工作日
  - 符号：`!优先级`、`#标签`、`^清单`，剩余文本为标题
- 输入框实时高亮识别部分，允许撤销识别。
- 本地规则解析是 AI 的离线降级方案，必须保留。

### 4.3 提醒
- 移动端用系统本地通知，离线可用。
- 安卓 12+ 需精确闹钟权限（`SCHEDULE_EXACT_ALARM` / `USE_EXACT_ALARM`）；引导关闭电池优化；监听 `BOOT_COMPLETED` 重新注册。
- 国产手机后台限制严，需引导开启自启动；国内收不到 FCM，主要靠本地通知，服务端推送后续接厂商通道或第三方推送。
- iOS 本地通知上限 64 条：只注册最近 N 条，启动时重排。
- 服务端补偿推送：Redis ZSET 按触发时间存储 + 定时扫描。

### 4.4 多端同步
- 本地库优先 + 增量同步：`POST /sync {since, changes}`，按 `version`/`updated_at` 比较，字段级最后写入为准，返回他端变更与新同步点；WebSocket 通知"有更新"。
- 待办场景很少多人同改同字段，不必上 CRDT。
- 排序用分数/LexoRank。

### 4.5 番茄钟
- 状态机：空闲 → 专注 ⇄ 暂停 → 短/长休息 → 空闲。
- 记录开始时间戳，用当前时间推算剩余时间，不依赖后台计时器累加。
- 安卓需前台服务（Android 14 声明 `foregroundServiceType`，可用 `specialUse`）；iOS 靠本地通知。

### 4.6 习惯打卡
- 连续天数由 `habit_checkin` 实时计算；频率规则复用 RRULE。

### 4.7 日历视图
- 同一天时间重叠的任务用贪心分列算法并排显示。
- 外部日历：ICS 解析、CalDAV 双向同步、Google Calendar API。

## 5. 技术选型

### 5.1 最终方案
| 层 | 选型 | 作用 |
|---|---|---|
| 界面 | Flutter（Dart） | 自绘引擎，一套代码覆盖 Windows/安卓/iOS |
| 本地数据库 | drift + sqlite3_flutter_libs | 类型安全查询、事务、迁移、响应式查询 |
| 状态管理 | flutter_riverpod | 状态与依赖注入，便于测试 |
| 路由 | go_router | |
| 重复规则 | rrule | |
| 通知 | flutter_local_notifications | 两端本地通知 |
| 日历 | table_calendar（月视图），周/日时间轴自写 | |
| 拖拽 | ReorderableListView / drag_and_drop_lists | |
| Windows 专用 | window_manager、tray_manager、hotkey_manager、launch_at_startup | 窗口、托盘、全局快捷键、开机自启 |
| 安卓专用 | home_widget | 桌面小组件 |
| 网络 | dio + web_socket_channel | |

数据流：Widget → riverpod Provider → Repository → drift(SQLite)；drift 响应式查询驱动界面刷新。

### 5.2 候选方案对比（含 iOS 需求）
| 方案 | 安卓 | Windows | iOS | 结论 |
|---|---|---|---|---|
| Flutter | ✅ | ✅ | ✅ | **选用** |
| KMP + Compose Multiplatform | ✅ | ✅（JVM，体积大启动慢） | ✅ | 无 Kotlin 经验，排除 |
| React Native + RN Windows | ✅ | ⚠️ 社区小 | ✅ | Windows 是短板 |
| .NET MAUI | ✅ | ✅ | ✅ | 口碑与稳定性一般 |
| Avalonia / Uno | ✅ | ✅ | ✅ | 移动端生态偏弱 |
| Tauri 2 | ⚠️ | ✅ | ⚠️ | 移动端后台能力弱，不利于提醒 |
| Capacitor + Electron | ✅ | ✅ | ✅ | 体积大、流畅度一般，需两个壳 |
| 双原生 | ✅ | ✅ | ✅ | 成本最高 |

选型关键：待办应用的价值取决于提醒是否准时，需重点看安卓后台能力与 Windows 托盘常驻、全局快捷键。

### 5.3 平台差异要点
- **安卓：** 精确闹钟、前台服务（番茄钟）、开机重注册、国产 ROM 引导、通知栏操作、小组件。
- **Windows：** 通知需应用身份（AUMID），MSIX 打包最省事；关闭窗口最小化到托盘；全局快捷键（如 Ctrl+Shift+A）快速添加；番茄钟置顶小窗；开机自启。
- **iOS（以后）：** 必须有 Mac 打包上架（或云 Mac/CI）；Apple 开发者账号 99 美元/年；通知 64 条上限；无后台常驻；小组件与 Live Activity 须原生 SwiftUI + App Group 共享数据；推送走 APNs；需隐私政策与"删除账号"入口。

## 6. AI 自然语言生成任务（后期）

```
用户输入
 ├─ 本地规则解析器（毫秒级、离线）→ 能解析则直接生成草稿
 └─ 解析不了/复杂（多任务、模糊时间）→ AI 接口 → 结构化任务列表
                         ↓
               用户预览确认 → 写入 drift
```

- 接口：
  ```dart
  abstract class TaskParser { Future<List<TaskDraft>> parse(String text, ParseContext ctx); }
  // RuleBasedParser / AiParser / HybridParser
  ```
- `ParseContext` 带当前时间、时区、已有清单与标签，AI 才能换算"下周三"并归入已有清单。
- 用接口的结构化输出（JSON Schema）或 tool use 约束格式；客户端再校验时间格式、RRULE 可解析性、清单是否存在。
- 示例输出字段：title、due、all_day、priority、repeat_rule、reminders、list、tags、subtasks。
- 模型：短文本抽取，选小而快的模型（如 Haiku 系列）；模型 ID 放服务端配置，不写死在客户端。
- **API Key 不得放客户端**：自建轻量后端代理，负责保管密钥、鉴权限流、用量记录、切换模型；可与同步服务合并。
- 体验：预览确认页（可编辑/删除/全部添加）；先显示本地结果再替换为 AI 结果；断网/超时/额度用完自动降级；语音转文字后复用同一链路。
- 隐私合规：任务内容会发往第三方，需在隐私政策说明并提供开关；仅发送当次输入与必要上下文；国内上架使用境外模型需核实数据出境及生成式 AI 服务相关规定。

## 7. 实施阶段

1. **第一阶段：** Windows 桌面。任务/清单、智能清单、截止时间、RRULE 重复、本地提醒、托盘。数据先存本地。
2. **第二阶段：** `TaskParser` 接口 + 本地规则解析器 + 预览确认页；上线安卓（权限引导、前台服务、小组件）；日历、看板、四象限、番茄钟、习惯。
3. **第三阶段：** 后端代理 + `AiParser` + 混合模式；账号与增量同步。
4. **第四阶段：** 共享协作、外部日历订阅、统计；语音输入、AI 任务拆解与重新排期；iOS。

原则：先把单机版数据模型与重复规则做扎实，再接同步，否则几乎必然重构。

## 8. 命名

- 定名：**Jide（记得）**。"记下来、你要记得"同时覆盖记录与提醒，口语化，也契合"说一句它就记得"的 AI 助手形象。
- 备选：待办铃（Dailing）、到点（Daodian）、说办、Duely、Noted。
- 避免与 TickTick/滴答清单名称、图标相似。
- 上线前核查：应用商店重名、商标类别 9 与 42、域名与包名。

## 9. 开发环境

- Windows 本机安装：Flutter SDK（如 `C:\dev\flutter`，加入 PATH）、Visual Studio 2022 Community（"使用 C++ 的桌面开发"）、Android Studio；运行 `flutter doctor` 全部通过；开启 Windows 开发者模式（插件需要符号链接）。
- 项目目录：`D:\code\diy\Jide`。
- Windows 安装包只能在 Windows 本机编译；WSL 中适合写纯 Dart（core 层、解析器、测试），再由 Windows 编译运行。
- 初始化：`flutter create --org com.jide --project-name jide .`

## 10. 待办 / 下一步

- [ ] 确认骨架生成方式（先手写 `pubspec.yaml` + `lib/`，或等装好 Flutter 后 `flutter create`）
- [ ] 建立 `lib/` 目录结构
- [ ] 定义 `TaskParser`、`TaskDraft`、`ParseContext`
- [ ] 写 drift 数据表（list/task/tag/focus_session/habit/habit_checkin）
- [ ] 封装 RRULE（下一次生成、虚拟实例展开）
- [ ] 中文自然语言解析器 + 单元测试（含每月 31 号、跨夏令时等边界）
- [ ] 应用商店/商标/域名核查
