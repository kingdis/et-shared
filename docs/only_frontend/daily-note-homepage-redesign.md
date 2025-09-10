# DailyNoteHomePage 重新设计文档

## 概述

本文档描述了 `DailyNoteHomePage.tsx` 的重新设计方案，旨在改善用户体验和界面布局。新设计将采用左侧导航面板的布局，参考 `TaskManagementHomePage.tsx` 的设计模式，提供更好的笔记查询和导航功能。

## 当前状态分析

### 现有问题

1. **布局不够优化**: 当前设计缺乏清晰的导航结构
2. **查询方式有限**: 主要基于日期查询，缺乏其他查询维度
3. **导航位置不合理**: NoteNavigation 组件位于右侧，不够突出
4. **移动端体验待优化**: 移动端布局需要更好的适配

### 现有功能保留

- ✅ **搜索功能**: 保持现有的 SearchCard 和 SearchResults 功能不变
- ✅ **分类筛选**: 保持 work/study/life 分类功能
- ✅ **笔记操作**: 保持编辑、删除、标记等操作功能

## 新设计方案

### 整体布局结构

参考 `TaskManagementHomePage.tsx` 的设计，采用左右分栏布局：

```
┌─────────────────────────────────────────────────────────────┐
│                        Header Area                          │
├─────────────────┬───────────────────────────────────────────┤
│   Left Panel    │            Main Content Area             │
│   (w-80)        │              (flex-1)                   │
│                 │                                         │
│ ┌─────────────┐ │ ┌─────────────────────────────────────┐ │
│ │Query Methods│ │ │          Search Card                │ │
│ │   Card      │ │ └─────────────────────────────────────┘ │
│ └─────────────┘ │                                         │
│                 │ ┌─────────────────────────────────────┐ │
│ ┌─────────────┐ │ │                                     │ │
│ │Note         │ │ │         Note List Content           │ │
│ │Navigation   │ │ │      (Search Results or            │ │
│ │   Card      │ │ │       Daily Notes Content)         │ │
│ └─────────────┘ │ │                                     │ │
│                 │ └─────────────────────────────────────┘ │
└─────────────────┴───────────────────────────────────────────┘
```

### 左侧面板设计

#### 第一个卡片: 查询方式选择器 (Query Methods Card)

```typescript
interface QueryMethod {
  key: 'recent' | 'flagged' | 'by-date';
  label: string;
  icon: React.ComponentType;
  description: string;
  color: string;
  bgColor: string;
  hoverBg: string;
  activeBg: string;
  borderColor: string;
}

const queryMethods: QueryMethod[] = [
  {
    key: 'recent',
    label: t('feature.noteMgmt.recentEdited'),
    icon: HiOutlineClock,
    description: t('feature.noteMgmt.recentEditedDesc'),
    color: 'text-blue-500',
    bgColor: 'bg-blue-50 dark:bg-blue-900/20',
    hoverBg: 'hover:bg-blue-50 dark:hover:bg-blue-900/20',
    activeBg: 'bg-blue-100 dark:bg-blue-900/30',
    borderColor: 'border-blue-200 dark:border-blue-800'
  },
  {
    key: 'flagged',
    label: t('feature.noteMgmt.flaggedNotes'),
    icon: HiOutlineFlag,
    description: t('feature.noteMgmt.flaggedNotesDesc'),
    color: 'text-red-500',
    bgColor: 'bg-red-50 dark:bg-red-900/20',
    hoverBg: 'hover:bg-red-50 dark:hover:bg-red-900/20',
    activeBg: 'bg-red-100 dark:bg-red-900/30',
    borderColor: 'border-red-200 dark:border-red-800'
  },
  {
    key: 'by-date',
    label: t('feature.noteMgmt.byCreationDate'),
    icon: HiOutlineCalendar,
    description: t('feature.noteMgmt.byCreationDateDesc'),
    color: 'text-green-500',
    bgColor: 'bg-green-50 dark:bg-green-900/20',
    hoverBg: 'hover:bg-green-50 dark:hover:bg-green-900/20',
    activeBg: 'bg-green-100 dark:bg-green-900/30',
    borderColor: 'border-green-200 dark:border-green-800'
  }
];
```

#### 第二个卡片: 笔记导航 (Note Navigation Card)

- 从右侧移动到左侧面板
- 保持现有的 `NoteNavigation` 组件功能
- 调整样式以适配左侧面板

### 右上角创建按钮

```typescript
// 创建按钮位置: 右上角固定位置
const CreateNoteButton = () => (
  <div className="fixed top-20 right-6 z-50">
    <button
      onClick={handleCreateNote}
      className="bg-primary hover:bg-primary-dark text-white p-3 rounded-full shadow-lg transition-all duration-200"
    >
      <HiOutlinePlus className="w-6 h-6" />
    </button>
  </div>
);
```

## 新增 API 需求

### 1. 最近编辑的笔记 API

```typescript
// Route: GET /user/notes/recent-edited
// File: .lnk-et-api-main/src/core.controllers/routes/noteRoutes.ts

router.get('/user/notes/recent-edited', auth, validateRequest(noteSchemas.getRecentEditedNotes), getRecentEditedNotes);
```

**API 规范:**
- **路径**: `/user/notes/recent-edited`
- **方法**: GET
- **参数**: 
  - `limit`: number (默认 20, 最大 50)
  - `offset`: number (默认 0)
- **响应**: 按 `updatedAt` 降序排列的笔记列表

**Controller 实现:**
```typescript
export const getRecentEditedNotes = async (req: Request, res: Response) => {
  try {
    const { limit = 20, offset = 0 } = req.query;
    const userId = req.user._id;
    
    const notes = await Note.find({ 
      userId,
      isTodoNote: false // 排除待办笔记
    })
    .sort({ updatedAt: -1 })
    .limit(Number(limit))
    .skip(Number(offset))
    .populate('project')
    .populate('activityCategory')
    .populate('noteTags');
    
    res.json(notes);
  } catch (error) {
    res.status(500).json({ message: 'Failed to fetch recent edited notes' });
  }
};
```

### 2. 标记笔记 API

```typescript
// Route: GET /user/notes/flagged
// File: .lnk-et-api-main/src/core.controllers/routes/noteRoutes.ts

router.get('/user/notes/flagged', auth, validateRequest(noteSchemas.getFlaggedNotes), getFlaggedNotes);
```

**API 规范:**
- **路径**: `/user/notes/flagged`
- **方法**: GET
- **参数**: 
  - `limit`: number (默认 50, 最大 100)
  - `offset`: number (默认 0)
- **响应**: 按 `updatedAt` 降序排列的已标记笔记列表

**Controller 实现:**
```typescript
export const getFlaggedNotes = async (req: Request, res: Response) => {
  try {
    const { limit = 50, offset = 0 } = req.query;
    const userId = req.user._id;
    
    const notes = await Note.find({ 
      userId,
      isFlagged: true,
      isTodoNote: false // 排除待办笔记
    })
    .sort({ updatedAt: -1 })
    .limit(Number(limit))
    .skip(Number(offset))
    .populate('project')
    .populate('activityCategory')
    .populate('noteTags');
    
    res.json(notes);
  } catch (error) {
    res.status(500).json({ message: 'Failed to fetch flagged notes' });
  }
};
```

### 3. 按创建日期查询笔记 API

```typescript
// Route: GET /user/notes/by-creation-date
// File: .lnk-et-api-main/src/core.controllers/routes/noteRoutes.ts

router.get('/user/notes/by-creation-date', auth, validateRequest(noteSchemas.getNotesByCreationDate), getNotesByCreationDate);
```

**API 规范:**
- **路径**: `/user/notes/by-creation-date`
- **方法**: GET
- **参数**: 
  - `date`: string (YYYY-MM-DD 格式)
  - `limit`: number (默认 50, 最大 100)
  - `offset`: number (默认 0)
- **响应**: 指定日期创建的笔记列表，按创建时间排序

**Controller 实现:**
```typescript
export const getNotesByCreationDate = async (req: Request, res: Response) => {
  try {
    const { date, limit = 50, offset = 0 } = req.query;
    const userId = req.user._id;
    
    if (!date) {
      return res.status(400).json({ message: 'Date parameter is required' });
    }
    
    const startDate = new Date(date as string);
    const endDate = new Date(startDate);
    endDate.setDate(endDate.getDate() + 1);
    
    const notes = await Note.find({ 
      userId,
      isTodoNote: false, // 排除待办笔记
      userDate: {
        $gte: startDate,
        $lt: endDate
      }
    })
    .sort({ userDate: -1, createdAt: -1 })
    .limit(Number(limit))
    .skip(Number(offset))
    .populate('project')
    .populate('activityCategory')
    .populate('noteTags');
    
    res.json(notes);
  } catch (error) {
    res.status(500).json({ message: 'Failed to fetch notes by creation date' });
  }
};
```

### 4. Schema 验证

```typescript
// File: .lnk-et-api-main/src/middleware/validators/schemas/noteSchemas.ts

export const noteSchemas = {
  // ... 现有 schemas
  
  getRecentEditedNotes: {
    query: Joi.object({
      limit: Joi.number().min(1).max(50).default(20),
      offset: Joi.number().min(0).default(0)
    })
  },
  
  getFlaggedNotes: {
    query: Joi.object({
      limit: Joi.number().min(1).max(100).default(50),
      offset: Joi.number().min(0).default(0)
    })
  },
  
  getNotesByCreationDate: {
    query: Joi.object({
      date: Joi.string().pattern(/^\d{4}-\d{2}-\d{2}$/).required(),
      limit: Joi.number().min(1).max(100).default(50),
      offset: Joi.number().min(0).default(0)
    })
  }
};
```

## 移动端设计

### 移动端布局调整

参考 `TaskManagementHomePage` 的移动端设计模式：

```typescript
const MobileQuerySelector = () => (
  <div className="mb-6">
    <div className="flex bg-gray-50 dark:bg-gray-800 rounded-lg p-1 mb-3">
      {queryMethods.map((method) => {
        const IconComponent = method.icon;
        return (
          <button
            key={method.key}
            onClick={() => handleQueryMethodChange(method.key)}
            className={`flex-1 flex items-center justify-center space-x-2 py-3 px-4 rounded-md transition-all duration-200 ${
              activeQueryMethod === method.key
                ? 'bg-white dark:bg-gray-700 text-primary-deep shadow-sm'
                : 'text-gray-600 dark:text-gray-400 hover:text-gray-900 dark:hover:text-gray-200'
            }`}
          >
            <IconComponent className="w-5 h-5" />
            <span className="font-medium text-sm">{method.label}</span>
          </button>
        );
      })}
    </div>
  </div>
);
```

### 移动端特殊考虑

- **不显示笔记导航**: 移动端屏幕空间有限，不显示 NoteNavigation 组件
- **简化查询选择器**: 使用水平滑动的选择器
- **浮动创建按钮**: 创建按钮位于右下角，适合移动端操作

## 状态管理

### 新增状态

```typescript
interface NoteQueryState {
  activeQueryMethod: 'recent' | 'flagged' | 'by-date';
  queryDate?: Date; // 用于按日期查询
  notes: INoteHydrated[];
  loading: boolean;
  error: string | null;
}

// 新增 Store
export const useNoteQueryStore = create<NoteQueryState>((set, get) => ({
  activeQueryMethod: 'recent',
  queryDate: undefined,
  notes: [],
  loading: false,
  error: null,
  
  setActiveQueryMethod: (method: 'recent' | 'flagged' | 'by-date') => {
    set({ activeQueryMethod: method });
  },
  
  setQueryDate: (date: Date) => {
    set({ queryDate: date });
  },
  
  setNotes: (notes: INoteHydrated[]) => {
    set({ notes });
  },
  
  setLoading: (loading: boolean) => {
    set({ loading });
  },
  
  setError: (error: string | null) => {
    set({ error });
  }
}));
```

### URL 参数管理

```typescript
// URL 参数结构
interface URLParams {
  queryMethod?: 'recent' | 'flagged' | 'by-date';
  date?: string; // YYYY-MM-DD 格式
  category?: TabCategory;
  // 保留现有搜索参数
  searchText?: string;
  // ... 其他搜索相关参数
}
```

## 实现计划

### 阶段 1: 后端 API 开发

1. **创建新的 API 路由**
   - [ ] 添加三个新的路由到 `noteRoutes.ts`
   - [ ] 实现对应的 Controller 函数
   - [ ] 添加 Schema 验证

2. **测试 API**
   - [ ] 编写单元测试
   - [ ] 测试 API 响应和性能

### 阶段 2: 前端状态管理

1. **创建新的 Store**
   - [ ] 实现 `useNoteQueryStore`
   - [ ] 添加相关的 service 函数

2. **URL 状态同步**
   - [ ] 实现 URL 参数与状态的双向绑定
   - [ ] 保持页面刷新后状态一致

### 阶段 3: UI 组件开发

1. **左侧面板组件**
   - [ ] 创建 `QueryMethodsCard` 组件
   - [ ] 移动 `NoteNavigation` 组件到左侧
   - [ ] 实现桌面端布局

2. **移动端适配**
   - [ ] 实现 `MobileQuerySelector` 组件
   - [ ] 适配移动端布局

3. **创建按钮**
   - [ ] 实现右上角浮动创建按钮
   - [ ] 处理创建后的视图切换逻辑

### 阶段 4: 功能集成

1. **查询方法切换**
   - [ ] 实现不同查询方法的数据加载逻辑
   - [ ] 保持搜索功能不受影响

2. **创建笔记流程**
   - [ ] 实现创建后自动切换到 "最近编辑" 视图
   - [ ] 新创建的笔记自动显示在列表顶部

### 阶段 5: 测试和优化

1. **功能测试**
   - [ ] 测试所有查询方法
   - [ ] 测试移动端和桌面端布局
   - [ ] 测试搜索功能兼容性

2. **性能优化**
   - [ ] 优化 API 查询性能
   - [ ] 实现适当的缓存机制

## 技术考虑

### 性能优化

1. **分页加载**: 所有查询方法都支持分页
2. **缓存策略**: 实现适当的前端缓存避免重复请求
3. **懒加载**: 在用户切换查询方法时才加载对应数据

### 用户体验

1. **加载状态**: 在数据加载时显示适当的加载指示器
2. **错误处理**: 优雅地处理 API 错误
3. **状态保持**: 保持用户的查询偏好和滚动位置

### 兼容性

1. **向后兼容**: 保持现有搜索功能完全不变
2. **渐进式增强**: 新功能不影响现有用户工作流
3. **响应式设计**: 确保在所有设备上都有良好体验

## 更新历史

| 日期 | 版本 | 更新内容 | 作者 |
|------|------|----------|------|
| 2024-01-XX | 1.0 | 初始设计文档 | 开发团队 |

---

本文档将随着实现过程的进展持续更新和完善。 