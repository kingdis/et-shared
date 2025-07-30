# 睡眠数据模型移除总结

## 概述

本次重构完全移除了 `DailySleepDurationModel` 及其相关的预聚合睡眠数据系统，改为直接从 `ActivityLogModel` 查询睡眠数据。这一改动简化了系统架构，提高了数据一致性，并实现了更灵活的睡眠时间边界逻辑。

## 移除的文件

### 模型文件
- `src/core.models/db/DailySleepDurationModel.ts` - 睡眠数据模型定义

### 服务文件
- `src/core.services/DailySleepDurationService.ts` - 睡眠数据业务逻辑
- `src/core.services/DailySleepDurationAbstractService.ts` - 睡眠数据抽象服务

### 测试文件
- `tests/dailySleepDurationService.test.ts` - 相关测试

## 修改的文件

### 核心服务更新

#### `src/core.services/StatisticsService.ts`
- **移除依赖**：不再使用 `DailySleepDurationService`
- **新增方法**：`getPrimarySleepForDate()` - 直接从ActivityLog获取主睡眠数据
- **性能优化**：禁用不必要的population，提高查询效率
- **睡眠检查**：使用 `ActivityCategoryService.isSleepCategory()` 替代硬编码

#### `src/core.services/index.ts` (ServiceFactory)
- **移除服务**：删除 `getDailySleepDurationService()` 方法
- **清理导入**：移除相关导入语句

#### `src/core.models/index.ts`
- **移除导入**：删除 `DailySleepDurationModel` 的导入和导出

#### `src/core.services/DailyActivityAndDurationService.ts`
- **移除睡眠更新**：不再调用睡眠数据聚合方法
- **删除方法**：移除 `findSleepActivity()` 私有方法

### 控制器更新

#### `src/core.controllers/statisticsController.ts`
- **移除服务引用**：删除 `dailySleepDurationService` 实例
- **移除方法**：
  - `updateAllSleepDuration` - 不再需要预聚合睡眠数据
- **重新实现方法**：
  - `getSleepLastDayQuality` - 🆕 基于新的ActivityLog直接查询重新实现

#### `src/core.controllers/routes/statisticsRoutes.ts`
- **移除路由**：
  - `GET /statistics/sleep-duration/update-all`
- **重新实现路由**：
  - `GET /statistics/sleep-last-day-quality` - 🆕 使用新的ActivityLog查询逻辑
- **清理导入**：移除相关控制器方法的导入

## 架构改进

### 数据查询策略
- **之前**：预聚合睡眠数据到 `DailySleepDurationModel`，存在数据同步问题
- **现在**：直接从 `ActivityLogModel` 实时查询，确保数据一致性

### 睡眠边界逻辑
- **之前**：使用午夜12:00作为日期边界，与用户期望不符
- **现在**：使用6:00 AM作为睡眠日期边界，符合用户直觉

### 性能优化
- **Population控制**：禁用不必要的字段population
- **直接ID比较**：避免复杂的对象解构和类型检查
- **缓存机制**：`ActivityCategoryService` 使用缓存提高类别查询效率

### 多睡眠处理
- **策略**：选择时长最长的睡眠作为主睡眠
- **健康指导**：忽略碎片化睡眠，鼓励连续睡眠

## API变更

### 移除的端点
```
GET /statistics/sleep-duration/update-all
```

### 保留/重新实现的端点
```
GET /statistics/sleep-plan - 获取睡眠计划统计（增强功能）
GET /statistics/sleep-last-day-quality - 睡眠质量统计（🆕 基于新查询逻辑重新实现）
GET /statistics/daily-sleep-non-sleep-durations - 睡眠/非睡眠时长对比
```

## 数据库影响

### 集合移除
- `dailysleepduration` 集合将不再使用
- 现有数据可以安全删除（已有ActivityLog作为数据源）

### 索引优化
- ActivityLog集合的现有索引足以支持新的查询模式
- 无需额外的数据库迁移

## 测试更新

### 移除的测试
- `dailySleepDurationService.test.ts` - 相关服务测试

### 需要更新的测试
- 统计服务相关测试需要更新以反映新的数据查询逻辑
- 睡眠计划API测试需要验证新的6:00边界逻辑

## 向后兼容性

### API兼容性
- 核心睡眠统计API (`/statistics/sleep-plan`) 保持兼容
- 移除的API需要前端相应更新

### 数据兼容性
- 新实现完全基于现有ActivityLog数据
- 无需数据迁移，历史数据完全可用

## 性能影响

### 查询性能
- **提升**：减少了数据库JOIN操作
- **提升**：避免了不必要的字段population
- **提升**：使用缓存的类别查询

### 内存使用
- **降低**：减少了对象创建和复杂数据结构
- **降低**：简化了数据处理流程

## 维护性改进

### 代码简化
- 移除了复杂的数据同步逻辑
- 统一了睡眠数据的查询入口
- 减少了代码重复

### 错误处理
- 简化了错误场景（无需处理数据同步失败）
- 提高了数据一致性保证

## 后续工作

### 清理工作
1. 数据库中的 `dailysleepduration` 集合可以安全删除
2. 相关的数据库索引可以清理
3. 监控日志中的相关警告可以移除

### 功能增强
1. 考虑添加睡眠质量分析功能
2. 优化时区处理逻辑
3. 增加睡眠模式识别功能

## 总结

本次重构成功简化了睡眠数据处理架构，提高了系统的一致性和性能。通过直接查询ActivityLog，我们消除了数据同步问题，同时实现了更符合用户期望的睡眠时间边界逻辑。这为未来的睡眠分析功能奠定了坚实的基础。 