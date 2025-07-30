# Activity Log User Interaction Access Control

## 概述

本文档详细描述了活动日志模块中基于用户账户类型的交互访问控制逻辑。考虑到应用既需要支持用户个人自身的管理，又需要实现用户对自己孩子的管理，因此在页面交互功能的渲染上需要根据 `isInChildAccount` 来判断用户是否在自己的管理页面，还是在接受对孩子的管理。

## 核心概念

### isInChildAccount 判断逻辑

`isInChildAccount` 是一个布尔值，用于判断当前用户是否在子账户模式下：

```typescript
export const checkIsInChildAccount = (
    children?: IUserProfileBasicInfo[],
    currentDataGroup?: string
): boolean => {
    // 如果没有 children 数组，说明不是父账户，无法判断
    if (!children || children.length === 0 || !currentDataGroup) {
        return false;
    }
    
    // 检查当前的 dataGroup 是否属于某个孩子
    return children.some(child => child.dataGroup === currentDataGroup);
}
```

### 两种账户模式

1. **家长个人账户模式** (`isInChildAccount = false`)
   - 用户在管理自己的活动日志
   - 当前 `dataGroup` 是家长自己的数据组
   
2. **家长控制子账户模式** (`isInChildAccount = true`)
   - 家长在管理孩子的活动日志
   - 当前 `dataGroup` 是某个孩子的数据组

## 功能访问控制规则

### 家长个人账户模式 (isInChildAccount = false)

在这种模式下，用户可以：

✅ **可以使用的功能**：
- 项目标签显示和编辑
- 受益人标签显示和编辑
- 添加和管理项目
- 添加和管理受益人
- 活动标签管理
- 子活动管理
- 笔记编辑
- 时长配置

❌ **不能使用的功能**：
- 星级评分功能

### 家长控制子账户模式 (isInChildAccount = true)

在这种模式下，用户可以：

✅ **可以使用的功能**：
- 星级评分功能
- 活动标签管理
- 子活动管理
- 笔记编辑
- 时长配置

❌ **不能使用的功能**：
- 项目标签显示和编辑
- 受益人标签显示和编辑
- 添加和管理项目
- 添加和管理受益人

## 实现细节

### 桌面端组件 (ExecutedActivityRecordDesktop.tsx)

#### 项目功能控制
```typescript
// 项目标签显示 - 仅在家长个人账户下显示
{activity.executedProject && !isInChildAccount && (
    <div key={activity.executedProject?._id} className={`my-[6px] transition-all duration-200`}>
        <ModernTag color="sky" variant="subtle" size="sm">
            {activity.executedProject?.project.name}
        </ModernTag>
    </div>
)}

// 添加项目按钮 - 仅在家长个人账户下显示
{!activity.executedProject && activity.activityCategory.meaningId !== 'sleep' && !isInChildAccount && (
    <div className='my-[6px]'>
        <Button icon={<PiFolderOpen />} />
    </div>
)}
```

#### 受益人功能控制
```typescript
// 受益人标签显示 - 仅在家长个人账户下显示
{!isInChildAccount && activity.beneficiaries?.map(beneficiary => (
    <div key={beneficiary._id} className={`my-[6px] transition-all duration-200`}>
        <ModernTag color="fuchsia" variant="subtle" size="sm">
            {beneficiary.recipient.alias}
        </ModernTag>
    </div>
))}

// 添加受益人按钮 - 仅在家长个人账户下显示
{activity.beneficiaries?.length === 0 && activity.activityCategory.meaningId !== 'sleep' && !isInChildAccount && (
    <div className='my-[6px]'>
        <Button icon={<PiUsers />} />
    </div>
)}
```

#### 评分功能控制
```typescript
// 星级评分 - 仅在子账户模式下显示
{activity.activityCategory.meaningId !== 'sleep' && isInChildAccount && (
    <div className={`my-[6px] transition-all duration-200`}>
        <StarRating
            value={activity.rating?.overall || 0}
            size="sm"
            readonly={false}
            showZero={true}
            onChange={(value) => {
                handleUpdateRating(record._id, activity._id, value);
            }}
        />
    </div>
)}
```

#### 配置面板控制
```typescript
// 项目配置面板 - 仅在家长个人账户下显示
{activeEditor?.id === activity._id && activeEditor.type === 'activity_project_configuration' && !isInChildAccount && (
    <div className="mt-2 mb-2">
        <ExecutedActivityRecordElementConfigPanel elementType='executedProject' />
    </div>
)}

// 受益人配置面板 - 仅在家长个人账户下显示
{activeEditor?.id === activity._id && activeEditor.type === 'activity_beneficiaries_configuration' && !isInChildAccount && (
    <div className="mt-2 mb-2">
        <ExecutedActivityRecordElementConfigPanel elementType='beneficiaries' />
    </div>
)}
```

### 移动端组件 (ExecutedActivityRecordMobile.tsx)

移动端组件遵循相同的逻辑：

```typescript
// 项目标签 - 仅在家长个人账户下显示
{activity.executedProject && !isInChildAccount && (
    <div className='flex-shrink-0'>
        <ModernTag color="sky" variant="subtle" size="sm">
            {activity.executedProject.project.name}
        </ModernTag>
    </div>
)}

// 受益人标签 - 仅在家长个人账户下显示
{!isInChildAccount && activity.beneficiaries?.map((beneficiary, idx) => (
    <div key={idx} className='flex-shrink-0'>
        <ModernTag color="fuchsia" variant="subtle" size="sm">
            {beneficiary.recipient.alias}
        </ModernTag>
    </div>
))}

// 星级评分 - 仅在子账户模式下显示
{activity.activityCategory.meaningId !== 'sleep' && isInChildAccount && (
    <div className='flex-shrink-0'>
        <StarRating value={activity.rating?.overall || 0} readonly={true} />
    </div>
)}
```

### 跨天活动记录组件 (CrossDayActivityExecutedRecord.tsx)

跨天活动记录组件也遵循相同的访问控制逻辑：

```typescript
// 项目标签 - 仅在家长个人账户下显示
{activity.executedProject && !isInChildAccount && (
    <div className="mr-2">
        <ModernTag color="sky" variant="subtle" size="sm">
            {activity.executedProject.project.name}
        </ModernTag>
    </div>
)}

// 受益人标签 - 仅在家长个人账户下显示
{!isInChildAccount && activity.beneficiaries?.map((beneficiary, idx) => (
    <div key={idx} className="mr-2">
        <ModernTag color="fuchsia" variant="subtle" size="sm">
            {beneficiary.recipient.alias}
        </ModernTag>
    </div>
))}

// 星级评分 - 仅在子账户模式下显示
{activity.activityCategory.meaningId !== 'sleep' && isInChildAccount && (
    <div className='flex-shrink-0'>
        <StarRating value={activity.rating?.overall || 0} readonly={true} />
    </div>
)}
```

## 设计理念

### 功能分离原则

1. **个人管理功能**：项目和受益人管理属于个人规划和资源管理功能，适合在家长个人账户模式下使用
2. **监督评价功能**：星级评分属于对活动质量的评价和监督功能，适合在家长控制子账户模式下使用

### 用户体验考虑

1. **角色清晰**：通过功能的显示和隐藏，让用户清楚地知道当前处于哪种管理模式
2. **功能聚焦**：不同模式下显示不同的功能，避免界面混乱和功能冲突
3. **权限明确**：每种模式下的权限边界清晰，避免误操作

## 相关文件

### 主要组件文件
- `src/app_lb_business_logic/core/feat_activity_logs/log_list_item_executed_activity_item/ExecutedActivityRecordDesktop.tsx`
- `src/app_lb_business_logic/core/feat_activity_logs/log_list_item_executed_activity_item/ExecutedActivityRecordMobile.tsx`
- `src/app_lb_business_logic/core/feat_activity_logs/log_list_item_executed_activity_item/CrossDayActivityExecutedRecord.tsx`

### 工具函数
- `src/utils/childAccountUtils.ts` - 包含 `checkIsInChildAccount` 函数

### 状态管理
- `src/app_lc_store/ui/authStore.ts` - 用户认证状态管理
- `src/app_lb_business_logic/models/auth.model.ts` - 用户模型定义

## 注意事项

1. **一致性**：所有相关组件都应该遵循相同的访问控制逻辑
2. **测试**：需要在两种模式下都进行充分的测试
3. **文档更新**：当访问控制逻辑发生变化时，需要及时更新本文档

## 更新历史

- **2024-01-XX**: 初始版本，定义了基于 `isInChildAccount` 的访问控制逻辑
- **2024-01-XX**: 完善了移动端和跨天活动记录组件的访问控制实现 