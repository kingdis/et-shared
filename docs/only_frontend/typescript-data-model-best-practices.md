# TypeScript 数据模型最佳实践

## 概述

在前端应用中，我们经常需要为API请求和响应定义不同的数据结构。本文档介绍了在TypeScript中处理这种情况的最佳实践。

## 核心原则

### 1. 使用TypeScript工具类型而非重复定义

**✅ 推荐做法：**
```typescript
// 基础响应数据结构
export interface ISharedUser {
  user: string | IUserProfileBasicInfo;
  dataGroup: string;
  permissions: Array<ITaskPermissionType>;
}

// 使用 Omit 工具类型创建请求结构
export interface ISharedUserUpdateRequest extends Omit<ISharedUser, 'dataGroup'> {
  // dataGroup 字段由后端根据用户ID自动填充，因此从请求中排除
}
```

**❌ 不推荐做法：**
```typescript
// 重复定义相似结构
export interface ISharedUser {
  user: string | IUserProfileBasicInfo;
  dataGroup: string;
  permissions: Array<ITaskPermissionType>;
}

export interface ISharedUserRequest {
  user: string | IUserProfileBasicInfo;
  permissions: Array<ITaskPermissionType>;
}
```

### 2. 清晰的命名约定

| 用途 | 命名模式 | 示例 |
|------|----------|------|
| 响应数据 | `I{Entity}` | `IUser`, `ITask`, `ISharedUser` |
| 创建请求 | `I{Entity}CreateRequest` | `ITaskCreateRequest`, `IUserCreateRequest` |
| 更新请求 | `I{Entity}UpdateRequest` | `ITaskUpdateRequest`, `IUserUpdateRequest` |
| 查询参数 | `I{Entity}QueryParams` | `ITaskQueryParams`, `IUserSearchParams` |
| 摘要/简化版本 | `I{Entity}Summary` | `ITaskSummary`, `IUserSummary` |

### 3. 常用工具类型模式

#### 创建请求（排除系统生成字段）
```typescript
// 排除 ID 和时间戳字段
export interface ITaskCreateRequest extends Omit<ITask, '_id' | 'createdAt' | 'updatedAt'> {}
```

#### 更新请求（部分字段可选）
```typescript
// 除了 ID 外，其他字段都是可选的
export interface ITaskUpdateRequest extends Partial<Omit<ITask, '_id'>> {
  _id: string;
}
```

#### 选择特定字段
```typescript
// 只选择需要的字段
export interface ITaskSummary extends Pick<ITask, '_id' | 'title' | 'status'> {}
```

#### 复杂组合
```typescript
// 必需字段 + 可选字段的组合
export interface ITaskPartialUpdate extends 
  Pick<ITask, '_id'> & 
  Partial<Pick<ITask, 'title' | 'status' | 'priority'>> {}
```

### 4. 文档化设计决策

始终为特殊的数据结构添加注释，说明为什么要这样设计：

```typescript
// 用于更新任务分享的请求接口
// 使用 Omit 工具类型，清晰地表达与 ISharedUser 的关系
// dataGroup 字段由后端根据用户ID自动填充，因此从请求中排除
export interface ISharedUserUpdateRequest extends Omit<ISharedUser, 'dataGroup'> {}
```

## 实际应用示例

### 场景：任务分享功能

```typescript
// 1. 基础数据结构（从后端返回）
export interface ISharedUser {
  user: string | IUserProfileBasicInfo;
  dataGroup: string;  // 后端自动填充
  permissions: Array<ITaskPermissionType>;
}

// 2. 更新请求结构（发送到后端）
export interface ISharedUserUpdateRequest extends Omit<ISharedUser, 'dataGroup'> {
  // dataGroup 由后端根据用户ID自动确定
}

// 3. API 服务函数
export async function apiUpdateTaskSharedWith(
  taskId: string, 
  assignee: string, 
  sharedWith: ISharedUserUpdateRequest[]
) {
  return ApiService.fetchDataWithAxios<ITaskHydrated>({
    url: endpointConfig.taskSub.assigneeAndSharedWith,
    method: 'put',
    data: {
      _id: taskId,
      assignee,
      sharedWith,
    },
  });
}

// 4. 业务逻辑中的使用
const updateTaskAssigneeAndSharedWith = useCallback(async (
  taskId: string, 
  assigneeId: string, 
  sharedWithUserIds: string[]
) => {
  try {
    const updatedTask = await apiUpdateTaskSharedWith(
      taskId, 
      assigneeId,
      sharedWithUserIds.map(userId => ({
        user: userId,
        permissions: ['view', 'edit'] as ITaskPermissionType[]
      })) as ISharedUserUpdateRequest[]
    );
    
    // 更新本地状态...
  } catch (error) {
    // 错误处理...
  }
}, []);
```

## 优势

### 1. 类型安全
- 编译时捕获类型错误
- 自动完成和智能提示
- 重构时的类型检查

### 2. 可维护性
- 当基础类型改变时，相关类型自动更新
- 单一数据源原则
- 减少重复代码

### 3. 可读性
- 清晰表达数据结构之间的关系
- 明确的命名约定
- 自文档化的代码

### 4. 开发效率
- 减少手动同步类型定义的工作
- IDE 提供更好的支持
- 更容易进行代码审查

## 常见陷阱与解决方案

### 陷阱1：过度使用工具类型
```typescript
// ❌ 过度复杂
export interface IComplexRequest extends 
  Omit<Pick<Partial<IUser>, 'name' | 'email'>, never> & 
  Required<Pick<IUser, 'id'>> {}

// ✅ 简单明了
export interface IUserUpdateRequest {
  id: string;
  name?: string;
  email?: string;
}
```

### 陷阱2：缺少文档
```typescript
// ❌ 没有说明为什么要排除某些字段
export interface ITaskRequest extends Omit<ITask, 'createdAt' | 'updatedAt'> {}

// ✅ 清晰的文档说明
// 创建任务的请求接口
// 排除时间戳字段，因为这些字段由后端自动生成
export interface ITaskCreateRequest extends Omit<ITask, 'createdAt' | 'updatedAt'> {}
```

### 陷阱3：命名不一致
```typescript
// ❌ 命名不一致
export interface IUserReq {}
export interface TaskUpdateData {}
export interface SharedUserInput {}

// ✅ 一致的命名约定
export interface IUserCreateRequest {}
export interface ITaskUpdateRequest {}
export interface ISharedUserUpdateRequest {}
```

## 总结

使用TypeScript工具类型来定义API请求和响应数据结构是现代前端开发的最佳实践。它不仅提高了代码的类型安全性和可维护性，还使代码更加清晰和易于理解。

记住关键原则：
1. **使用工具类型而非重复定义**
2. **保持一致的命名约定**
3. **为设计决策添加文档**
4. **优先考虑可读性和可维护性**

通过遵循这些实践，你可以构建更加健壮和可维护的TypeScript应用程序。 