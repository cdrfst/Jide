# 进度与交接

> 每次收工更新本文件。最近更新：2026-10-26

## 当前阶段

第 0 阶段：选型与项目搭建。尚无代码。

## 已完成

- 2026-10-25 完成技术选型与设计讨论，结论见 `DECISIONS.md`、`DESIGN.md`。
- 2026-10-25 建立项目目录 `D:\code\diy\Jide`（WSL：`/mnt/d/code/diy/Jide`）。
- 2026-10-25 编写 `CLAUDE.md`、`docs/DESIGN.md`、`docs/DECISIONS.md`、`docs/PROGRESS.md`、`README.md`、`.gitignore`，初始化 git 仓库。

- 2026-10-26 实测并正式采用 OpenSpec 1.15.0（D-009），已在项目内初始化（`openspec/`、`.claude/skills`、`.claude/commands/opsx`）。
- 2026-10-26 用中文写出第一个变更 `add-mvp-task-core`（task、repeat、reminder、smart-list 四份增量规格），`--strict` 校验通过。

## 进行中

- 变更 `add-mvp-task-core` 为**草案，待用户确认**后才能 `/opsx:apply`。确认前不要实现，也不要归档。

## 下一步（按顺序）

1. 用户审阅并确认 `openspec/changes/add-mvp-task-core/` 下的规格（如需修改直接说，调整后再校验）。
2. 确认 Flutter 骨架生成方式：先手写 `pubspec.yaml` 与 `lib/`，或装好 Flutter 后 `flutter create --org com.jide --project-name jide .`。
3. 建 `lib/` 目录结构；定义 `TaskParser`、`TaskDraft`、`ParseContext`。
4. 写 drift 数据表（list、task、tag、focus_session、habit、habit_checkin）。
5. 封装 RRULE（下一次生成、虚拟实例展开）。
6. 中文自然语言解析器及单元测试（含每月 31 号、跨夏令时等边界）。
7. 应用商店、商标（类别 9 和 42）、域名与包名核查。

## 阻塞项 / 待用户处理

- Flutter 未安装（WSL、Windows 均未检测到）。需在 Windows 本机安装 Flutter SDK、Visual Studio 2022（"使用 C++ 的桌面开发"）、Android Studio，开启开发者模式，`flutter doctor` 通过。
- 尚未决定是否在项目内启用夜间自动推进（Ralph 循环 / Routines）。建议等有了任务清单和单元测试之后再开，详见 `DESIGN.md` 之外的讨论结论：只处理已确认的需求，设迭代与费用上限，隔离运行，不自动合并主分支。

## 已知问题

- `DESIGN.md` 中 TickTick 的价格、免费版限制来自搜索摘要，未核对原文。
- 自动化方案相关信息来自博客和第三方说明，未核对官方文档。
- 云端无法编译 Flutter Windows 端；agent 夜间只适合推进纯 Dart 部分，界面需本机验证。
