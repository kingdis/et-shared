# 全局消息系统架构文档

## 概述

为了解决 Store 层无法使用国际化的问题，我们实现了一个全局消息系统，实现了消息发送与显示的分离。

## 问题背景

### 原有问题
```typescript
// Store 中直接使用 Toast（有问题）
export const useReminderStore = create((set) => ({
  createReminder: async (taskData) => {
    try {
      // ...
    } catch (error) {
      // ❌ 问题：Store 无法使用国际化
      toast.push(createToastNotification(
        'danger',
        'Error', // 硬编码，无法国际化
        'Failed to create reminder'
      ));
    }
  }
}));
```

### 架构冲突
- **Store 层**：不应该依赖 UI 层的功能（Toast、国际化）
- **职责混乱**：数据管理层处理了 UI 显示逻辑

## 解决方案

### 架构设计
```
Store → MessageStore → Hook → Toast + i18n
  ↑         ↑          ↑        ↑
数据层    消息队列    UI控制    显示层
```

### 核心组件

#### 1. MessageStore（消息队列）
```typescript
// src/app_lc_store/ui/messageStore.ts
export interface SystemMessage {
  id: string;
  type: MessageType;
  titleKey: string;    // 国际化键
  messageKey: string;  // 国际化键
  params?: Record<string, any>; // 参数化支持
  timestamp: number;
  duration?: number;
}
```

**职责：**
- 管理消息队列
- 提供便捷的消息发送方法
- 自动生成消息ID和时间戳

#### 2. useSystemMessages Hook（消息处理）
```typescript
// src/app_lb_business_logic/hooks/useSystemMessages.tsx
export const useSystemMessages = () => {
  // 监听消息变化
  // 处理国际化翻译
  // 显示 Toast 通知
  // 管理消息生命周期
};
```

**职责：**
- 监听 MessageStore 变化
- 处理消息的国际化翻译
- 调用 Toast 显示消息
- 支持消息参数化

#### 3. 更新的 Store（消息发送）
```typescript
// Store 只发送消息键，不处理显示
export const useReminderStore = create((set) => ({
  createReminder: async (taskData) => {
    try {
      // ...
    } catch (error) {
      // ✅ 正确：只发送消息键
      useMessageStore.getState().showError(
        'common.error',
        'common.addFailedMessage'
      );
    }
  }
}));
```

## 实现细节

### 1. 消息类型映射
```typescript
const mapMessageTypeToToastType = (type: MessageType) => {
  switch (type) {
    case 'success': return 'success';
    case 'error': return 'danger';
    case 'warning': return 'warning';
    case 'info': return 'info';
  }
};
```

### 2. 参数化支持
```typescript
// 发送带参数的消息
useMessageStore.getState().showError(
  'common.error',
  'reminder.deleteFailedWithName',
  { name: reminderTitle }
);

// 翻译时替换参数
const translateWithParams = (key: string, params?: Record<string, any>) => {
  let translated = t(key);
  Object.entries(params || {}).forEach(([paramKey, value]) => {
    translated = translated.replace(`{{${paramKey}}}`, String(value));
  });
  return translated;
};
```

### 3. 集成到应用根组件
```typescript
// src/app_la_base/layout/Layouts.tsx
const Layout = ({ children }: CommonProps) => {
  // 在根组件使用系统消息处理
  useSystemMessages();
  
  return (
    // ...
  );
};
```

## 使用示例

### Store 中发送消息
```typescript
// 成功消息
useMessageStore.getState().showSuccess(
  'common.success',
  'reminder.createSuccess'
);

// 错误消息（带参数）
useMessageStore.getState().showError(
  'common.error',
  'reminder.deleteFailedWithName',
  { name: taskTitle }
);

// 警告消息（自定义持续时间）
useMessageStore.getState().addMessage({
  type: 'warning',
  titleKey: 'common.warning',
  messageKey: 'reminder.duplicateWarning',
  duration: 10000 // 10秒
});
```

### 国际化文件
```json
// locales/en.json
{
  "common": {
    "success": "Success",
    "error": "Error",
    "warning": "Warning",
    "addFailedMessage": "Failed to create item",
    "updateFailedMessage": "Failed to update item",
    "deleteFailedMessage": "Failed to delete item"
  },
  "reminder": {
    "createSuccess": "Reminder created successfully",
    "deleteFailedWithName": "Failed to delete reminder: {{name}}"
  }
}
```

## 优势总结

### 1. 架构清晰
- ✅ **职责分离**：Store 专注数据，Hook 处理 UI
- ✅ **单向数据流**：Store → MessageStore → Hook → UI

### 2. 国际化支持
- ✅ **完全支持**：消息键在 UI 层翻译
- ✅ **参数化**：支持动态内容插入

### 3. 灵活性
- ✅ **消息队列**：支持多条消息管理
- ✅ **自定义持续时间**：不同类型消息不同显示时长
- ✅ **类型安全**：TypeScript 完整支持

### 4. 可维护性
- ✅ **集中管理**：所有系统消息统一处理
- ✅ **易于测试**：Store 和 UI 逻辑分离
- ✅ **易于扩展**：可添加消息优先级、分组等功能

## 迁移指南

### 对于现有 Store
1. 移除直接的 Toast 调用
2. 使用 `useMessageStore.getState().showXxx()` 方法
3. 使用国际化键而非硬编码文本

### 对于新功能
1. 在 Store 中只发送消息键
2. 在国际化文件中添加对应的翻译
3. 使用参数化支持动态内容

### 集成步骤
1. 在应用根组件添加 `useSystemMessages()`
2. 更新所有 Store 中的错误处理
3. 添加对应的国际化文本

## 最佳实践

### 1. 消息键命名
```typescript
// 推荐的命名规范
'common.error'              // 通用错误标题
'common.addFailedMessage'   // 通用添加失败消息
'feature.action.result'     // 具体功能的具体结果
```

### 2. 错误处理模式
```typescript
// Store 中的标准错误处理
try {
  const result = await apiCall();
  return result;
} catch (error) {
  console.error('Operation failed:', error);
  useMessageStore.getState().showError(
    'common.error',
    'feature.operationFailed'
  );
  throw error; // 重新抛出以便上层处理
}
```

### 3. 成功消息
```typescript
// 只在用户主动操作时显示成功消息
const result = await createReminder(data);
useMessageStore.getState().showSuccess(
  'common.success',
  'reminder.createSuccess'
);
```

这个全局消息系统完美解决了 Store 层国际化问题，同时提供了更好的架构和用户体验。 