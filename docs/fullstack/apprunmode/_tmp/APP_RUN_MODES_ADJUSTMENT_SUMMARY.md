## 运行模式与儿童视角改造总结

### 前端（UI/状态/显隐）

- 新增视图值：`appView = 'self_mangement_child'`
  - 语义：当账户为 PERSONAL 且属于某家庭组、角色为 child 时，进入“儿童视角”。
  - 视图判定统一用 `accountMode + appView`：
    - `isChildView = (accountMode==='PERSONAL' && appView==='self_mangement_child')`
    - `isParentalView = (accountMode==='PARENTAL') || (accountMode==='DUAL' && appView==='parental_control')`
    - `isPersonalView = (accountMode==='PERSONAL' && appView==='self_mangement')`
  - 移除 `isParentalControlMode` 的使用（以视图判定替代）。

- 会话与切换
  - 移除“activeTarget”概念；仅以 `Token.dataGroup` 表达“当前操作主体”。
  - `/auth/take-over`：
    - 无 id → 指向本人；有 child dataGroup → 指向该孩子。
  - PERSONAL 下即便是 `self_mangement_child`，`Token.dataGroup` 仍指向本人。

- 视图规范化
  - 登录/家庭组加载后：
    - 若当前用户在任一家庭组 `role==='child'` → 设 `appView='self_mangement_child'`；
    - 否则 → 设 `appView='self_mangement'`。
  - `FeatureGate` 将 `self_mangement_child` 与 `parental_control` 同等对待（同属“家长/孩子视角”）做显隐。

- 能力范围与显隐
  - `PERSONAL + self_mangement_child`：
    - 当前界面与“家长控制视角”一致，但不提供账户模式切换（孩子不能切到 PARENTAL/DUAL）。
    - 未来会逐步与家长控制视角拉开能力差异（更细化的限制与差异化体验）。
  - `PERSONAL + self_mangement`：个人功能（提醒、笔记、个人仪表板、个人统计、个人日志等）；隐藏孩子专属能力。
  - `PARENTAL` / `DUAL + parental_control`：显示孩子相关能力；隐藏个人提醒/笔记/个人日志。
  - `DUAL + self_mangement`：依据 `enableSelfJournaling` 决定是否隐藏个人日志体系。

- 组件改动建议
  - 新增 `useViewContext()` 输出 `isChildView / isParentalView / isPersonalView`，替换原 `isParentalControlMode` 分支。
  - `UserDropdown`：`isChildView` 下隐藏“切孩子/接管”等家长操作；仅保留自头像。
  - `AccountModeSettings`：`isChildView` 隐藏/禁用模式切换。
  - `FamilyManagement`：`isChildView` 隐藏“创建家庭”入口；仅读为主（后端权限兜底）。

### 后端（模型/规则/接口）

- 不改模型
  - 不持久化 `appView`；`AccountMode` 仍为 `'PERSONAL' | 'PARENTAL' | 'DUAL'`。
  - `enableSelfJournaling` 语义不变（仅对 DUAL + self_mangement 的前端显隐生效）。

- 一次性迁移
  - 为缺少 `appRunMode` 的用户补 `{ accountMode:'PERSONAL', enableSelfJournaling:true }`。
  - 虽有 GET 懒初始化，仍建议批量迁移清理历史脏数据。

- 家庭/邀请约束（关键）
  - 发送邀请（被邀请为 child）：若被邀请账号已存在，且其 `accountMode !== 'PERSONAL'` → 拒绝（403/409）。
  - 接受邀请：当前用户 `accountMode !== 'PERSONAL'` → 拒绝（403）。
  - 创建家庭组：若当前用户已在任意家庭组以 `role='child'` 存在（active）→ 拒绝（403）。
  - 实施位置：`FamilyGroupApplicationService.createInvitation / acceptInvitation / create`。

- `/auth/take-over`
  - 保持不变：有 child dataGroup 需家长/管理员权限；无 id 回本人允许。

- `/user/account-mode`
  - 仅持久化 `accountMode/enableSelfJournaling`。
  - 可在响应中派生 `appView` 供初始化（PERSONAL→self_mangement，PARENTAL→parental_control，DUAL→默认 self_mangement）；不派生/不持久化 `self_mangement_child`（由前端基于家庭角色判定）。

- 索引/监控（建议）
  - 为家庭成员查询加索引：`members.user, members.dataGroup, members.role, isActive`，顶层 `isActive`、`dataGroup`。
  - 审计：统计“因 accountMode≠PERSONAL 被拒绝的邀请”；保持 `/auth/take-over` 审计。

- 测试
  - 迁移幂等；邀请/接受在非 PERSONAL 下被拒绝；child 角色用户不能创建家庭组；take-over 权限校验不变；GET `/user/account-mode` 不持久化 `appView`。


