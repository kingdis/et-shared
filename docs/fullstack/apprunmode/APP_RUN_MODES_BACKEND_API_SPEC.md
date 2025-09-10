## App Run Modes Backend API Spec (Desktop-first)

本规范用于支持桌面端“运行模式与能力范围管控”的前端实现，定义所需的后端接口契约与审计要求。

### 1. 账户级运行模式接口 `/user/account-mode`

- Method: GET/POST
- Purpose: 读取/更新“应用运行模式（AppRunMode）”。此信息与用户账户绑定（userId 维度），而非偏好（dataGroup 维度）。

#### 1.1 数据模型（仅账户级 appRunMode）

```json
{
  "appRunMode": {
    "accountMode": "PERSONAL|PARENTAL|DUAL",
    "appView": "self_mangement|parental_control|self_mangement_child",
    "enableSelfJournaling": true
  },
  "_meta": { "version": 1 }
}
```

- 字段说明（位于 `appRunMode` 下）：
  - `accountMode`: PERSONAL/PARENTAL/DUAL，用于从根本上区分应用运行模式（账户级）
  - `appView`：不在后端存储。由前端通过 cookies 记录当前视图取向；与 `accountMode` 的关系：
    - 当 accountMode=PERSONAL → 前端应固定 appView=self_mangement（无需回传）
    - 当 accountMode=PARENTAL → 前端应固定 appView=parental_control（无需回传）
    - 当 accountMode=DUAL → 前端可在 cookies 中保存 appView（self_mangement 或 parental_control），无需持久化到后端
  - `enableSelfJournaling`: 是否开启“自我日志”能力（仅当 accountMode=DUAL 且 appView=self_mangement 时生效；其他情形忽略该设置）
  - 选中孩子上下文：不在本对象中存储；由前端 cookies 保存“最近选择的孩子 dataGroup”，必要时配合 `/auth/take-over` 切换。

实现说明：
- 后端将账户级 `appRunMode` 存储在用户文档 `User.appRunMode` 字段（非独立集合）。
- `appView` 不在后端持久化；GET 响应会基于 `accountMode` 派生一个 `appView` 字段用于前端初始化（PERSONAL→self_mangement；PARENTAL→parental_control；DUAL→由前端 cookies 决定，若缺省则默认为 self_mangement）。
- 前端在检测到“用户属于某家庭且其角色为 child”时，应将 `appView` 规范化为 `self_mangement_child`（此值仍不由后端存储，仅作为前端视图信号）。

#### 1.2 GET `/user/account-mode`

- Response 200
```json
{
  "appRunMode": {
    "accountMode": "PERSONAL",
    "appView": "self_mangement",
    "enableSelfJournaling": true
  },
  "_meta": { "version": 1 }
}
```

#### 1.3 POST `/user/account-mode`（仅浅合并更新）

- Request Body: 部分字段更新，服务端需做浅合并（仅更新传入字段）。
```json
{
  "appRunMode": {
    "accountMode": "DUAL",
    "appView": "parental_control",
    "enableSelfJournaling": false
  }
}
```

也支持“扁平”提交（两种皆可）：
```json
{
  "accountMode": "DUAL",
  "enableSelfJournaling": false
}
```

- 服务端将忽略传入的 `appView` 字段（不持久化），仅对 `accountMode` 与 `enableSelfJournaling` 做浅合并更新。

- Response 200: 返回合并后的完整账户级 appRunMode 对象（建议）。

> 说明：AppRunMode 存储与返回均以“账户级”维度进行；请勿再放置于 `/user/preference`（dataGroup 维度）下。

### 2. 会话切换 `/auth/take-over`

- Method: POST
- Purpose: 切换会话 Token 的 `dataGroup`

#### 2.1 请求与语义

```json
// 切到孩子
{ "id": "<childDataGroup>" }

// 切回本人
{}
```

#### 2.2 响应

- Response 200
```json
{
  "token": "<jwt>",
  "user": {
    "userId": "...",         
    "userName": "...",
    "email": "...",
    "avatar": "...",
    "authority": ["..."],     
    "dataGroup": "<currentDataGroup>",
    "children": [
      { "firstName": "...", "lastName": "...", "dataGroup": "..." }
    ]
  }
}
```

说明：`authority` 等价于 `roles`，保留以兼容前端历史字段。

### 3. 鉴权与审计要求

- 必须校验当前用户对目标 `dataGroup` 是否具备家长控制权限；非法请求返回 403。
- 记录审计日志：源用户、目标 `dataGroup`、时间、来源客户端（可选）、上下文描述（可选）。
- 频率限制：对单用户短时间内频繁切换可加限流（可选）。

### 4. 兼容性与新增后端约束

- 前端将移除 `user.isParentalControlMode` 的使用，统一以 `accountMode + appView` 推导视角；后端无需返回该字段。
- 现有 `dataGroup` 授权与资源隔离不变。

#### 新增约束（邀请为孩子）：
- 在发出邀请（createInvitation）与接受邀请（acceptInvitation）时，被邀请者/受邀者账户的 `appRunMode.accountMode` 必须为 `PERSONAL`，否则后端拒绝。
- 防止孩子创建家庭组（推荐）：当调用 createFamilyGroup 时，如当前用户在任一家庭组中以 `child` 角色存在，则拒绝创建。


### 5. 错误码建议

- 400: 参数不合法（如无效的 `dataGroup`）
- 403: 未授权操作（无家长控制权限）
- 409: 当前已处于目标会话，无需切换（可选）
- 500: 服务器内部错误


