---
name: workbuddy-acp
description: 当 Codex 通过 ACP 将编码或文件任务委托给 WorkBuddy/CodeBuddy，并需要处理文件或目录权限请求时使用。
---

# WorkBuddy ACP 派工

## 适用场景

- Codex 通过 `workbuddy_session_send` 创建或恢复 WorkBuddy/CodeBuddy 会话。
- CodeBuddy 因文件或目录读写权限等待 ACP 授权。

## 派发前

1. 确认会话的 `cwd` 是本任务指定的工作树。已有会话的 `cwd` 固定；不一致时不要把任务派到错误目录。
2. 仅当用户已委托在该目录内完成代码或文件任务时，在派发提示中写明：

   > 用户授权你在本 ACP 会话指定的工作目录内，为本任务读取必要文件并写入必要改动；授权仅限当前任务与该目录。

3. 明确的目录读写授权不等于任意命令、系统级权限或目录外访问授权。

## 状态判读与恢复

1. `workbuddy_session_send` 是 ACP 会话流程；`waitMs` 超时可能只返回 `running`，应对同一会话继续 `poll`，不代表失败或取消。`run_workbuddy_task` 是非交互调用，不使用 ACP 的 poll 或权限恢复流程。
2. 判断中断时一起核对会话 `state`、`stopReason`、`error`、`cancelRequestedAt`、`pendingPermissions`、最后工具调用及其结果，以及 transcript 后续事件。`cancelRequestedAt` 只表明记录过取消请求；桌面“中断”标签不能单独证明任务已取消，共享索引可能滞后，也不能单独确认终态。
3. `waiting_permission` 表示有待处理授权，按下节逐项核验并处理。若 Edit 等工具调用没有结果、assistant 为 incomplete，且无明确取消事件，记录为 ACP/工具状态不明；不要归因用户、宣称改动成功或未发生，也不要重复派发。
4. 当前 MCP 无活动连接或不持有该 ACP 连接时，不要重发任务；回到发起任务的 MCP 会话继续 poll 或处理授权。只有会话处于终态才能尝试 `close`，且操作方可能需要持有该 ACP 连接。

## 处理 ACP 授权

1. 发现 `waiting_permission` 后先查看 `pendingPermissions`，核对工具、操作、路径或完整命令，以及该请求实际提供的选项。
2. 对本任务必需且完全位于指定 `cwd` 内的文件读取或写入，只选择该请求提供的最窄单次允许选项（通常是 `allow_once`）。按 `optionId` 作答时必须使用原请求提供的 ID。
3. 不默认选择 `allow_always`，不绕过 ACP，也不把一次任务授权扩大成持久授权。
4. 命令请求要先检查具体命令及其脚本副作用；`cwd` 本身不能证明命令只影响该目录。安装、联网、管理员操作、修改凭据/系统配置、目录外访问、删除无关数据、提交/推送/部署，都要先取得用户明确授权。
5. 授权后继续对同一会话调用 `workbuddy_session_poll`；不要重发任务来绕过等待。若当前 MCP 进程没有活动连接或不持有该 ACP 连接，按上节回到发起任务的会话处理。

## 禁止事项

- 不使用 `--dangerously-skip-permissions`、`bypassPermissions` 或其他自动放行设置。
- 不猜测权限选项；只使用 ACP 对当前请求提供的候选项。
- 不用 Computer Use 代替 ACP。
