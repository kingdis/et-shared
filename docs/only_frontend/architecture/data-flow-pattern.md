# 数据流架构设计模式

## 概述

本项目采用 **组件 → Hook → Store → API** 的四层架构模式，实现了清晰的职责分离和高效的数据管理。

## 架构图

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   组件层     │ -> │   Hook层    │ -> │   Store层   │ -> │   API层     │
│ Components  │    │   Hooks     │    │   Stores    │    │   Services  │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
      │                   │                   │                   │
      │                   │                   │                   │
   UI渲染              业务逻辑            数据管理             网络请求
   用户交互            状态封装            缓存策略             错误处理
```

## 各层职责

### 1. 组件层 (Components)
**职责：**
- UI 渲染和用户交互
- 调用 Hook 获取数据和方法
- 专注于展示逻辑

**示例：**
```typescript
const WeekNavigation: React.FC = () => {
    // 通过 Hook 获取数据和方法
    const { datesWithReminders } = useRemindersOverview();
    
    return (
        <div>
            {/* UI 渲染 */}
        </div>
    );
};
```

### 2. Hook层 (Hooks)
**职责：**
- 封装业务逻辑
- 监听状态变化，自动触发数据获取
- 提供统一的数据访问接口
- 处理组件级别的副作用

**示例：**
```typescript
export const useRemindersOverview = () => {
    const { dataGroup } = useSessionUser();
    const { fetchOverview, datesWithReminders } = useRemindersOverviewStore();

    // 自动监听 dataGroup 变化
    useEffect(() => {
        if (dataGroup) {
            fetchOverview(dataGroup);
        }
    }, [dataGroup, fetchOverview]);

    return {
        datesWithReminders,
        // 其他封装的方法...
    };
};
```

### 3. Store层 (Stores)
**职责：**
- 全局状态管理
- 数据缓存和去重
- 调用 API 服务
- 处理加载状态和错误状态

**示例：**
```typescript
export const useRemindersOverviewStore = create<RemindersOverviewState>((set, get) => ({
    datesWithReminders: new Set(),
    loading: false,
    error: null,
    
    fetchOverview: async (dataGroup: string) => {
        // 缓存检查
        if (isCacheValid()) return;
        
        // 请求去重
        if (hasPendingRequest()) return;
        
        // 调用 API
        const response = await apiGetRemindersOverview();
        set({ datesWithReminders: new Set(response.dates) });
    },
}));
```

### 4. API层 (Services)
**职责：**
- 网络请求封装
- 数据格式转换
- 错误处理
- 请求拦截和响应拦截

**示例：**
```typescript
export const apiGetRemindersOverview = async () => {
    const response = await apiClient.get('/user/task/reminders/overview');
    return response.data;
};
```

## 设计原则

### 1. 单一职责原则
- 每一层只负责自己的核心职责
- 避免跨层级的直接调用

### 2. 数据流向单一
- 数据只能从上层流向下层
- 避免循环依赖

### 3. 状态集中管理
- 全局状态统一在 Store 层管理
- 避免组件间的状态传递

### 4. 缓存和优化
- Store 层实现智能缓存
- 避免重复的网络请求

## 最佳实践

### 1. Hook 命名规范
```typescript
// 功能性 Hook
use[Feature]Overview()     // 获取概览数据
use[Feature]Controller()   // 控制器模式
use[Feature]Manager()      // 管理器模式
```

### 2. Store 命名规范
```typescript
// 数据存储
use[Feature]Store()        // 基础数据存储
use[Feature]OverviewStore() // 概览数据存储
```

### 3. API 命名规范
```typescript
// API 服务
apiGet[Resource]()         // 获取资源
apiCreate[Resource]()      // 创建资源
apiUpdate[Resource]()      // 更新资源
apiDelete[Resource]()      // 删除资源
```

## 实际案例：提醒概览功能

### 问题背景
多个组件需要显示"哪些日期有提醒"，导致重复的 API 请求。

### 解决方案
采用四层架构模式：

1. **组件层**：`WeekNavigation`, `ReminderListContent`
2. **Hook层**：`useRemindersOverview` - 封装业务逻辑，监听 dataGroup 变化
3. **Store层**：`useRemindersOverviewStore` - 全局状态管理，缓存和去重
4. **API层**：`apiGetRemindersOverview` - 网络请求封装

### 效果
- ✅ 消除重复请求
- ✅ 数据一致性
- ✅ 性能优化
- ✅ 代码复用

## 与传统模式对比

### 传统模式：组件 → Hook → API → Store
```typescript
// Hook 负责 API 调用和 Store 更新
const useReminders = () => {
    const { setTasks } = useTaskStore();
    
    const fetchReminders = async () => {
        const data = await apiGetReminders(); // Hook 调用 API
        setTasks(data); // Hook 更新 Store
    };
    
    return { fetchReminders };
};
```

**问题：**
- 重复请求难以避免
- 缓存策略分散
- 状态管理复杂

### 新模式：组件 → Hook → Store → API
```typescript
// Store 负责 API 调用和数据管理
const useRemindersStore = create((set) => ({
    fetchReminders: async () => {
        const data = await apiGetReminders(); // Store 调用 API
        set({ tasks: data }); // Store 更新自己
    }
}));

// Hook 只负责业务逻辑
const useReminders = () => {
    const { fetchReminders } = useRemindersStore();
    return { fetchReminders };
};
```

**优势：**
- 天然避免重复请求
- 集中的缓存策略
- 清晰的职责分离

## 数据同步策略

### 跨 Store 同步
当一个 Store 的数据变化影响另一个 Store 时：

```typescript
// 在 reminderStore 中
updateReminderMine: (reminder) => {
    set(/* 更新本地状态 */);
    // 同步更新相关的概览数据
    useRemindersOverviewStore.getState().refetch();
},
```

### 同步时机
- ✅ **数据修改时**：add, update, delete 操作
- ❌ **数据加载时**：set 操作（从服务器加载数据）

## 总结

这种四层架构模式实现了：
- **清晰的职责分离**
- **高效的数据管理**
- **优秀的性能表现**
- **良好的可维护性**

适用于中大型 React 应用的状态管理和数据流控制。 