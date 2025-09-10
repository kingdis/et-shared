# 星星、皇冠、金币和平均星星计算指南

## 概述

本文档详细说明了 ET UI 应用中星星、皇冠、金币和平均星星的计算逻辑，这些元素用于激励用户记录和评价他们的日常活动。

## 1. 星星计算 (Stars)

### 1.1 单个活动星星
- **来源**: 用户对每个活动的评分 (rating.overall)
- **范围**: 1-5 星
- **排除条件**: 睡觉活动不参与星星计算
- **计算公式**: 
  ```typescript
  if (activityCategory.name !== '睡觉' && rating.overall > 0) {
      totalStars += rating.overall;
  }
  ```

### 1.2 每日总星星
- **计算方式**: 当日所有非睡觉活动的星星总和
- **公式**: 
  ```typescript
  totalStars = sum(所有非睡觉活动的 rating.overall)
  ```

## 2. 皇冠计算 (Crowns)

### 2.1 皇冠获得条件
- **评分要求**: 活动评分必须达到 5 星
- **排除条件**: 睡觉活动不参与皇冠计算
- **计算公式**:
  ```typescript
  if (activityCategory.name !== '睡觉' && rating.overall >= 5) {
      totalCrowns++;
  }
  ```

### 2.2 皇冠统计
- **计算方式**: 统计当日获得5星评分的非睡觉活动数量
- **公式**:
  ```typescript
  totalCrowns = count(非睡觉活动 where rating.overall >= 5)
  ```

## 3. 金币计算 (Coins)

### 3.1 金币计算公式
金币计算公式：**(持续时间 ÷ 5) × 星星数量**

### 3.2 详细计算步骤

#### 3.2.1 前置条件
- **评分要求**: 活动必须有评分 (rating.overall > 0)
- **排除条件**: 睡觉活动不参与金币计算
- **基础公式**: `coins = timeUnits × rating`

#### 3.2.2 活动持续时间计算
根据 `timeProportion` 类型计算活动实际持续时间：

```typescript
// 1. 获取日志总持续时间（分钟）
const logDurationMinutes = activityLog.duration / 60; // 转换为分钟

// 2. 计算活动持续时间
let activityDurationMinutes;
if (activity.timeProportion?.type === 'duration') {
    // duration类型：值是秒数，转换为分钟
    activityDurationMinutes = activity.timeProportion.value / 60;
} else if (activity.timeProportion?.type === 'ratio') {
    // ratio类型：值是百分比，转换为小数比例
    const ratioDecimal = activity.timeProportion.value / 100;
    activityDurationMinutes = logDurationMinutes * ratioDecimal;
} else {
    // 无时间比例：平均分配给所有活动
    activityDurationMinutes = logDurationMinutes / activityLog.executedActivities.length;
}
```

#### 3.2.3 时间单位计算
```typescript
// 每5分钟为一个时间单位
const timeUnits = Math.floor(activityDurationMinutes / 5);
```

#### 3.2.4 最终金币计算
```typescript
const coins = timeUnits * rating;
```

### 3.3 完整计算函数
```typescript
export const calculateActivityCoins = (activity: IExecutedActivity, activityLog: IActivityLog): number => {
    // 如果没有评分，返回0
    const rating = activity.rating?.overall || 0;
    if (rating === 0) {
        return 0;
    }

    // 计算活动持续时间（分钟）
    const logDurationMinutes = activityLog.duration / 60;
    
    let activityDurationMinutes;
    if (activity.timeProportion?.type === 'duration') {
        activityDurationMinutes = activity.timeProportion.value / 60;
    } else if (activity.timeProportion?.type === 'ratio') {
        const ratioDecimal = activity.timeProportion.value / 100;
        activityDurationMinutes = logDurationMinutes * ratioDecimal;
    } else {
        activityDurationMinutes = logDurationMinutes / activityLog.executedActivities.length;
    }
    
    // 计算时间单位（每5分钟为一个单位）
    const timeUnits = Math.floor(activityDurationMinutes / 5);
    
    // 计算金币：时间单位 × 星星数量
    const coins = timeUnits * rating;
    
    return coins;
};
```

### 3.4 计算示例

#### 示例1：单一活动
- 活动时长：30分钟
- 评分：4星
- 计算：`Math.floor(30/5) × 4 = 6 × 4 = 24金币`

#### 示例2：比例分配活动
- 日志总时长：60分钟
- 活动比例：50%
- 评分：5星
- 计算：`Math.floor((60×0.5)/5) × 5 = Math.floor(30/5) × 5 = 6 × 5 = 30金币`

#### 示例3：无评分活动
- 活动时长：20分钟
- 评分：0星
- 计算：`0金币`（无评分不计算金币）

### 3.5 每日总金币
- **计算方式**: 当日所有非睡觉活动的金币总和
- **公式**:
  ```typescript
  totalCoins = sum(calculateActivityCoins(activity, activityLog) for 所有非睡觉活动)
  ```

## 4. 平均星星计算 (Average Stars)

### 4.1 计算逻辑
```typescript
const averageRating = activitiesWithRating > 0 
    ? Math.round((totalStars / activitiesWithRating) * 10) / 10 
    : 0;
```

### 4.2 计算步骤
1. **统计有评分的活动数量**:
   ```typescript
   activitiesWithRating = count(非睡觉活动 where rating.overall > 0)
   ```

2. **计算平均分**:
   ```typescript
   averageRating = totalStars / activitiesWithRating
   ```

3. **精度控制**: 保留一位小数

## 5. 鼓励文字生成

### 5.1 评分分级
根据平均评分生成不同的鼓励文字：

| 平均评分范围 | 鼓励文字类型 |
|-------------|-------------|
| ≥ 4.5 | 杰出表现 |
| ≥ 4.0 | 很棒 |
| ≥ 3.5 | 非常不错 |
| ≥ 3.0 | 做得不错 |
| ≥ 2.5 | 正在进步 |
| ≥ 2.0 | 有进步的空间 |
| ≥ 1.5 | 继续练习 |
| ≥ 1.0 | 需要更多练习 |
| < 1.0 | 继续努力 |

### 5.2 特殊情况
- **无评分数据**: 显示"开始记录活动吧"
- **子账户模式**: 仅在子账户模式下显示鼓励文字

## 6. 核心计算函数

### 6.1 主要计算逻辑
```typescript
const calculateDailyStats = (activityLogs: IActivityLog[], locale: string) => {
    let totalStars = 0;
    let totalCrowns = 0;
    let totalCoins = 0;
    let activitiesWithRating = 0;

    activityLogs.forEach(log => {
        log.executedActivities?.forEach(activity => {
            if (getLocaleName(activity.activityCategory.name, locale) !== '睡觉') {
                const rating = activity.rating?.overall || 0;
                if (rating > 0) {
                    totalStars += rating;
                    activitiesWithRating++;
                    if (rating >= 5) {
                        totalCrowns++;
                    }
                    totalCoins += calculateActivityCoins(activity, log);
                }
            }
        });
    });

    return {
        totalStars,
        totalCrowns,
        totalCoins,
        activitiesWithRating,
        averageRating: activitiesWithRating > 0 ? Math.round((totalStars / activitiesWithRating) * 10) / 10 : 0
    };
};
```

## 7. 数据模型

### 7.1 输入数据
- `IActivityLog[]`: 活动日志数组
- `locale`: 语言环境（用于识别睡觉活动）

### 7.2 输出数据
```typescript
interface DailyStats {
    totalStars: number;        // 总星星数
    totalCrowns: number;       // 总皇冠数
    totalCoins: number;        // 总金币数
    activitiesWithRating: number; // 有评分的活动数量
    averageRating: number;     // 平均评分
}
```

## 8. 注意事项

1. **睡觉活动排除**: 所有计算都排除睡觉活动
2. **评分有效性**: 只有评分大于0的活动才参与计算
3. **精度控制**: 平均评分保留一位小数
4. **性能考虑**: 使用 `useMemo` 优化计算性能
5. **数据完整性**: 确保活动数据包含必要的评分信息

## 9. 未来扩展

- 支持自定义评分权重
- 添加更多成就类型
- 实现连续打卡奖励
- 支持团队/家庭统计
- 添加历史趋势分析 