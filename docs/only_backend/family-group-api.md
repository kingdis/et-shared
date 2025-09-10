# 家庭组管理 API 文档

## 概述

家庭组管理系统允许用户创建家庭组、邀请成员、管理成员权限等功能。每个用户只能拥有一个家庭组，但可以通过邀请加入其他家庭组。

## 基础信息

- **Base URL**: `/api/user/family-group`
- **认证方式**: Bearer Token
- **数据格式**: JSON

## 数据模型

### FamilyGroup 家庭组
```typescript
{
  _id: string;                    // 家庭组ID
  name: string;                   // 家庭组名称
  description?: string;           // 描述
  members: FamilyGroupMember[];   // 成员列表
  settings: {
    allowChildrenToInvite: boolean;     // 是否允许子用户发送邀请
    requireApprovalForJoin: boolean;    // 是否需要审批加入
    maxMembers: number;                 // 最大成员数量 (2-50)
  };
  dataGroup: string;             // 数据组ID（用于确定管理员身份）
  isActive: boolean;             // 是否激活
  createdAt: Date;              // 创建时间
  updatedAt: Date;              // 更新时间
}
```

### FamilyGroupMember 家庭组成员
```typescript
{
  user: {                       // 用户信息 (通过MongoDB populate自动获取)
    _id: string;                // 用户ID
    email: string;              // 用户邮箱地址
    name: string;               // 用户姓名
  };
  role: 'parent' | 'child';     // 角色（只有父母和子女两种角色）
  alias?: string;               // 别名
  joinedAt: Date;              // 加入时间
  isActive: boolean;           // 是否激活
}
```

**注意：** 
- `user` 字段使用 MongoDB 的 `populate` 功能自动填充用户信息，包括邮箱地址和姓名。这确保了数据的一致性和实时性。
- **管理员身份**：通过 `dataGroup` 字段隐含确定。拥有该 `dataGroup` 的用户即为管理员，无需在 `role` 字段中显式标记。
- **创建者角色**：家庭组创建者会自动以 `parent` 角色加入 `members` 数组，同时通过 `dataGroup` 拥有管理员权限。

### FamilyGroupInvitation 邀请
```typescript
{
  _id: string;                  // 邀请ID
  familyGroup: string;          // 家庭组ID
  inviter: string;              // 邀请人用户ID
  inviteeEmail: string;         // 被邀请人邮箱
  invitee?: string;             // 被邀请人用户ID (接受后填充)
  role: 'parent' | 'child';     // 邀请角色
  alias?: string;               // 别名
  status: 'pending' | 'accepted' | 'rejected' | 'expired' | 'cancelled';  // 状态
  expiresAt: Date;             // 过期时间 (7天)
  dataGroup: string;           // 数据组ID
  createdAt: Date;             // 创建时间
  cancelledAt?: Date;          // 取消时间 (仅当状态为cancelled时)
}
```

## API Endpoints

### 1. 家庭组 CRUD 操作

#### 1.1 创建家庭组
```http
POST /api/user/family-group
```

**请求体:**
```json
{
  "name": "我的家庭",
  "description": "温馨的家庭组",
  "settings": {
    "allowChildrenToInvite": false,
    "requireApprovalForJoin": true,
    "maxMembers": 10
  }
}
```

**响应 (201):**
```json
{
  "_id": "family-group-uuid",
  "name": "我的家庭",
  "description": "温馨的家庭组",
  "members": [
    {
      "user": {
        "_id": "creator-user-uuid",
        "email": "dad@example.com",
        "name": "爸爸"
      },
      "role": "parent",
      "alias": "爸爸",
      "joinedAt": "2025-05-30T20:00:00.000Z",
      "isActive": true
    }
  ],
  "settings": {
    "allowChildrenToInvite": false,
    "requireApprovalForJoin": true,
    "maxMembers": 10
  },
  "dataGroup": "creator-datagroup-uuid",
  "isActive": true,
  "createdAt": "2025-05-30T20:00:00.000Z",
  "updatedAt": "2025-05-30T20:00:00.000Z"
}
```

**错误响应 (409):**
```json
{
  "status": "error",
  "code": "ALREADY_EXISTS",
  "fields": ["familyGroup"],
  "details": {
    "familyGroup": "User already has a family group"
  }
}
```

#### 1.2 获取用户的家庭组
```http
GET /api/user/family-group
```

**响应 (200) - 有家庭组:**
```json
{
  "_id": "family-group-uuid",
  "name": "我的家庭",
  "description": "温馨的家庭组",
  "members": [
    {
      "user": {
        "_id": "creator-user-uuid",
        "email": "dad@example.com",
        "name": "爸爸"
      },
      "role": "parent",
      "alias": "爸爸",
      "joinedAt": "2025-05-30T20:00:00.000Z",
      "isActive": true
    },
    {
      "user": {
        "_id": "child-user-uuid",
        "email": "child@example.com",
        "name": "小明"
      },
      "role": "child",
      "alias": "小明",
      "joinedAt": "2025-05-30T21:00:00.000Z",
      "isActive": true
    }
  ],
  "settings": {
    "allowChildrenToInvite": false,
    "requireApprovalForJoin": true,
    "maxMembers": 10
  },
  "dataGroup": "creator-datagroup-uuid",
  "isActive": true,
  "createdAt": "2025-05-30T20:00:00.000Z",
  "updatedAt": "2025-05-30T21:00:00.000Z"
}
```

**响应 (200) - 无家庭组:**
```json
null
```

#### 1.3 更新家庭组
```http
PUT /api/user/family-group
```

**请求体:**
```json
{
  "name": "更新的家庭名称",
  "description": "新的描述",
  "settings": {
    "allowChildrenToInvite": true,
    "maxMembers": 15
  }
}
```

**响应 (200):** 返回更新后的家庭组对象

**错误响应 (400):**
```json
{
  "status": "error",
  "code": "INVALID_PARAMS",
  "fields": ["familyGroup"],
  "details": {
    "familyGroup": "You must create a family group first before updating it"
  }
}
```

#### 1.4 删除家庭组
```http
DELETE /api/user/family-group
```

**响应 (200):**
```json
{
  "message": "Family group deleted successfully"
}
```

### 2. 邀请管理

#### 2.1 创建邀请
```http
POST /api/user/family-group/invitations
```

**请求体:**
```json
{
  "inviteeEmail": "child@example.com",
  "role": "child",
  "alias": "小明"
}
```

**响应 (201):**
```json
{
  "_id": "invitation-uuid",
  "familyGroup": "family-group-uuid",
  "inviter": {
    "_id": "inviter-user-uuid",
    "email": "dad@example.com",
    "name": "爸爸"
  },
  "inviteeEmail": "child@example.com",
  "role": "child",
  "alias": "小明",
  "status": "pending",
  "expiresAt": "2025-06-06T20:00:00.000Z",
  "dataGroup": "creator-datagroup-uuid",
  "createdAt": "2025-05-30T20:00:00.000Z"
}
```

**错误响应:**
- **409** - 邀请已存在
- **403** - 权限不足或达到成员上限
- **400** - 用户没有家庭组

#### 2.2 获取家庭组的邀请列表 (邀请方查看)
```http
GET /api/user/family-group/invitations
```

**响应 (200):**
```json
[
  {
    "_id": "invitation-uuid",
    "familyGroup": "family-group-uuid",
    "inviter": {
      "_id": "inviter-user-uuid",
      "email": "dad@example.com",
      "name": "爸爸"
    },
    "inviteeEmail": "child@example.com",
    "role": "child",
    "alias": "小明",
    "status": "pending",
    "expiresAt": "2025-06-06T20:00:00.000Z",
    "dataGroup": "creator-datagroup-uuid",
    "createdAt": "2025-05-30T20:00:00.000Z"
  }
]
```

**响应 (200) - 无家庭组:**
```json
[]
```

#### 2.3 获取用户的待处理邀请 (被邀请方查看)
```http
GET /api/user/family-group/invitations/pending
```

**响应 (200):**
```json
[
  {
    "_id": "invitation-uuid",
    "familyGroup": {
      "_id": "family-group-uuid",
      "name": "张家大院",
      "description": "温馨的家庭"
    },
    "inviter": {
      "_id": "inviter-user-uuid",
      "email": "dad@example.com",
      "name": "爸爸"
    },
    "inviteeEmail": "myemail@example.com",
    "role": "child",
    "alias": "小明",
    "status": "pending",
    "expiresAt": "2025-06-06T20:00:00.000Z",
    "dataGroup": "creator-datagroup-uuid",
    "createdAt": "2025-05-30T20:00:00.000Z"
  }
]
```

#### 2.4 接受邀请
```http
POST /api/user/family-group/invitations/{invitationId}/accept
```

**响应 (200):**
```json
{
  "familyGroup": {
    "_id": "family-group-uuid",
    "name": "张家大院",
    "members": [
      {
        "user": {
          "_id": "creator-user-uuid",
          "email": "dad@example.com",
          "name": "爸爸"
        },
        "role": "parent",
        "alias": "爸爸",
        "joinedAt": "2025-05-30T20:00:00.000Z",
        "isActive": true
      },
      {
        "user": {
          "_id": "current-user-uuid",
          "email": "child@example.com",
          "name": "小明"
        },
        "role": "child",
        "alias": "小明",
        "joinedAt": "2025-05-30T21:00:00.000Z",
        "isActive": true
      }
    ],
    "dataGroup": "creator-datagroup-uuid",
    // ... 其他家庭组信息
  },
  "invitation": {
    "_id": "invitation-uuid",
    "status": "accepted",
    "invitee": "current-user-uuid",
    // ... 其他邀请信息
  }
}
```

**错误响应:**
- **404** - 邀请不存在、已过期或已处理
- **409** - 用户已是成员或达到成员上限

#### 2.5 拒绝邀请
```http
POST /api/user/family-group/invitations/{invitationId}/reject
```

**响应 (200):**
```json
{
  "_id": "invitation-uuid",
  "status": "rejected",
  "invitee": "current-user-uuid",
  // ... 其他邀请信息
}
```

#### 2.6 取消邀请 (邀请方取消)
```http
DELETE /api/user/family-group/invitations/{invitationId}
```

**说明:** 邀请方可以取消自己发出的待处理邀请

**响应 (200):**
```json
{
  "_id": "invitation-uuid",
  "status": "cancelled",
  "familyGroup": "family-group-uuid",
  "inviter": "inviter-user-uuid",
  "inviteeEmail": "child@example.com",
  "role": "child",
  "alias": "小明",
  "expiresAt": "2025-06-06T20:00:00.000Z",
  "dataGroup": "creator-datagroup-uuid",
  "createdAt": "2025-05-30T20:00:00.000Z",
  "cancelledAt": "2025-05-31T10:00:00.000Z"
}
```

**错误响应:**
- **404** - 邀请不存在或已过期
- **403** - 只有邀请发送者可以取消邀请
- **400** - 邀请已被接受或拒绝，无法取消

### 3. 成员管理

#### 3.1 更新成员信息
```http
PUT /api/user/family-group/members
```

**请求体:**
```json
{
  "user": "member-user-uuid",
  "role": "parent",
  "alias": "妈妈",
  "isActive": true
}
```

**响应 (200):** 返回更新后的家庭组对象

**错误响应:**
- **403** - 只有管理员可以更新成员
- **400** - 用户没有家庭组

#### 3.2 移除成员
```http
DELETE /api/user/family-group/members/{user}
```

**响应 (200):**
```json
{
  "message": "Member removed successfully",
  "familyGroup": {
    // ... 更新后的家庭组信息
  }
}
```

**错误响应:**
- **403** - 权限不足或管理员不能移除自己
- **400** - 用户没有家庭组

## 完整业务流程示例

### 场景：爸爸创建家庭组并邀请孩子加入

#### 步骤1: 爸爸创建家庭组
```http
POST /api/user/family-group
Authorization: Bearer <爸爸的token>

{
  "name": "张家大院",
  "description": "我们温馨的家",
  "settings": {
    "allowChildrenToInvite": false,
    "requireApprovalForJoin": true,
    "maxMembers": 5
  }
}
```

**响应:** 创建成功，爸爸自动以 `parent` 角色加入成员列表，同时通过 `dataGroup` 拥有管理员权限

#### 步骤2: 爸爸发送邀请给孩子
```http
POST /api/user/family-group/invitations
Authorization: Bearer <爸爸的token>

{
  "inviteeEmail": "xiaoming@example.com",
  "role": "child",
  "alias": "小明"
}
```

**响应:** 邀请创建成功

#### 步骤3: 爸爸查看发出的邀请
```http
GET /api/user/family-group/invitations
Authorization: Bearer <爸爸的token>
```

**响应:** 显示所有发出的邀请列表，包括待处理的邀请

#### 步骤4: 孩子查看收到的邀请
```http
GET /api/user/family-group/invitations/pending
Authorization: Bearer <孩子的token>
```

**响应:** 显示孩子收到的所有待处理邀请

#### 步骤5: 孩子接受邀请
```http
POST /api/user/family-group/invitations/{invitationId}/accept
Authorization: Bearer <孩子的token>
```

**响应:** 接受成功，返回家庭组信息和邀请状态

#### 步骤6: 双方查看家庭组成员
**重要：被邀请人接受邀请后，必须调用此API来查看自己所在的家庭组**

```http
GET /api/user/family-group
Authorization: Bearer <爸爸的token>
```

```http
GET /api/user/family-group
Authorization: Bearer <孩子的token>
```

**响应:** 两人都能看到完整的家庭组信息，包括所有成员列表：
- 爸爸 (parent角色，同时是管理员)
- 小明 (child角色)

### 重要说明

**对于被邀请人：**
1. 接受邀请后，您已经成功加入家庭组
2. 要查看您所在的家庭组和所有成员，请调用：`GET /api/user/family-group`
3. 这个API会返回您所属的家庭组的完整信息，包括所有成员列表
4. 如果返回 `null`，说明您当前不属于任何家庭组

**对于邀请人：**
1. 被邀请人接受邀请后，您可以通过 `GET /api/user/family-group` 看到更新后的成员列表
2. 新成员会立即显示在家庭组的 `members` 数组中

### 场景：邀请方取消邀请

#### 步骤1: 爸爸查看发出的邀请
```http
GET /api/user/family-group/invitations
Authorization: Bearer <爸爸的token>
```

**响应:** 显示所有发出的邀请列表，包括待处理的邀请

#### 步骤2: 爸爸取消某个邀请
```http
DELETE /api/user/family-group/invitations/{invitationId}
Authorization: Bearer <爸爸的token>
```

**响应:** 邀请状态更新为 `cancelled`，被邀请人将无法再接受此邀请

#### 步骤3: 被邀请人查看邀请状态
```http
GET /api/user/family-group/invitations/pending
Authorization: Bearer <孩子的token>
```

**响应:** 已取消的邀请不会出现在待处理邀请列表中

**注意事项:**
- 只有邀请发送者可以取消自己发出的邀请
- 只能取消状态为 `pending` 的邀请
- 已被接受或拒绝的邀请无法取消
- 取消后的邀请状态变为 `cancelled`，并记录取消时间

## 权限说明

### 管理员身份确定
**管理员身份**通过 `dataGroup` 字段隐含确定：
- 拥有家庭组 `dataGroup` 的用户即为管理员
- 管理员检查：`user.dataGroup === familyGroup.dataGroup`
- 管理员同时会以 `parent` 角色存在于 `members` 数组中

### 角色权限
- **管理员（通过dataGroup确定）**: 可以管理家庭组设置、邀请成员、管理所有成员、删除家庭组
- **parent**: 可以邀请成员、查看家庭组信息
- **child**: 基础成员权限，默认不能邀请成员（除非管理员启用 `allowChildrenToInvite` 设置）

### 操作权限详细说明

#### 家庭组管理
- **创建家庭组**: 任何用户都可以创建（但每个用户只能拥有一个）
- **更新家庭组**: 仅管理员（dataGroup拥有者）
- **删除家庭组**: 仅管理员（dataGroup拥有者）
- **查看家庭组**: 所有成员

#### 邀请管理
- **发送邀请**: 
  - 管理员（dataGroup拥有者）：总是可以发送邀请
  - 父母（parent）：总是可以发送邀请
  - 子用户（child）：只有在管理员启用 `allowChildrenToInvite` 设置时才能发送邀请
- **取消邀请**: 仅邀请发送者可以取消自己发出的待处理邀请
- **查看邀请列表**: 管理员和父母可以查看自己家庭组的邀请
- **查看待处理邀请**: 被邀请人可以查看发给自己的邀请
- **接受/拒绝邀请**: 仅被邀请人

#### 成员管理
- **更新成员信息**: 仅管理员（dataGroup拥有者）
- **移除成员**: 
  - 管理员可以移除任何成员（但不能移除自己）
  - 成员可以移除自己

### 权限控制示例

#### 邀请权限控制
```json
{
  "settings": {
    "allowChildrenToInvite": false  // 默认值，子用户不能发送邀请
  }
}
```

当 `allowChildrenToInvite` 为 `false` 时：
- ✅ 管理员（dataGroup拥有者）可以发送邀请
- ✅ parent 可以发送邀请  
- ❌ child 不能发送邀请

当 `allowChildrenToInvite` 为 `true` 时：
- ✅ 管理员（dataGroup拥有者）可以发送邀请
- ✅ parent 可以发送邀请
- ✅ child 可以发送邀请

#### 删除权限控制
- ✅ 只有家庭组的管理员（dataGroup拥有者）可以删除家庭组
- ❌ parent 和 child 角色无法删除家庭组

#### 管理员权限检查示例
```typescript
// 检查用户是否为家庭组管理员
function isAdmin(user: AuthUser, familyGroup: FamilyGroup): boolean {
  return user.dataGroup === familyGroup.dataGroup;
}

// 权限控制示例
if (!isAdmin(currentUser, familyGroup)) {
  throw new Error('Only admin can perform this operation');
}
```

### 操作权限总结
- **创建/更新/删除家庭组**：仅管理员（dataGroup拥有者）
- **发送邀请**：管理员、父母，或有权限的子用户
- **管理成员**：仅管理员（dataGroup拥有者）
- **移除成员**：管理员可移除任何人（除自己），成员可移除自己

## 错误码说明

- **INVALID_PARAMS**: 参数错误或用户状态不符合操作要求
- **ALREADY_EXISTS**: 资源已存在（如用户已有家庭组、邀请已存在）
- **NOT_FOUND**: 资源不存在
- **FORBIDDEN**: 权限不足

## 注意事项

1. 每个用户只能拥有一个家庭组
2. 邀请有效期为7天
3. 家庭组成员数量限制为2-50人
4. 删除家庭组为软删除，数据会保留
5. 移除成员为软删除，可以重新激活
6. 管理员身份通过 `dataGroup` 隐含确定，无需在角色中显式标记
7. 创建者会自动以 `parent` 角色加入成员列表，同时拥有管理员权限 