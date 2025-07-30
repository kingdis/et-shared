# Activity Logs Context-Aware Query Design

## 概述

活动日志的查询系统采用了一种复杂而巧妙的**上下文感知查询设计**，旨在为前端提供完整的时间轴上下文信息，支持准确的空白时间计算、跨天记录处理和用户友好的界面展示。

## 核心设计理念

### 问题背景

传统的简单日期范围查询（如 `startDate <= record.startTime < endDate`）无法满足以下需求：

1. **跨天活动的完整显示**：一个活动可能从昨天开始，今天结束
2. **空白时间的准确计算**：需要知道前一个活动的结束时间
3. **时间轴的连续性**：用户需要了解当前查询日期前后的活动上下文
4. **跨天记录的视觉处理**：需要特殊显示跨越午夜的活动

### 解决方案：上下文感知查询

我们的查询系统返回**三类记录**：

1. **前置上下文记录**：影响查询日期的历史活动
2. **主要记录**：查询日期当天开始的活动
3. **后置上下文记录**：提供未来活动预览的记录

## API 层实现

### Controller 层

```typescript
// activityLogController.ts
const options = {
    lean: true,
    showPreviousAndNext: true  // 🔑 关键选项：启用上下文查询
};
const activityLogs = await activityLogService.findActivityLogs(
    getAuthUser(req), 
    startDate, 
    endDate, 
    isSequential, 
    options
);
```

### Service 层核心逻辑

#### 1. 基础查询：重叠检测

```typescript
// 查找所有与查询时间范围有重叠的记录
query = {
    $and: [
        { startTime: { $lt: queryEndDate } },    // 开始时间在查询结束前
        { endTime: { $gte: queryStartDate } },   // 结束时间在查询开始后
        { isSequential: true }
    ],
};
```

**重叠检测的优势：**
- 捕获跨天活动：开始于查询日期之前但影响查询日期的活动
- 包含未来跨天：开始于查询日期但结束于未来的活动
- 自然处理边界情况

#### 2. 前置上下文查询

```typescript
// 如果第一个记录不是从查询日期开始，添加前置上下文
if (startDateOfFirstLog >= queryStartDate) {
    const previousLog = await super.findRecords(authUser, {
        isSequential: true,
        endTime: { $lte: firstLog.startTime }
    }, { limit: 1, sort: { endTime: -1 } });
}
```

**前置上下文的作用：**
- 提供空白时间计算的起点
- 支持跨天记录的完整显示
- 可能包含几天前开始的长期活动

#### 3. 后置上下文查询

```typescript
// 总是添加下一个活动，提供未来上下文
const nextLog = await super.findRecords(authUser, {
    isSequential: true,
    startTime: { $gte: lastLog.endTime }
}, { limit: 1, sort: { startTime: 1 } });
```

**后置上下文的智能性：**
- 自动适应最后活动是否跨天
- 为用户提供下一步计划的预览
- 支持连续时间轴的导航

## 前端处理逻辑

### 记录分类系统

前端根据记录与查询日期的关系，将API返回的记录分为三类：

```typescript
// 按显示角色分类，而非简单的日期分类
const recordsBeforeQueryDate: Array<{ record: IActivityLog; originalIndex: number }> = [];
const recordsOnQueryDate: Array<{ record: IActivityLog; originalIndex: number }> = [];
const recordsAfterQueryDate: Array<{ record: IActivityLog; originalIndex: number }> = [];

records.forEach((record, index) => {
    const recordStartDate = dayjs(record.startTime).startOf('day');
    const queryStartDate = dayjs(queryDate).startOf('day');
    
    if (recordStartDate.isBefore(queryStartDate)) {
        recordsBeforeQueryDate.push({ record, originalIndex: index });
    } else if (recordStartDate.isSame(queryStartDate)) {
        recordsOnQueryDate.push({ record, originalIndex: index });
    } else {
        recordsAfterQueryDate.push({ record, originalIndex: index });
    }
});
```

### 不同类型记录的处理策略

#### 1. 前置记录（recordsBeforeQueryDate）
- **跨天记录**：渲染为 `CrossDayActivityRecord` 组件
- **非跨天记录**：显示为上下文信息文本
- **目的**：提供时间轴的历史连续性

#### 2. 当天记录（recordsOnQueryDate）
- **完整渲染**：使用 `ActivityLogItem` 组件
- **空白时间计算**：基于完整记录序列计算间隙
- **目的**：主要的用户交互和编辑界面

#### 3. 后置记录（recordsAfterQueryDate）
- **上下文预览**：显示基本信息，不支持编辑
- **导航提示**：提供跳转到未来日期的入口
- **目的**：帮助用户了解未来安排

## 空白时间计算

### 核心算法

```typescript
const calculateTimeGap = (previousEndTime: Date | null, nextStartTime: Date | null): number => {
    if (!previousEndTime || !nextStartTime) return 0;
    
    // 处理跨天情况
    if (!isTheSameDay(previousEndTime, nextStartTime)) {
        return calculateDuration(
            new Date(nextStartTime.getFullYear(), nextStartTime.getMonth(), nextStartTime.getDate(), 0, 0, 0, 0), 
            nextStartTime
        ) || 0;
    }
    
    return calculateDuration(previousEndTime, nextStartTime) || 0;
};
```

### 跨天时间处理

```typescript
const getRightPreviousEndTime = (previousEndTime: Date | null, currentStartTime: Date) => {
    if (previousEndTime && isTheSameDay(previousEndTime, currentStartTime)) {
        return previousEndTime;
    }
    // 跨天情况：使用当天 00:00 作为起点
    return new Date(currentStartTime.getFullYear(), currentStartTime.getMonth(), currentStartTime.getDate(), 0, 0, 0, 0);
};
```

## 查询示例

### 示例：查询 2024年1月2日

**API返回可能包含：**

1. **前置上下文**：
   ```json
   {
     "_id": "prev1",
     "startTime": "2023-12-30T22:00:00Z",  // 3天前开始
     "endTime": "2024-01-02T02:00:00Z",    // 跨天到查询日期
     "isSequential": true
   }
   ```

2. **当天主记录**：
   ```json
   [
     {
       "_id": "main1", 
       "startTime": "2024-01-02T08:00:00Z",  // 当天开始
       "endTime": "2024-01-02T12:00:00Z"
     },
     {
       "_id": "main2",
       "startTime": "2024-01-02T14:00:00Z",  // 当天开始
       "endTime": "2024-01-04T10:00:00Z"     // 跨天到未来
     }
   ]
   ```

3. **后置上下文**：
   ```json
   {
     "_id": "next1",
     "startTime": "2024-01-04T15:00:00Z",   // 未来活动
     "endTime": "2024-01-04T18:00:00Z"
   }
   ```

### 前端处理结果

- **跨天记录显示**：`prev1` 显示为特殊的跨天组件
- **空白时间**：
  - 02:00-08:00 (6小时空白)
  - 12:00-14:00 (2小时空白)
- **上下文信息**：显示来自后天的下一个活动预览

## 设计优势

### 1. 用户体验优势
- **完整的时间轴视图**：用户可以看到活动的完整上下文
- **准确的空白时间**：支持精确的时间管理
- **直观的跨天显示**：清楚地表示跨越午夜的活动

### 2. 技术实现优势
- **单次查询获取完整上下文**：减少API调用次数
- **前端逻辑简化**：后端提供足够信息，前端专注展示
- **扩展性良好**：支持复杂的时间场景

### 3. 数据一致性优势
- **基于重叠检测**：避免遗漏边界情况
- **上下文感知**：确保时间轴的连续性
- **智能补全**：自动提供必要的上下文信息

## 注意事项

### 开发注意事项

1. **originalIndex 的重要性**：
   ```typescript
   // 保持在完整序列中的位置，确保空白时间计算准确
   const actualPreviousRecord = originalIndex > 0 ? records[originalIndex - 1] : null;
   ```

2. **条件性渲染**：
   ```typescript
   // 不同类型记录需要不同的渲染策略
   if (isCrossDayRecord) {
       return <CrossDayActivityRecord />;
   } else {
       return <ContextInfoDisplay />;
   }
   ```

3. **时区处理**：所有时间计算都需要考虑时区转换

### 性能考虑

- **查询优化**：使用索引优化重叠检测查询
- **数据量控制**：限制上下文记录数量（limit: 1）
- **缓存策略**：考虑对频繁查询的日期进行缓存

## 总结

这个上下文感知查询设计通过复杂但精心设计的后端逻辑，为前端提供了丰富的上下文信息，实现了：

- **准确的时间轴展示**
- **完整的跨天记录处理**
- **精确的空白时间计算**
- **良好的用户导航体验**

虽然实现复杂，但这种设计显著提升了时间管理应用的用户体验和功能完整性。 