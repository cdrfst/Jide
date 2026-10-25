## Why

Jide 第一阶段（MVP）需要把"任务管理 + 到期提醒"的核心行为定义清楚，作为后续实现、测试与需求变更的基线。范围来自 docs/DECISIONS.md 的 D-006：任务与清单、子任务、标签、优先级、截止时间、重复任务、本地提醒、智能清单。

## What Changes

- 新增 task 规格：清单、任务、子任务、标签、优先级、截止时间、软删除
- 新增 repeat 规格：基于 RRULE 的重复任务，区分按截止日与按完成日
- 新增 reminder 规格：本地提醒与滚动窗口调度
- 新增 smart-list 规格：今天、明天、最近 7 天、收集箱

不在本次范围：日历、看板、四象限、番茄钟、习惯打卡、同步、协作、AI 解析。

## Impact

- 影响代码：lib/data（drift 表与仓库）、lib/core（RRULE 封装）、lib/platform/notify（提醒）、lib/features/task 与 smart_list
- 状态：规格草案，待用户确认后再进入实现
