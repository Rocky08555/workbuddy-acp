# WorkBuddy ACP 派工技能

这是一个供 Codex 使用的用户级技能，帮助通过 WorkBuddy/CodeBuddy 的 ACP 会话执行编码或文件任务，并安全处理授权等待与中断状态。

## 适用场景

- 通过 `workbuddy_session_send` 创建或恢复 WorkBuddy/CodeBuddy ACP 会话。
- CodeBuddy 因文件或目录操作等待用户授权。
- WorkBuddy 界面显示“中断”，但 ACP 会话、工具调用或共享索引状态尚不明确。

## 主要规则

- 派发前确认会话工作目录，并将用户授权限定在任务和目录范围内。
- 对 `waiting_permission` 逐项核对工具、操作、路径或完整命令，只使用请求实际提供的最窄单次授权；不默认选择 `allow_always`，不绕过 ACP。
- 综合检查会话 `state`、`stopReason`、`error`、`cancelRequestedAt`、`pendingPermissions`、最后工具调用结果及后续 transcript 事件。取消时间、桌面标签或共享索引都不能单独证明任务已取消；共享索引也可能滞后。
- `waitMs` 超时可能只返回 `running`，应继续 poll 同一 ACP 会话。
- 工具调用没有结果、assistant 为 incomplete、又没有明确取消事件时，记录为 ACP/工具状态不明，不归因用户、不宣称改动结果，也不重复派发。
- 当前 MCP 无活动连接或不持有该 ACP 连接时，回到发起任务的 MCP 会话继续处理；不要另发同一任务。只在终态尝试 `close`，并留意连接所有权要求。
- `run_workbuddy_task` 是非交互调用，不套用 ACP 会话的 poll 和授权恢复流程。

## 安装

将 `SKILL.md` 放入 Codex 用户级技能目录：

`%USERPROFILE%\.codex\skills\workbuddy-acp\SKILL.md`

## 验证范围

此技能提供派工和状态判读指引，不会修改 WorkBuddy ACP 桥接运行时。真实 ACP 多请求授权往返尚未验证；相关中断日志的根因也仍未确定。
