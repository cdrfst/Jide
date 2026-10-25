# Jide（记得）

个人任务管理 + 到期提醒应用，参考 TickTick（滴答清单）的功能，但不是其衍生品，名称与图标避免与其混淆。
本文件只放每次会话都要知道的内容；细节在 `docs/` 下，需要时再读。

## 开始工作前先读

1. `docs/PROGRESS.md`：当前阶段、已完成、下一步、阻塞项
   `openspec list` 与 `openspec/specs/`：需求现状与进行中的变更
2. `docs/DECISIONS.md`：已定决策及原因（不要重新讨论已定的事，除非用户提出）
3. `docs/DESIGN.md`：总体设计与选型背景（按需读）

## 已定决策（摘要）

- 技术栈：Flutter + drift（SQLite）+ riverpod，路由 go_router，重复规则 rrule，本地通知 flutter_local_notifications。
- 平台顺序：Windows 桌面 → 安卓 → iOS（以后）。
- 离线优先，本地库为准，之后再做增量同步。
- 后期要做 AI 自然语言生成任务：API Key 不得放客户端，经自建后端代理调用。
- 第一阶段（MVP）只做：任务与清单、子任务、标签、优先级、截止时间、RRULE 重复、本地提醒、智能清单。
  暂不做：日历、看板、四象限、番茄钟、习惯、同步、协作、AI。

## 设计约束

1. 平台相关代码只放 `lib/platform/`，业务层依赖接口，不直接调用平台 API（为 iOS 预留）。
2. 输入解析统一走 `TaskParser` 接口，产出 `TaskDraft`，用户确认后才入库；实现为 `RuleBasedParser`、`AiParser`、`HybridParser`；需带 `ParseContext`（当前时间、时区、已有清单和标签）。
3. 时间存 UTC 并保存时区；全天任务只存日期。
4. 软删除（`deleted`），记录 `version` 与 `updated_at`，为同步预留。
5. 重复任务：完成时生成下一条，不预生成；日历里只临时展开；区分按截止日与按完成日。
6. 提醒只注册最近 N 条（iOS 上限 64），启动与同步后重排。
7. 排序用分数或 LexoRank。
8. 连续打卡等统计实时计算，不存字段。

## 工作规则

**收工规则（每完成一项工作都要做）：**
- 更新 `docs/PROGRESS.md`（已完成、下一步、已知问题）。
- 做了重大决策，追加到 `docs/DECISIONS.md`（只追加，不改历史；写明日期、决策、原因、放弃的方案）。
- 提交 git，提交信息说明做了什么。

**需求变更规则：**
- 需求的新增、修改、推翻，必须先告知用户并获批准，才能写入规格或文档；不得把讨论中的设想当成定论。
- 推翻或修改需求时，同时列出需要清理或调整的代码和测试，并写明原因。
- 需求管理使用 OpenSpec（已采用，见 D-009）。规格在 `openspec/specs/`（当前状态），变更在 `openspec/changes/<名>/`（proposal、design、tasks、增量规格），完成后归档。
  - 流程：`/opsx:propose` → 用户确认 → `/opsx:apply` 实现 → `/opsx:archive` 归档并合并回主规格。
  - 增量规格用 ADDED / MODIFIED / REMOVED / RENAMED。MODIFIED 是整块替换，必须复制完整原需求（含全部场景）再改。
  - 改完用 `openspec validate <变更名> --strict` 校验。规格正文用中文，Requirement / Scenario / SHALL / WHEN / THEN 关键字保持英文。
  - 被推翻需求的原因不会留在主规格里，需同时追加到 `docs/DECISIONS.md`。

**环境与质量：**
- 开始写代码前先检查环境（Flutter 是否已安装），不要假设。
- Windows 安装包必须在 Windows 本机编译；WSL 适合写纯 Dart（core、data、解析器、测试）。
- 重复规则与解析器必须有单元测试，测试是客观的"做对了"标准，不要自己宣布完成。
- 不要自动合并到主分支；不要把密钥写进仓库。

## 目录

```
lib/core/       时间工具、RRULE 封装、自然语言解析（纯 Dart）
lib/data/       db（drift）、repo、sync
lib/features/   task、smart_list、calendar、board、focus、habit
lib/platform/   notify、android、windows、（ios）
lib/ui/         响应式布局：手机单栏 / 桌面三栏
test/           重点测试 RRULE 与解析器
docs/           DESIGN / DECISIONS / PROGRESS
openspec/       需求规格与变更（OpenSpec）
```
（以上为规划，实际创建情况以仓库为准。）

## 协作约定

- 回复使用中文。
- 本项目文档是内部开发文档，不适用全局"公文/汇报文档"规范。
