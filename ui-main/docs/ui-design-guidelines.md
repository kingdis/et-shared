# UI设计规范文档

## 概述

本文档记录了项目中UI组件的设计规范和最佳实践，确保整个应用的视觉一致性和用户体验的统一性。

## 按钮设计规范

### 圆形按钮设计

#### 基础圆形按钮模式（仅图标）

```tsx
<Button
    onClick={onActionHandler}
    variant="solid"
    shape="circle"
    color="primary"
    size="sm"
    className="ml-4"
    icon={iconComponent}
/>
```

#### 圆形按钮模式（图标+文字）

```tsx
<Button
    onClick={onActionHandler}
    shape="circle"
    variant="solid"
    color="primary"
    size="md"
    className="flex items-center gap-2"
    icon={iconComponent}
>
    {t('action.buttonText')}
</Button>
```

#### 设计原则

1. **形状与尺寸**
   - 使用 `shape="circle"` 创建圆形按钮
   - **仅图标按钮**：必须使用 `icon` 属性且无 `children`，推荐 `size="sm"` (40x40px)
   - **图标+文字按钮**：使用 `icon` 属性配合 `children`，推荐 `size="md"` (48x48px) 提供更好的视觉平衡

2. **视觉样式**
   - 主要操作使用 `variant="solid"` + `color="primary"`
   - 次要操作可使用 `variant="plain"` 或 `variant="default"`
   - 保持足够的点击区域 (最小44x44px)

3. **图标规范**
   - 统一使用 `@/configs/app-icon.config` 中定义的图标
   - 图标应具有语义化的含义，如 `plusIcon` 用于创建操作
   - 图标大小与按钮尺寸保持协调

#### 常用场景

**仅图标模式（移动端/紧凑布局）**
- **创建操作**: 使用 `plusIcon`，主色调实心样式
- **编辑操作**: 使用 `editIcon`，次要样式
- **删除操作**: 使用 `deleteIcon`，危险色或次要样式
- **更多操作**: 使用 `moreIcon`，平淡样式

**图标+文字模式（桌面端/主要操作）**
- **主要创建操作**: 结合 `plusIcon` 和文字说明，提供更好的用户体验
- **重要功能入口**: 图标增强识别度，文字提供明确说明
- **表单提交**: 如"保存"、"提交"等操作

#### 技术要点

⚠️ **重要提醒**: 
- 圆形按钮必须使用 `icon` 属性
- **仅图标模式**：不能有 `children`，按钮会应用固定宽高确保正方形
- **图标+文字模式**：可以同时使用 `icon` 和 `children`，需要添加 `className="flex items-center gap-2"` 确保良好的布局

### 尺寸规范

| 尺寸 | 类名 | 实际大小 | 使用场景 |
|------|------|----------|----------|
| xs   | h-8 w-8   | 32x32px | 紧凑型界面、表格内操作（仅图标） |
| sm   | h-10 w-10 | 40x40px | 移动端常规操作按钮（仅图标推荐） |
| md   | h-12 w-12 | 48x48px | 桌面端主要操作（图标+文字推荐） |
| lg   | h-14 w-14 | 56x56px | 重要操作、大屏显示 |

## 颜色规范

### 主题色彩

- **Primary**: 主要操作、重要元素
- **Success**: 成功状态、确认操作
- **Warning**: 警告状态、需要注意的操作
- **Danger**: 危险操作、删除等破坏性操作

## 间距规范

### 按钮间距

- 相邻按钮间距：`gap-2` (8px) 或 `gap-3` (12px)
- 按钮与其他元素间距：`ml-4` (16px) 或 `mr-4` (16px)
- 垂直间距：`mb-4` (16px) 或 `mt-4` (16px)

## 响应式设计

### 移动端适配

- 移动端按钮尺寸适当增大，最小点击区域44x44px
- 考虑触摸友好性，按钮间距适当增大
- 在窄屏幕上合理调整按钮布局

## 最佳实践

1. **一致性**: 相同功能的按钮在整个应用中保持一致的样式
2. **可访问性**: 确保按钮有足够的对比度和点击区域
3. **语义化**: 图标选择要符合用户认知和操作语义
4. **反馈**: 按钮应提供适当的交互反馈（hover、active状态）

## 代码示例

### 创建按钮组合

```tsx
// 桌面端创建按钮（图标+文字）
<Button
    onClick={onCreateNote}
    shape="circle"
    variant="solid"
    color="primary"
    size="md"
    className="flex items-center gap-2"
    icon={plusIcon}
>
    {t('feature.noteMgmt.createNote')}
</Button>

// 移动端工具栏中的创建按钮（仅图标）
<div className="flex items-center justify-between">
    <CategoryTabs
        selectedTab={selectedTab}
        onTabChange={onTabChange}
        // ... other props
    />
    <Button
        onClick={onCreateNote}
        variant="solid"
        shape="circle"
        color="primary"
        size="sm"
        className="ml-4"
        icon={plusIcon}
    />
</div>
```

### 操作按钮组

```tsx
// 卡片操作按钮组
<div className="flex gap-2">
    <Button
        shape="circle"
        size="sm"
        variant="plain"
        icon={editIcon}
        onClick={onEdit}
    />
    <Button
        shape="circle"
        size="sm"
        variant="plain"
        icon={moreIcon}
        onClick={onMore}
    />
</div>
```

---

*更新日期: 2024年12月*
*版本: v1.0* 