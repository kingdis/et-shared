# ActivityLog Rating字段 - 前端开发指南

## 概述

ActivityLog的每个`executedActivity`现在都包含一个必填的`rating`字段，用于记录星级评分。

## 数据结构

### IRating接口

```typescript
interface IRating {
  overall: number;        // 0-5 总体评分星级，0表示未评分
  timestamp?: Date;       // 评分时间（可选，暂时不使用）
  notes?: string;         // 评分备注（可选）
  dimensions?: {          // 多维度评分（可选，面向未来）
    [key: string]: number;
  };
}
```

### IExecutedActivity更新

```typescript
interface IExecutedActivity {
  _id?: string;
  activityCategory: string;
  // ... 其他字段
  rating: IRating; // 新增：必填的星级评分字段
}
```

## 数据示例

### 未评分状态（默认）
```json
{
  "_id": "activity_123",
  "activityCategory": "work",
  "rating": {
    "overall": 0
  }
}
```

### 已评分状态
```json
{
  "_id": "activity_123", 
  "activityCategory": "work",
  "rating": {
    "overall": 4,
    "notes": "工作效率很高"
  }
}
```

## API端点

### 更新评分
```http
PUT /user/log/activity-log/executed-activity/rating
Content-Type: application/json

{
  "activityLogId": "log_123",
  "executedActivityId": "activity_456", 
  "rating": {
    "overall": 4,
    "notes": "很好的体验"
  }
}
```

### 重置评分（设为未评分）
```http
PUT /user/log/activity-log/executed-activity/rating/reset
Content-Type: application/json

{
  "activityLogId": "log_123",
  "executedActivityId": "activity_456"
}
```

## 前端实现建议

### 1. 星级显示组件

```tsx
interface RatingDisplayProps {
  rating: IRating;
  readonly?: boolean;
}

const RatingDisplay: React.FC<RatingDisplayProps> = ({ rating, readonly = false }) => {
  return (
    <div className="rating-display">
      <StarRating 
        value={rating.overall} 
        max={5}
        readonly={readonly}
        showZero={true} // 显示0星状态
      />
      {rating.notes && (
        <div className="rating-notes">{rating.notes}</div>
      )}
    </div>
  );
};
```

### 2. 评分编辑组件

```tsx
interface RatingEditorProps {
  rating: IRating;
  onSave: (rating: IRating) => void;
  onReset: () => void;
}

const RatingEditor: React.FC<RatingEditorProps> = ({ rating, onSave, onReset }) => {
  const [localRating, setLocalRating] = useState(rating);

  const handleSave = () => {
    onSave(localRating);
  };

  const handleReset = () => {
    onReset();
    setLocalRating({ overall: 0 });
  };

  return (
    <div className="rating-editor">
      <StarRating 
        value={localRating.overall}
        onChange={(value) => setLocalRating({...localRating, overall: value})}
        max={5}
      />
      <textarea 
        value={localRating.notes || ''}
        onChange={(e) => setLocalRating({...localRating, notes: e.target.value})}
        placeholder="评分备注（可选）"
        maxLength={500}
      />
      <div className="rating-actions">
        <button onClick={handleSave}>保存评分</button>
        <button onClick={handleReset}>重置</button>
      </div>
    </div>
  );
};
```

### 3. API调用示例

```typescript
// 更新评分
const updateRating = async (activityLogId: string, executedActivityId: string, rating: IRating) => {
  try {
    const response = await fetch('/user/log/activity-log/executed-activity/rating', {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        activityLogId,
        executedActivityId,
        rating
      })
    });
    
    if (!response.ok) throw new Error('更新评分失败');
    return await response.json();
  } catch (error) {
    console.error('更新评分错误:', error);
    throw error;
  }
};

// 重置评分
const resetRating = async (activityLogId: string, executedActivityId: string) => {
  try {
    const response = await fetch('/user/log/activity-log/executed-activity/rating/reset', {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        activityLogId,
        executedActivityId
      })
    });
    
    if (!response.ok) throw new Error('重置评分失败');
    return await response.json();
  } catch (error) {
    console.error('重置评分错误:', error);
    throw error;
  }
};
```

## 数据处理注意事项

### 1. 兼容性处理

所有现有数据已通过迁移脚本添加了默认的rating字段，新数据无需特殊处理。

### 2. 评分状态判断

```typescript
// 判断是否已评分
const isRated = (rating: IRating): boolean => {
  return rating.overall > 0;
};

// 获取评分显示文本
const getRatingText = (rating: IRating): string => {
  if (rating.overall === 0) return '未评分';
  return `${rating.overall}星`;
};
```

### 3. 表单验证

```typescript
const validateRating = (rating: IRating): string[] => {
  const errors: string[] = [];
  
  // 验证评分范围
  if (rating.overall < 0 || rating.overall > 5) {
    errors.push('评分必须在0-5之间');
  }
  
  // 验证评分为整数
  if (!Number.isInteger(rating.overall)) {
    errors.push('评分必须是整数');
  }
  
  // 验证备注长度
  if (rating.notes && rating.notes.length > 500) {
    errors.push('评分备注不能超过500字符');
  }
  
  return errors;
};
```

## UI/UX建议

### 1. 视觉设计
- 使用星形图标显示评分
- 0星状态使用灰色或空心星
- 已评分状态使用金色或实心星
- 提供清晰的"未评分"状态指示

### 2. 交互设计
- 点击星星可直接设置评分
- 提供快速重置按钮
- 支持键盘导航（1-5数字键）
- 评分变更时提供即时反馈

### 3. 性能优化
- 评分更新使用防抖处理
- 批量更新时显示进度指示
- 缓存评分数据减少重复请求

## 常见问题

### Q: 如何处理历史数据？
A: 所有历史数据已自动添加默认评分`{overall: 0}`，表示未评分状态。

### Q: 评分是否必须填写？
A: 评分字段是必填的，但`overall: 0`表示未评分状态，这是默认值。

### Q: 是否支持半星评分？
A: 当前只支持整数评分(0-5)，不支持小数。

### Q: 评分备注有长度限制吗？
A: 是的，评分备注最大长度为500字符。

### Q: 未来是否会添加多维度评分？
A: 接口已预留`dimensions`字段用于未来扩展，当前暂不使用。

---

## 迁移检查清单

- [ ] 更新TypeScript类型定义
- [ ] 实现星级显示组件
- [ ] 实现评分编辑功能
- [ ] 添加API调用方法
- [ ] 更新相关的列表和详情页面
- [ ] 测试评分功能的各种状态
- [ ] 验证数据兼容性 