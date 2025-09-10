# 活动日志 - 当天区间 + 临近记录 查询说明

> 目标：给定一个时间区间 `[startDate, endDate)`（UTC），查询与该区间相交的“当天记录”（main），并且额外提供各一条“前一个邻接记录”（prev）和“后一个邻接记录”（next），用于时间轴等视图的连贯展示。

## 接口与位置
- 控制器方法：`getActivityLogsWithAdjacent`
  - 路由：`GET /api/user/log/activity-logs/with-adjacent`
  - Query 参数：
    - `startDate`：ISO 字符串（UTC），必填
    - `endDate`：ISO 字符串（UTC），必填
    - `isSequential`：可选（`true`/`false`），筛选顺序型或非顺序型日志
- 应用服务：`ActivityLogApplicationService.findRangeWithAdjacent(authUser, startDate, endDate, isSequential?, options?)`
  - 代码位置：`src/core.services/application/ActivityLogApplicationService.ts`

## 区间与邻接定义（严格使用请求边界）
- 当天（main）：与 `[startDate, endDate)` 有交集
  - 条件：`startTime < endDate AND endTime >= startDate`
- 前一个（prev）：严格在区间之前结束
  - 条件：`endTime < startDate`
  - 取法：按 `endTime` 倒序排序，取 1 条（“最新的上一条”）
- 后一个（next）：严格在区间之后（或跨过区间上界）结束
  - 条件：`endTime >= endDate`
  - 取法：按 `endTime` 正序排序，取 1 条（“最早的下一条”）

注意：不再使用“当天 00:00 / 次日 00:00”的日界线做邻接判断，完全以请求传入的 `startDate`/`endDate` 为边界，避免在前端传入非整点日界（如 22:00→次日 22:00）时出现错配。

## 过滤与去重
- `isSequential`：
  - `true`：仅返回顺序型记录（main/prev/next 均受此条件约束）
  - `false`：仅返回非顺序型记录
  - 未传：两类均可
- 去重：若 next/prev 候选已包含在 main 结果中（例如跨天记录本身落入 main），不会重复追加。
- 返回顺序：`[prev?, ...main, next?]`

## 时区与时间基准
- 统一以 UTC 处理并比较时间。
- 前端与服务端应传递/解析 UTC ISO 时间；不要在服务端将请求时间转换为“本地日界”。

## 常见错误与避免方式
- 错误使用“当天 00:00 / 次日 00:00”作为邻接边界：应当以请求的 `startDate`/`endDate` 为唯一边界。
- 将请求时间转换为“本地日界”再计算：应全链路使用 UTC。
- 未按 `endTime` 排序选邻接：prev 需 `endTime` 倒序取 1 条，next 需 `endTime` 正序取 1 条。
- 未对候选去重：若候选已在 main 中（跨天记录），不要重复追加。
- 在前端二次过滤 `isSequential`：应直接通过参数交给后端过滤，避免遗漏或错序。

## 示例
- 请求：
  - `GET /api/user/log/activity-logs/with-adjacent?startDate=2025-08-18T22:00:00.000Z&endDate=2025-08-19T22:00:00.000Z`
- 期望：
  - prev：`endTime < 2025-08-18T22:00:00Z` 的最新一条
  - main：与 `[2025-08-18T22:00Z, 2025-08-19T22:00Z)` 相交的所有日志（含跨天）
  - next：`endTime >= 2025-08-19T22:00:00Z` 的最早一条

## 调试脚本（可选）
- 位置：`scripts/debug-findRangeWithAdjacent.ts`
- 用法：
  - 指定 dataGroup 与单日：
    - `DATA_GROUP=xxx ts-node scripts/debug-findRangeWithAdjacent.ts 2025-08-19`
  - 指定 dataGroup 与自定义起止：
    - `DATA_GROUP=xxx ts-node scripts/debug-findRangeWithAdjacent.ts 2025-08-18T22:00:00.000Z 2025-08-19T22:00:00.000Z`
  - 可选筛选：`IS_SEQUENTIAL=true|false`
- 输出：为每条记录标注 `PREV`/`MAIN`/`NEXT` 分类，便于与页面结果对齐排查。

## 测试
- 单元/集成测试覆盖：`tests/findRangeWithAdjacent.test.ts`
  - 使用真实 Mongo 测试环境，测试前清空数据并写入用例，验证顺序、去重、过滤逻辑。

## 对接建议
- 前端若需要“当天”概念，请在前端以 UTC 计算日界，并将期望的 `[startDate, endDate)` 传给后端。
- 若需要仅顺序型或仅非顺序型，传 `isSequential` 明确筛选，避免前端二次过滤导致遗漏或错序。
