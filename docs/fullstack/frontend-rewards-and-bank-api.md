## 前端集成指南：奖励统计与儿童银行账户 API

### 概览
- 目标：在前端展示历史奖励统计（星/皇冠/金币/平均星）与“模拟银行账户”（余额、提现、汇率）。
- 基本单位：日期使用 `YYYY-MM-DD`，时区默认 `Europe/Berlin`（可通过 `timezone` 覆盖）。
- 认证：所有接口需携带登录态（`Authorization: Bearer <token>` 或 Cookie）。

---

### 日期与时区约定
- `start`、`end`、`date` 参数格式：`YYYY-MM-DD`。
- 如果未传 `timezone`，后端默认使用 `Europe/Berlin`。
- 统计口径：排除“睡觉”类别；只统计 `rating.overall > 0` 的活动。

---

### API 列表

#### 1) 获取日期区间的每日奖励统计
- 方法/路径：GET `/api/rewards/daily`
- Query：
  - `start` string 必填（YYYY-MM-DD）
  - `end` string 必填（YYYY-MM-DD）
  - `timezone` string 可选（默认 Europe/Berlin）
- 响应：
```json
{
  "success": true,
  "data": [
    {
      "userDate": "2025-08-20",
      "totalStars": 24,
      "totalCrowns": 3,
      "totalCoins": 120,
      "activitiesWithRating": 6,
      "averageRating": 4.0
    }
  ]
}
```

#### 2) 触发单日重算（管理用途）
- 方法/路径：POST `/api/rewards/rebuild`
- Body：
```json
{ "date": "2025-08-20", "timezone": "Europe/Berlin" }
```
- 说明：前端正常不需要调用；活动变更后系统会自动异步重算受影响日期。

#### 3) 获取当前账户信息
- 方法/路径：GET `/api/bank-accounts/me`
- 响应：
```json
{
  "success": true,
  "data": {
    "coinBalance": 350,
    "rate": { "coinsPerMoneyUnit": 100, "currency": "CNY" },
    "availableMoney": 3.5
  }
}
```

#### 4) 查询账户流水（倒序）
- 方法/路径：GET `/api/bank-accounts/me/transactions`
- Query：
  - `limit` number 可选（默认 50）
  - `skip` number 可选（默认 0）
- 响应（示例）：
```json
{
  "success": true,
  "data": [
    {
      "_id": "trx_1",
      "type": "WITHDRAW", // EARN | ADJUSTMENT | WITHDRAW | DEPOSIT
      "coins": -200,
      "money": 2,
      "rateSnapshot": { "coinsPerMoneyUnit": 100, "currency": "CNY" },
      "withdrawMeta": { "moneyFromDepositsPortion": 1.5 },
      "depositMeta": { "sourceType": "RELATIVE", "sourceName": "Grandpa", "note": "gift" },
      "adjustmentMeta": {
        "entries": [
          {
            "mainCategoryMeaningId": "study",
            "helpingCategoryMeaningId": "help",
            "recipientAliases": ["self"],
            "activityTagNames": ["Reading"],
            "subActivityNames": ["Focus"]
          }
        ]
      },
      "createdAt": "2025-08-21T10:20:00.000Z"
    }
  ]
}
```

- 字段说明（Meta 变更）：
  - `depositMeta`：仅对 `DEPOSIT` 类型返回来源信息（`sourceType` | `sourceName`）与可选 `note`。
  - `withdrawMeta.moneyFromDepositsPortion`：当本次提现使用了“存款池”时标记该部分金额；若采用“双流水”拆分，则 coins=0 的 `WITHDRAW` 代表存款部分。
  - `withdrawMeta.source`: `COINS | DEPOSITS`，表明当前记录的扣款来源。
  - `adjustmentMeta.entries`：数组，每个元素代表一次奖励差异的归因条目：
    - `mainCategoryMeaningId`：主类目 MeaningId（可选）
    - `helpingCategoryMeaningId`：辅助类目 MeaningId（可选）
    - `recipientAliases`：受众别名数组（可选）
    - `activityTagNames`：活动标签名数组（可选）
    - `subActivityNames`：子活动名数组（可选）

#### 5) 发起提现
- 方法/路径：POST `/api/bank-accounts/me/withdraw`
- Body（二选一）：
```json
{ "money": 10, "priority": "DEPOSITS" }
```
或
```json
{ "coins": 1000 }
```
- 响应：同“获取当前账户信息”。
- 说明：当前后端允许余额变为负数（若需限制，后续会加风控开关）。

- 新增参数：
  - `priority?: 'COINS'|'DEPOSITS'`（默认 `COINS`）。当传 `money` 时生效：
    - `DEPOSITS`：优先使用存款池（`moneyFromDeposits`），不足部分再从金币池（`coinBalance/rate`）换算扣减；
    - `COINS`：优先使用金币池，剩余再走存款池。
  - 超出可用总额将返回 400/422 错误。

#### 6) 获取/设置兑换率
- 方法/路径：
  - GET `/api/bank-accounts/me/rate`
  - PUT `/api/bank-accounts/me/rate`
- PUT Body：
```json
{ "coinsPerMoneyUnit": 100, "currency": "CNY" }
```
- 说明：建议仅在家长端暴露设置入口；历史提现保留 `rateSnapshot`，不回溯。

#### 7) 手动存入（DEPOSIT）
- 方法/路径：POST `/api/bank-accounts/me/deposits`（别名：`/api/child-bank/deposits`）
- Body：
```json
{ "money": 10, "sourceType": "RELATIVE", "sourceName": "Grandpa", "note": "gift", "idempotencyKey": "client-uuid-1" }
```
- 说明：
  - 存入只记录真实货币 `money`，不会改变金币余额 `coinBalance`。
  - 支持幂等：同一 `idempotencyKey` 不会重复入账。
  - 回执与“获取当前账户信息”相同。

#### 8) 获取余额与来源拆分
- 方法/路径：GET `/api/bank-accounts/me/balance-split`（别名：`/api/child-bank/balance`）
- 响应（示例）：
```json
{
  "success": true,
  "data": {
    "coinBalance": 1350,
    "rate": { "coinsPerMoneyUnit": 100, "currency": "CNY" },
    "availableMoney": 13.5,
    "coinsFromRewards": 350,
    "moneyFromRewardsEstimate": 13.5,
    "moneyFromDeposits": 10,
    "coinsFromRewardsYear": 1200
  }
}
```

- 口径更新：
  - `moneyFromDeposits = Σ(DEPOSIT money) − Σ(WITHDRAW.withdrawMeta.moneyFromDepositsPortion)`；
  - `availableMoney = coinBalance/rate + moneyFromDeposits`；
  - `moneyFromRewardsEstimate = coinBalance/rate`。
  - `coinsFromRewardsYear`：近 365 天内 `EARN/ADJUSTMENT` 的 coins 累加（用于前端概览展示）。

---

### 前端集成建议
- **历史趋势**：调用 GET `/api/rewards/daily` 获取区间序列，绘制折线/柱状图；空缺日期可用 0 填充。
- **账户余额与可兑换金额**：直接渲染 GET `/api/bank-accounts/me` 的 `coinBalance` 与 `availableMoney`；金额格式化两位小数。
- **提现流程**：
  - 家长端：输入金额或硬币数 → 调用 POST `/withdraw` → 刷新账户与流水。
  - UI 上提示当前兑换率：`1 货币单位 = coinsPerMoneyUnit 个金币`。
- **兑换率管理**：仅对家长开放 PUT `/rate`；修改后仅影响后续提现。
- **分页与加载**：流水默认 `limit=50`，按需做“加载更多”。
 - **流水 Meta 展示（entries 结构）**：
   - `DEPOSIT`：展示 `depositMeta.sourceName`（无则回退到 `sourceType`）与 `note`。
   - `ADJUSTMENT`：对 `adjustmentMeta.entries` 逐条展示或汇总：
     - 主类目：汇总每个条目的 `mainCategoryMeaningId`（可做 i18n 映射）
     - 辅助类目：`helpingCategoryMeaningId`
     - 受众：合并所有条目的 `recipientAliases`
     - 标签：合并所有条目的 `activityTagNames`
     - 子活动：合并所有条目的 `subActivityNames`
   - 所有字段均为可选，注意空值/空数组容错。

---

### 边界与错误处理
- **可编辑窗口（系统级）**：
  - 超出窗口的活动在后端将被拒绝修改；前端可在编辑界面提示“不可修改较早记录”。
  - 与本接口的读取无关，历史统计可读。
- **提现金额校验**：
  - 当前允许负余额，前端可根据产品策略在 UI 层做“不可透支”限制。
- **常见错误**：
  - 400/422：参数格式错误（日期格式、缺失参数）。
  - 401：未登录/Token 失效。
  - 403：无权限（非家长尝试修改汇率/提现）。
  - 500：服务端错误（重试或联系支持）。

---

### 前端示例（TypeScript + fetch）
```ts
// 1) 拉取日期区间统计
export async function fetchRewardsDaily(start: string, end: string, timezone?: string) {
  const url = new URL('/api/rewards/daily', location.origin);
  url.searchParams.set('start', start);
  url.searchParams.set('end', end);
  if (timezone) url.searchParams.set('timezone', timezone);
  const res = await fetch(url.toString(), { credentials: 'include' });
  if (!res.ok) throw new Error('Failed to load');
  const json = await res.json();
  return json.data as Array<{ userDate: string; totalStars: number; totalCrowns: number; totalCoins: number; activitiesWithRating: number; averageRating: number; }>;
}

// 2) 提现（按金额）
export async function withdrawByMoney(amount: number, priority: 'COINS'|'DEPOSITS' = 'DEPOSITS') {
  const res = await fetch('/api/bank-accounts/me/withdraw', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    credentials: 'include',
    body: JSON.stringify({ money: amount, priority })
  });
  if (!res.ok) throw new Error('Withdraw failed');
  return (await res.json()).data;
}
```

---

### 变更影响说明
- 奖励统计会在活动变更时自动重算“受影响日期”，并将差额计入账户流水（ADJUSTMENT）；前端无需手动刷新缓存，只需在关键操作后刷新对应接口即可。


