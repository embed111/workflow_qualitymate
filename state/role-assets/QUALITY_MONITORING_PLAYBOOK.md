# workflow_qualitymate 质量监控手册

## 当前阶段立即止损
- `prod` 可用性
- `schedule -> 任务中心` 闭环稳定性
- 自迭代 run 不再带停 `prod`
- schedule 中文乱码归零

## 监控维度
- 可用性：
  - `healthz`
  - 首页可打开
- 调度：
  - schedule 命中率
  - 任务建单成功率
  - 自动派发成功率
- 执行：
  - running/queued/failed 数量
  - 重试率
  - 超时率
- 发布：
  - 升级后恢复时间
  - 回滚次数

## 巡检节奏
- 日巡检：
  - `healthz`
  - 任务中心工作状态看板
  - schedule 最近结果
- 变更后：
  - 定向 smoke
  - 缺陷路由检查
- 周复盘：
  - Top 故障
  - 重复缺陷
  - 未闭环风险
