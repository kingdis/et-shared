# 睡眠指标 API 文档

## 概述

睡眠指标 API 提供了一个高性能的后端解决方案，用于计算各种睡眠健康指标。该API将原本在前端进行的复杂计算迁移到后端，显著提升了性能并减少了数据传输。

## API 端点

### GET /statistics/sleep-metrics

计算综合睡眠指标，包括一致性、平均值、健康评分和建议。

#### 请求参数

| 参数 | 类型 | 默认值 | 必填 | 描述 |
|------|------|--------|------|------|
| days | number | 180 | 否 | 要分析的天数 (1-365) |

#### 请求示例

```http
GET /statistics/sleep-metrics?days=180
Authorization: Bearer <your-token>
```

#### 响应格式

```json
{
  "sleepTimeConsistency": 85,
  "durationConsistency": 78,
  "avgSleepTime": {
    "hour": 23,
    "minute": 30
  },
  "avgWakeupTime": {
    "hour": 7,
    "minute": 15
  },
  "avgSleepDuration": 7.8,
  "avgSleepQuality": 82.3,
  "healthScore": 84.5,
  "timeRange": {
    "days": 157,
    "description": "157天数据",
    "startDate": "2024-01-01T00:00:00.000Z",
    "endDate": "2024-06-05T00:00:00.000Z",
    "totalDaysSpan": 180
  },
  "suggestions": [
    "maintainSleepHabitsSuggestion",
    "improveSleepConsistencySuggestion"
  ]
}
```

#### 字段说明

| 字段 | 类型 | 描述 |
|------|------|------|
| sleepTimeConsistency | number | 睡眠时间一致性百分比 (0-100) |
| durationConsistency | number | 睡眠时长一致性百分比 (0-100) |
| avgSleepTime | object\|null | 平均睡眠时间 {hour, minute} |
| avgWakeupTime | object\|null | 平均起床时间 {hour, minute} |
| avgSleepDuration | number | 平均睡眠时长（小时） |
| avgSleepQuality | number | 平均睡眠质量百分比 |
| healthScore | number | 综合健康评分 (0-100) |
| timeRange | object | 数据时间范围信息 |
| suggestions | string[] | 睡眠建议关键字数组 |

#### 健康评分计算

健康评分基于以下因素计算：
- **时间遵守度** (60%权重)：基于与睡眠计划目标时间的偏差
- **时长遵守度** (40%权重)：基于与目标睡眠时长的偏差，考虑效率因子

#### 一致性计算

- **睡眠时间一致性**：基于睡眠时间的标准差计算
- **睡眠时长一致性**：基于睡眠时长相对于平均值的变化计算

#### 建议系统

系统根据各项指标自动生成个性化睡眠建议：
- `maintainSleepHabitsSuggestion`：保持现有良好习惯
- `improveSleepConsistencySuggestion`：改善睡眠一致性
- `increaseSleepDurationSuggestion`：增加睡眠时长
- `reduceSleepDurationSuggestion`：减少睡眠时长
- `improveSleepQualitySuggestion`：改善睡眠质量

## 性能优势

相比原前端实现，此API提供了以下优势：

1. **减少数据传输**：只传输计算结果而不是原始数据
2. **服务器端计算**：利用服务器计算能力，减少客户端负载
3. **缓存优化**：后端可以实现更高效的数据缓存策略
4. **一致性保证**：统一的计算逻辑确保结果一致性

## 错误处理

API 使用标准HTTP状态码：

- `200`: 成功
- `400`: 请求参数错误
- `401`: 未授权
- `500`: 服务器内部错误

错误响应格式：
```json
{
  "error": "错误描述信息"
}
```

## 使用建议

1. 建议使用180天作为分析周期以获得最准确的健康评估
2. 对于数据不足30天的用户，结果仅供参考
3. 建议定期（如每周）调用此API更新用户的睡眠健康指标 