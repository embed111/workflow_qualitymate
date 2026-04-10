# V1-P4 prod 180347 质量巡检报告

## 质量结论
- 截至 2026-04-08 19:27:05 +08:00，live prod 仍运行 `20260408-180347`，更高 candidate 已变为 `20260408-191653`，但因为仍有 `3` 个 running 任务，`/api/runtime-upgrade/status` 仍返回 `can_upgrade=false`、`blocking_reason=存在运行中任务，暂不可升级`。
- 当前主线/保底接力没有断：`[持续迭代] workflow` 主线节点 `node-sti-20260408-bd6ab034` 仍在 running，保底计划 `sch-20260407-5ef5e5c8` 的 future 仍在 `2026-04-08T19:46:00+08:00`；helper ready 节点为 `0`，当前没有“该接力却没接上的”现象。
- 这轮不判定为“口径完全可信”。我确认现场至少还有 3 个需要继续追踪的质量缺口：`status-detail` 默认选中节点偏离主线、`failed` 总数与可见列表口径分叉、`status-detail.execution_chain.latest_run` 缺少 `provider_pid` 真相字段。
- 现场是 live 漂移态。我在 2026-04-08 19:22 至 19:27 之间看到 running 从 `5 -> 3` 收敛，`workflow_bugmate` 和 `workflow_testmate` 先后结束；以下数字统一以 `2026-04-08 19:27:05 +08:00` 快照为准。

## 证据
- 健康检查：`GET http://127.0.0.1:8090/healthz` 于 `2026-04-08T19:23:02+08:00` 返回 `ok=true`。
- dashboard/status：`GET http://127.0.0.1:8090/api/status` 于 `2026-04-08 19:27:05 +08:00` 显示 `running_task_count=3`、`failed_task_count=37`、`queued_task_count=0`；当前可见 failed agent 卡片合计只有 `9` 条。
- runtime-upgrade/status：`GET http://127.0.0.1:8090/api/runtime-upgrade/status` 显示 `current_version=20260408-180347`、`candidate_version=20260408-191653`、`candidate_is_newer=true`、`running_task_count=3`、`can_upgrade=false`。
- 图真相：`GET http://127.0.0.1:8090/api/assignments/asg-20260407-103450-fb8ba8/graph` 抽取结果为 `total_nodes=77`、`status_counts={running:3,succeeded:37,failed:37,ready:0}`。前 5 个节点依次是 `node-20260408-185759-b3dfba(workflow_qualitymate,running)`、`node-20260408-185632-41010f(workflow_devmate,running)`、`node-sti-20260408-bd6ab034(workflow,running)`、`node-20260408-185717-67fac0(workflow_testmate,succeeded)`、`node-20260408-185605-35144b(workflow_bugmate,succeeded)`。
- 计划接力：`GET http://127.0.0.1:8090/api/schedules/sch-20260407-20001ab4` 显示主线计划 `enabled=true`，最近命中仍是 `node-sti-20260408-bd6ab034` 且状态为 running；`GET http://127.0.0.1:8090/api/schedules/sch-20260407-5ef5e5c8` 显示保底计划 `enabled=true`，future trigger 仍是 `2026-04-08T19:46:00+08:00`。
- 节点文件真相：`C:/work/J-Agents/.output/tasks/asg-20260407-103450-fb8ba8/nodes/*.json` 统计结果为 `active_failed_nodes=37`、`active_helper_ready_nodes=0`。
- 默认详情选中规则：`GET http://127.0.0.1:8090/static/workflow-web-client.js` 中 `pickAssignmentDefaultNode(graphData)` 的规则是“先第一条 running，再第一条 failed，再第一条节点”。结合当前 graph 前 3 条 running 顺序，默认详情会落到 `node-20260408-185759-b3dfba`，不会落到主线 `node-sti-20260408-bd6ab034`。
- detail 真相缺口：`GET http://127.0.0.1:8090/api/assignments/asg-20260407-103450-fb8ba8/status-detail?node_id=node-20260408-185759-b3dfba` 中 `execution_chain.latest_run.run_id=arun-20260408-191039-8c1564`，且 `has_provider_pid=false`；对应真相文件 `C:/work/J-Agents/.output/tasks/asg-20260407-103450-fb8ba8/runs/arun-20260408-191039-8c1564/run.json` 明确记录 `provider_pid=7400`。

## 风险项
- P1：`status-detail.execution_chain.latest_run` 缺少 `provider_pid`。影响是页面 detail 看到的运行句柄信息不完整，出现 provider 进程排查、stale running 定位和真相核对时，用户需要额外翻 `run.json` 才能闭环。
- P1：`failed` 总数与当前可见列表口径分叉。当前 summary/file truth 都指向 `37` 条 active failed，但 dashboard 当前可见 failed 卡片只有 `9` 条，PM 或值班同学容易把“最近窗口”误当成“全量失败列表”。
- P2：`status-detail` 默认选中节点偏向第一条 running helper，而不是主线 workflow。当前现场默认会先打开 `workflow_qualitymate` 的 running 节点详情，容易让人误判“主线现在不在跑”或忽略主线实际状态。

## 建议路由
- 待建缺陷 1：`P1 prod status-detail latest_run provider_pid 缺失`，建议路由给 `workflow_bugmate`。
- 待建缺陷 2：`P1 prod dashboard/status failed 总数与可见列表口径分叉`，建议路由给 `workflow_bugmate`，必要时联动前端 owner 一起明确“全量/窗口”展示语义。
- 待建缺陷 3：`P2 prod status-detail 默认详情未优先落主线 workflow`，建议路由给 `workflow_bugmate` 或前端 owner。

## 回归建议
- 回归 `provider_pid`：修复后至少同时抽查 1 个 running 节点和 1 个刚结束节点，核对 `status-detail.execution_chain.latest_run.provider_pid` 与对应 `runs/*/run.json` 完全一致。
- 回归 `failed` 口径：同一时刻对比 `dashboard summary`、页面可见 failed 列表、`graph.status_counts.failed`、`nodes/*.json active_failed_nodes`，要么数值一致，要么页面明确标出“仅展示最近 N 条”。
- 回归默认详情：构造“helper 与 workflow 主线同时 running”的现场，确认默认详情是否优先主线节点，或至少保留上次用户手选节点而不是总回退到第一条 helper。
- 回归接力链：在下一次保底巡检 `2026-04-08T19:46:00+08:00` 后复查主线 running、保底 future、helper ready、runtime-upgrade 门禁，确认这轮 live 漂移收敛后口径没有继续分叉。
