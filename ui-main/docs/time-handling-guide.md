# 时间处理指南 (Time Handling Guide)

## 概述

本文档记录了项目中正确的时间处理方式，避免常见的时区和日期格式错误。

## 核心原则

### ❌ 错误做法
- 使用 `new Date().toISOString().split('T')[0]` 生成日期字符串
- 手动拼接日期字符串：`date.getFullYear() + '-' + ...`
- 混用 UTC 时间和本地时间
- 在不同地方使用不同的日期格式化方法

### ✅ 正确做法
- 使用项目标准的 `getDateString(date)` 函数
- 统一使用本地时区处理
- 复用已有的时间工具函数
- 保持日期格式的一致性

## 项目标准工具函数

### 1. 日期字符串生成

```typescript
import { getDateString } from '@/utils/time';

// ✅ 正确：使用标准函数
const dateStr = getDateString(new Date()); // 输出: "2024-01-15"

// ❌ 错误：手动拼接
const dateStr = date.getFullYear() + '-' + 
               String(date.getMonth() + 1).padStart(2, '0') + '-' + 
               String(date.getDate()).padStart(2, '0');

// ❌ 错误：使用 UTC 时间
const dateStr = new Date().toISOString().split('T')[0];
```

### 2. 其他常用时间函数

```typescript
import { 
    getDateString,
    alignToMidnight,
    isTheSameDay,
    formatDate,
    formatTime 
} from '@/utils/time';

// 日期对齐到午夜
const midnightDate = alignToMidnight(dateString, { timezone: 'Europe/Berlin' });

// 检查是否同一天
const isSameDay = isTheSameDay(date1, date2);

// 格式化日期显示
const displayDate = formatDate(date, 'short', 'zh-cn');
```

## 时区问题案例分析

### 问题场景
在德国时区（UTC+1/UTC+2）下，当本地时间是 2025-06-23 时：
- `new Date().toISOString()` 返回 `"2025-06-22T22:00:00.000Z"`
- `.split('T')[0]` 得到 `"2025-06-22"`
- 导致日期不匹配，功能失效

### 解决方案
```typescript
// ❌ 问题代码
const today = new Date().toISOString().split('T')[0]; // UTC 时间

// ✅ 修复代码  
const today = getDateString(new Date()); // 本地时间
```

## API 日期参数规范

### 天气 API 示例
```typescript
// ✅ 正确的 API 调用
const fetchWeatherData = async (location: string, date?: string) => {
    const targetDate = date || getDateString(new Date());
    return await apiGetHourlyForecast({
        location,
        date: targetDate // YYYY-MM-DD 格式
    });
};
```

## 常见错误模式

### 1. 时区混用
```typescript
// ❌ 错误：混用 UTC 和本地时间
const utcDate = new Date().toISOString().split('T')[0];
const localDate = getDateString(new Date());
const isToday = utcDate === localDate; // 可能不匹配

// ✅ 正确：统一使用本地时间
const today = getDateString(new Date());
const isToday = targetDate === today;
```

### 2. 重复造轮子
```typescript
// ❌ 错误：重新实现已有功能
const formatDate = (date: Date) => {
    return date.getFullYear() + '-' + 
           String(date.getMonth() + 1).padStart(2, '0') + '-' + 
           String(date.getDate()).padStart(2, '0');
};

// ✅ 正确：使用项目标准函数
import { getDateString } from '@/utils/time';
const dateStr = getDateString(date);
```

### 3. 性能问题
```typescript
// ❌ 错误：在渲染中创建 Date 对象
const MyComponent = () => {
    const isToday = targetDate === getDateString(new Date()); // 每次渲染都创建
    return <div>{isToday ? '今天' : '其他'}</div>;
};

// ✅ 正确：使用 useMemo 缓存
const MyComponent = () => {
    const isToday = useMemo(() => 
        targetDate === getDateString(new Date()), 
        [targetDate]
    );
    return <div>{isToday ? '今天' : '其他'}</div>;
};
```

## 最佳实践检查清单

- [ ] 使用 `getDateString()` 生成日期字符串
- [ ] 避免手动拼接日期格式
- [ ] 统一使用本地时区处理
- [ ] 在组件中使用 `useMemo` 缓存日期计算
- [ ] 复用 `@/utils/time.ts` 中的工具函数
- [ ] API 参数使用 YYYY-MM-DD 格式
- [ ] 避免在渲染过程中创建新的 Date 对象

## 相关文件

- `src/utils/time.ts` - 时间工具函数库
- `src/app_ld_io/services/StatisticsService.ts` - 使用示例
- `src/app_ld_io/services/NoteIOService.ts` - 使用示例

## 修复历史

### 2025-01-XX: 天气功能时区问题修复
- **问题**: 使用 `toISOString().split('T')[0]` 导致时区不匹配
- **修复**: 统一使用 `getDateString()` 函数
- **影响文件**: 
  - `ActivityLogItemListContainer.tsx`
  - `useWeatherController.tsx`

---

**记住**: 时间处理是容易出错的地方，始终优先使用项目已有的标准化工具函数！ 