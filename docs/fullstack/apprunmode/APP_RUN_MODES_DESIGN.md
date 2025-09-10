## 应用运行模式与能力范围管控设计（桌面版优先）

本设计文档定义应用的运行模式（个人/家长/双向），顶级模式切换方式，能力范围（功能可见/可用）的统一管控策略，以及前后端改造点与落地步骤。当前版本仅聚焦桌面端渲染与交互；移动端将于桌面端完成后再行适配并补充设计。

### 背景与目标
- 解决多账户（家长/孩子）下的“谁的数据被操作”的歧义，并避免用户误操作。
- 通过“顶级模式切换按钮”明确全局“激活目标”，从根上统一功能可用范围与页面显示，减少页面内分散切换导致的复杂性。
- 优先使用现有的整会话切换机制（/auth/take-over）与 Token 中的 dataGroup 来表达“当前目标账户”，避免每请求的代理头复杂度。

## 核心概念

### 应用运行模式（Account Mode）
- accountMode ∈ { 'PERSONAL', 'PARENTAL', 'DUAL' }
  - PERSONAL：仅个人视角
  - PARENTAL：仅家长控制视角
  - DUAL：双视角，允许在个人与家长控制之间切换

当 accountMode='DUAL' 时，增加一个“子目标状态/视图取向”用于记录前端当前应展示的视角：
- appView ∈ { 'self_mangement', 'parental_control', 'self_mangement_child' }
  - self_mangement：个人视角
  - parental_control：家长控制视角
  - self_mangement_child：儿童视角（仅前端视图，用于“孩子账号登录/在家庭中为 child 角色”时的 UI 呈现）

视图派生（前端只读布尔）：
- isChildView = (accountMode === 'PERSONAL' && appView === 'self_mangement_child')
- isParentalView = (accountMode === 'PARENTAL') 或 (accountMode === 'DUAL' 且 appView === 'parental_control')
- isPersonalView = (accountMode === 'PERSONAL' && appView === 'self_mangement')

### 会话切换与 Token.dataGroup
- 不再使用 activeTarget 概念。当前“操作主体”仅由 Token.dataGroup 决定，通过 /auth/take-over 进行切换。
- 关系说明：
  - PERSONAL：appView 可为 self_mangement 或 self_mangement_child；Token.dataGroup 始终指向本人；/auth/take-over（无 id）用于校正指向本人。
  - PARENTAL：appView 固定为 parental_control；/auth/take-over（child dataGroup）将 Token.dataGroup 指向对应孩子。
  - DUAL：appView='self_mangement' 时调用 /auth/take-over（无 id）；appView='parental_control' 时调用 /auth/take-over（child dataGroup）。

### 能力范围（Feature Gate）
根据 accountMode、appView 与 enableSelfJournaling 决定“哪些功能可见/可用”，用于菜单、路由守卫和页面内模块的显示控制。
注意：在功能显隐层面，将 `self_mangement_child` 与 `parental_control` 视为等价（同属“家长/孩子视角”）。

## 用户体验设计（桌面端）

### 目标切换（集成于 TopBar 右上角 UserDropdown 头像区）
- 交互整合到 UserDropdown 的头像区，不再使用独立的顶部切换按钮组件。
- accountMode 不同场景：
  - accountMode='PERSONAL'：仅显示当前用户头像；点击头像可执行自我接管（无 id）。
  - accountMode='PARENTAL'：仅显示所有已关联孩子的头像列表；点击任一孩子头像切换至该孩子（携带 dataGroup）。
  - accountMode='DUAL'：同时显示“自己头像 + 孩子头像列表”；
    - 点击自己头像：设置 cookies.appView='self_mangement' 并调用 /auth/take-over（无 id）。
    - 点击孩子头像：写入 cookies.selectedChildDataGroup=child.dataGroup，设置 cookies.appView='parental_control' 并调用 /auth/take-over（携带该 dataGroup）。
- 若 DUAL 且系统中无孩子（children.length=0）：仅显示自己头像；在合适位置引导前往“家庭管理/成员邀请”。
- 当前激活目标以视觉高亮呈现（如头像外环高亮）。
- 切换后触发统一的“数据组切换副作用”（清缓存、刷新、路由校正）。

### 能力范围矩阵（示例）
- accountMode='PARENTAL' 或（accountMode='DUAL' 且 appView='parental_control'）：
  - 显示：孩子相关功能（时间金币、时间皇冠、孩子银行账户、趋势洞察等）。
  - 隐藏：提醒（Reminders）、笔记（Notes）、家长个人日志。
- accountMode='PERSONAL' 且 appView='self_mangement_child'（孩子视角）：
  - 显示：与家长控制视角当前一致的孩子相关界面与模块（后续会逐步引入差异化控制，以区分家长控制与孩子视角）。
  - 隐藏：成人个人侧功能（例如模式切换入口等）；该用户不可切换到其他账户模式（PARENTAL/DUAL）。
  - 说明：此情形用于“账户为个人，且属于某家庭组并具有 child 角色”的用户；UI 为儿童体验，能力不包含任何家长操作。
- accountMode='PERSONAL' 且 appView='self_mangement'：
  - 显示：个人功能（提醒、笔记、个人仪表板、个人统计、个人日志等）。
  - 隐藏：孩子专属功能（金币、皇冠、孩子银行等）。
- accountMode='DUAL' 且 appView='self_mangement'：
  - 若 enableSelfJournaling=true：同 PERSONAL（self_mangement）。
  - 若 enableSelfJournaling=false：
    - 隐藏/禁用：个人日志与其所有派生能力（计划/Plan、活动统计/Statistics、实践/Practice 系列等所有依赖日志数据的功能）。
    - 仍可用：提醒（Reminders）与笔记（Notes）。

#### 孩子上下文下日志展示的强制性
- 当 accountMode='PARENTAL'，或 accountMode='DUAL' 且 appView='parental_control' 时，孩子日志必须展示，且不可被任何个人侧配置隐藏。

### 自我日志开关（仅在 accountMode=DUAL 且 appView=self_mangement 生效）
- 偏好项：enableSelfJournaling: boolean。
- 生效条件：仅当 accountMode='DUAL' 且 appView='self_mangement' 时生效；其他任何模式或视图下不生效、也不需设置。
- 当生效且值为 false 时：
  - 隐藏“活动日志/记录”等自我日志相关入口、页面和模块；前端不发起相关请求。
  - 同时隐藏/禁用基于个人日志的派生能力（计划、活动统计、实践系列等）。
- 用途与原因：
  - 满足部分家长在 DUAL 下并不希望记录个人日志、也不关注自我统计的简化诉求；
  - 保留“提醒/笔记”等轻量自我管理能力，避免为不需要日志的用户制造负担；
  - 对孩子侧的日志完全不受影响（见上节）。

- ### 菜单与路由守卫
- 菜单渲染前统一通过 isFeatureVisible(featureKey, { accountMode, appView, enableSelfJournaling }) 过滤。
- 路由进入时再校验一次；若不可访问则跳转到可访问的主页并 toast 提示原因（当前模式不可用该功能）。

## 前端设计

### 状态与存储
- 账户模式（useAccountModeStore，账户级）：
  - 字段：
    - accountMode: 'PERSONAL' | 'PARENTAL' | 'DUAL'
    - enableSelfJournaling: boolean（仅当 accountMode=DUAL 且 cookies.appView=self_mangement 时使用）
  - 读写接口：/user/account-mode（账户级，绑定 userId）。
  - appView：仅前端 cookies 持久化，不写入后端。
  - 最近选择的孩子：仅前端 cookies 存储 dataGroup，不写入后端；配合 `/auth/take-over` 使用。

- 偏好（usePreferenceStore，dataGroup 级）：
  - 字段：
    - uiTheme
    - pageState
  - 读写沿用 /user/preference，保持按 dataGroup 绑定。

- 会话（useSessionUser）：
  - dataGroup：来源于 Token；切换通过 /auth/take-over。
  - 不再使用 isParentalControlMode。所有视角相关判断统一由 `accountMode + appView` 推导（可通过前端 Hook `useViewContext()` 提供 `isChildView/isParentalView/isPersonalView`）。

### 头像切换（UserDropdown 内）
- 放置位置：TopBar 右上角 UserDropdown 的头像区（固定可见）。
- 交互规则：
  - accountMode='PERSONAL'：仅自己头像；点击可触发自我接管；无需写入 cookies.appView（默认为 self_mangement）。
  - accountMode='PARENTAL'：仅孩子头像列表；点击某个孩子即写入 cookies.selectedChildDataGroup 并触发 take-over；无需写入 cookies.appView（默认为 parental_control）。
  - accountMode='DUAL'：同时呈现自己与孩子头像，使用细竖线进行视觉分隔；
    - 点击自己：cookies.appView='self_mangement' → take-over（无 id）。
    - 点击孩子：cookies.selectedChildDataGroup=child.dataGroup 且 cookies.appView='parental_control' → take-over（携带 id）。

- 更新 cookies.appView 与（如需）cookies.selectedChildDataGroup 后，触发 take-over 调用并在成功后：
  - 统一触发“数据组切换副作用”：
    - 通知各 Store 若 currentDataGroup 变化则清缓存与重拉；
    - 关键页数据刷新；
    - 路由落位（例如回到首页）。

### 头像与切换入口的一致性（TopBar 行为）
- 顶部切换整合进 UserDropdown 的头像区：
  - accountMode=PERSONAL：仅显示当前用户头像。
  - accountMode=PARENTAL：仅显示孩子头像列表。
  - accountMode=DUAL：同时显示当前用户头像与孩子头像列表，中间以细竖线分隔；当前激活目标头像以高亮描边标识。
- 孩子列表既用于选择“当前孩子”，也可承担在 DUAL 下的视角切换职责（点击孩子即切入 parental_control，点击自己即切回 self_mangement）。

TopBar 下拉一致性：
- UserDropdown 顶部头像区承载切换：
  - PERSONAL：仅自己头像。
  - PARENTAL：仅孩子头像列表。
  - DUAL：自己 + 孩子头像并列显示，使用细竖线分隔；高亮当前激活目标。
- 切换孩子时更新 cookies.selectedChildDataGroup 并触发 take-over；点击自己触发自我接管。

### 偏好与账户模式设置页行为（PreferenceSettings + AccountModeSettings）
- 可配置：
  - 账户模式（账户级）：accountMode、appView、enableSelfJournaling（通过 /user/account-mode）
  - 偏好（dataGroup 级）：uiTheme、pageState（通过 /user/preference）
- 自动选择孩子：当启用了家长相关模式（PARENTAL 或 DUAL）且没有默认孩子时，若存在孩子则默认选中第一个孩子；若无孩子则引导至邀请孩子页面。
- DUAL 无孩子的特殊处理：当 accountMode='DUAL' 但系统内无孩子时，显示与 PARENTAL 相同的提示与入口，引导用户前往“家庭管理/成员邀请”页面创建或邀请孩子；在成功新增孩子后，自动将第一个孩子设为当前/默认选择并允许使用 Parental 视角。
- 当 accountMode='PARENTAL' 时隐藏“自我日志开关”。
- 当 isChildView（即 accountMode='PERSONAL' 且 appView='self_mangement_child'）时：隐藏/禁用账户模式切换，不提供 PARENTAL/DUAL 的选择入口。
- 当账户模式导致 activeTarget 变化时，应触发一次 take-over，使 UI 立刻反映最新目标（头像/菜单/路由）。
  - 注意避免循环触发：仅当“账户模式设置本身”变化时触发，不因 user.dataGroup 的变更（take-over 结果）而再次触发。

### 能力范围判定
- 工具：isFeatureVisible(featureKey, context) → boolean。
- 应用场景：
  - 菜单项渲染；
  - 入口按钮/卡片的显隐；
  - 路由守卫；
  - 页面内部模块（如在个人主页中隐藏“自我日志”模块，当 enableSelfJournaling=false）。

### 与既有代码的兼容与迁移
- 移除 user.isParentalControlMode 字段的依赖，统一以 `accountMode + appView` 推导视角布尔：
  - isChildView = (accountMode==='PERSONAL' && appView==='self_mangement_child')
  - isParentalView / isPersonalView 同上
- 数据组驱动缓存的场景（如提醒模块）保持不变；将“dataGroup 变更”作为清缓存的触发条件。
－ 建议为本地存储/持久化 Store bump 版本或更换 key，以清理历史中 isParentalControlMode 的残留。

### 缓存与数据刷新
- 各 Store 增加/保留 currentDataGroup 标记；当切换时：
  - 清缓存：如 remindersMine, remindersShared 等；
  - 重置分页/游标；
  - 重新加载当前页面必需数据。

## 后端设计

### 会话与切换
- 沿用整会话切换：POST /auth/take-over。
  - 携带 id（child dataGroup）→ 切到孩子；
  - 不携带 id → 切回本人。
- Token 中 dataGroup 即为“当前操作目标”，后续所有请求按该 dataGroup 授权与隔离。

### 账户模式接口
- /user/account-mode（账户级，绑定 userId）：
  - appRunMode: {
    - accountMode: 'PERSONAL' | 'PARENTAL' | 'DUAL'
    - appView: 'self_mangement' | 'parental_control'（仅在响应中派生返回，用于前端初始化；不持久化）
    - enableSelfJournaling: boolean（仅在 DUAL:personal 生效）
  }
  - 后端将 appRunMode 存储在 `User.appRunMode` 字段；不再使用独立集合。
  - POST 仅对传入字段做浅合并更新；忽略并不持久化 `appView`。
  - 不影响业务数据，仅用于前端初始化与 UI 控制。

说明：`self_mangement_child` 为前端视图值，不在后端持久化；前端在检测到“用户属于某家庭且其角色为 child”时，将 AppView 规范化为 `self_mangement_child` 来呈现儿童视角（功能显隐与 `parental_control` 等价，但不暴露家长操作）。

### 授权与审计
- take-over 必须校验当前用户是否对目标 dataGroup 具备家长控制权限。
- 审计日志记录：源用户、目标 dataGroup、时间与操作上下文（可选）。

## 实施

### 分步落地（建议）
1) 后端：新增 /user/account-mode（账户级）实现 accountMode、appView、enableSelfJournaling；/user/preference 仅保留 UI 主题与页面状态；确认 /auth/take-over 已充分审计并鉴权。
2) 前端：实现 TargetSwitcher（仅 DUAL 时可 Personal/Parental 切换），串通 take-over，完成“数据组切换副作用”。
3) 前端：接入菜单与路由守卫的能力范围过滤；给出清晰的禁用提示与回退路由。
4) 前端：实现 enableSelfJournaling 偏好与相关模块的显隐控制（仅 DUAL:personal 生效）。
5) 完成桌面端功能后，撰写移动端适配方案并迭代实现。

### 风险与缓解
- 频繁切换导致数据抖动：使用统一的副作用管线与渐进加载策略，避免大范围闪烁。
- 缓存串数据：所有与 dataGroup 绑定的 Store 均以 currentDataGroup 为键，切换即清。
- 功能隐藏引起困惑：ModeBadge 永远可见，首次切换提供解释性提示；路由守卫以文案说明。

## 验收标准（桌面端）
- 顶部 UserDropdown 头像区可用：
  - PERSONAL：显示并可点击自己头像（自我接管）。
  - PARENTAL：显示孩子头像列表并可点击切换孩子（接管到对应 dataGroup）。
  - DUAL：同时显示自己与孩子头像，二者之间有明显分隔；点击自己/孩子可在 Self/Parental 之间切换并正确接管。
- 账户模式通过 /user/account-mode 读写；偏好通过 /user/preference 读写，二者边界清晰。
- 切换后 Token 的 dataGroup 与 UI 的激活目标保持一致；数据缓存按 dataGroup 切换正确清理与刷新。
- 菜单/路由/页面模块的显隐符合能力范围矩阵；不可访问的功能被隐藏或被安全拦截。
- 仅当 accountMode=DUAL 且 appView=self_mangement 时：
  - enableSelfJournaling=false → activeTarget=self 下个人日志模块不可见且不发起请求。

## 后续（移动端适配占位）
- 移动端顶部空间受限，ModeSwitcher 交互需适配：
  - 合并为“模式状态条 + 抽屉切换面板”；
  - 保留 Badge 与首次提示；
  - 与底部导航的能力范围联动，确保不可访问的 Tab 不出现。
- 等桌面端实现稳定后，补充移动端 UI/交互草图与实现计划。


