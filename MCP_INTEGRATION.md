# MCP 集成说明（MCP_INTEGRATION.md）

本文说明「麦当劳省钱决策官」实际使用的麦当劳 MCP Server、Tool、调用流程与业务价值。
文中所有工具名、参数约束、错误码和返回数据，均为 **2026-10-17 对 `https://mcp.mcd.cn` 的真实调用实测结果**，不是照抄接口文档。

---

## 一、使用的 MCP Server

| 项 | 值 |
|:--|:--|
| 端点 | `https://mcp.mcd.cn` |
| 协议 | Streamable HTTP（MCP 标准，无需本地起服务） |
| 鉴权 | `Authorization: Bearer <MCP Token>`（在 open.mcd.cn 申请） |
| Server 标识 | `mcd-mcp` 1.0.0（`initialize` 返回的 `serverInfo`） |
| 协商协议版本 | `2024-11-05` |
| **可用工具数** | **35 个**（`tools/list` 实测，2026-10-17） |
| 限流 | 600 次/分钟 |

> ⚠️ **实测纠偏**：网上流传的工具清单多为「33 个 / 24 个」，且有两个工具名与真实不一致。
> 实测 35 个中，`query-party-store-date`、`query-party-store-session` 常被误写为 `query-partystore-date` / `query-partystore-session`（会直接调用失败）；
> 另有 `query-promotions`、`query-survey-coupon` 两个工具未出现在部分旧清单中。

---

## 二、实际使用的 Tool（按角色分层）

### 2.1 核心链路（每次决策必用）

| Tool | 在本项目中的角色 | 关键入参 |
|:--|:--|:--|
| `now-time-info` | 取服务器真实时间，防止"券过期/今天明天"判断错 | 无 |
| `query-nearby-stores` | 定位门店，拿 `storeCode` | `beType`、`searchType`、`city`、`keyword` |
| `query-meals` | **标榜层数据源**：在售菜单与标价 | `storeCode`、`orderType`、`beType` |
| `query-meal-detail` | 套餐组成拆解（换品/加减项后重新计价要用） | `storeCode`、`orderType`、`beType`、`code` |
| `query-store-coupons` | **当前门店当前订单**真正可用的券 | `storeCode`、`orderType`、`beType` |
| `available-coupons` | 可领取的麦麦省券（领取前） | 无 |
| `auto-bind-coupons` | 一键领取全部可领券，把"可领"变"已领" | 无 |
| `query-my-coupons` | 已领券包（领取后） | `page`、`pageSize` |
| `calculate-price` | **事实层核心**：含券后真实实付 | `storeCode`、`orderType`、`beType`、`items[]` |
| `campaign-calendar` | 活动日历，做"现在买 / 等明天"预判 | 可选 `specifiedDate` |

### 2.2 辅助能力（按用户诉求按需调用）

| Tool | 什么时候用 |
|:--|:--|
| `query-my-account` | 用户问"我有多少积分""积分快过期了吗" |
| `mall-points-products` / `mall-product-detail` | 用户想用积分兑换 |
| `query-lottery-info` / `query-my-prizes` | 用户问抽奖 |
| `list-nutrition-foods` | 用户问热量/营养 |
| `order-list` / `query-order` | 用户问历史订单、订单进度 |
| `delivery-query-addresses` / `delivery-query-stores` | 用户明确要外送 |

### 2.3 资金动作（默认禁用，必须用户二次确认）

| Tool | 风险 |
|:--|:--|
| `create-order` | 真实下单，返回支付链接 |
| `mall-create-order` | 积分兑换下单，扣积分 |
| `party-order-create` | 派对/品鉴会下单，真实支付 |
| `draw-lottery` | 抽奖，消耗积分 |

本 Skill 的默认行为是 **只给决策、不碰钱**。上述工具不出现在自动流程里，只有用户明确说"就这个，下单"并确认完整明细后才调用。

---

## 三、调用流程

### 3.1 标准决策链路（6 步）

```
用户：这单怎么点最省？
  │
  ├─① now-time-info ──────────── 拿服务器真实时间（防"今天/明天"错判）
  │
  ├─② query-nearby-stores ─────── city + keyword 同时传 → storeCode
  │      └─ 检查 businessStatus / businessStartTime（未营业直接告知，不硬算）
  │
  ├─③ query-meals ─────────────── 取标榜层：菜品 code / currentPrice / originalPrice / discountType
  │      └─ query-meal-detail ─── 套餐类目拆解（换品重算时）
  │
  ├─④ auto-bind-coupons ───────── 一键全领（把"可领"变"已领"）
  │      ├─ available-coupons ─── 可领券池
  │      ├─ query-my-coupons ──── 已领券包
  │      └─ query-store-coupons ─ 当前门店+渠道真正可用的券（最终以它为准）
  │
  ├─⑤ calculate-price ×N ──────── 对 N 套券组合逐个真实计价，得到实付 p₁…pₙ 与基线 p₀
  │
  └─⑥ campaign-calendar ───────── 查 48h 内是否有更优活动 → 输出"现在买 / 等 X 小时"
```

### 3.2 真实调用示例（2026-10-17 实测，可复现）

**① 定位门店**

```json
// 入参
{"beType": 1, "searchType": 2, "city": "石家庄", "keyword": "麦当劳"}
```

```json
// 真实返回（节选）
{"success": true, "data": [{
  "storeCode": "1950878",
  "storeName": "麦当劳石家庄瑞天华安购物中心餐厅",
  "businessStatus": true,
  "businessStartTime": "07:00",
  "businessEndTime": "23:00"
}]}
```

**② 取标榜层（菜单）**

```json
// 入参
{"storeCode": "1950878", "orderType": 1, "beType": 1}
```

```json
// 真实返回（节选，当日早餐时段共 10 个分类 / 86 个餐品）
{"data": {"meals": {
  "9900003457": {"name": "原味板烧鸡腿麦满分套餐",
                 "currentPrice": "14.1",
                 "originalPrice": "23.5",
                 "discountType": "随单购早餐卡优惠"},
  "9900008751": {"name": "可乐", "currentPrice": "9.5", "originalPrice": "9.5"}
}}}
```

**③ 取事实层（真实计价）**

```json
// 入参
{"storeCode": "1950878", "orderType": 1, "beType": 1,
 "items": [{"productCode": "9900003457", "quantity": 1},
           {"productCode": "9900008751", "quantity": 1}]}
```

```json
// 真实返回
{"success": true, "data": {
  "productOriginalPrice": 3300, "productPrice": 3300,
  "discount": 0, "price": 3300,
  "productList": [
    {"productCode": "9900003457", "productName": "原味板烧鸡腿麦满分套餐",
     "originalSubtotal": 2350, "subtotal": 2350},
    {"productCode": "9900008751", "productName": "可乐",
     "originalSubtotal": 950, "subtotal": 950}]}}
```

> 金额单位为 **分**（`3300` = ¥33.00），输出给用户时必须除以 100。

---

## 四、实测发现的行为约束（本项目最核心的技术沉淀）

这些约束在官方接口文档里没有写清，是真实调用撞出来的。本 Skill 的全部流程都按这些约束设计。

| # | 现象 | 真实错误 / 数据 | 本 Skill 的处理 |
|:--|:--|:--|:--|
| 1 | **`query-nearby-stores` 的 `city` 与 `keyword` 必须同时传** | 只传 `city` 或只传 `keyword` → `{"code": 600058, "message": "城市名或者关键词不能为空"}` | 两者同时传；缺失时主动追问用户，不空转 |
| 2 | `searchType=1`（收藏餐厅）在无收藏时必失败 | `{"code": 600050, "message": "收藏餐厅列表为空，请您提供城市+关键词进行搜索"}` | 直接用 `searchType=2` + 城市/关键词搜索 |
| 3 | **门店未营业时菜单与计价全部不可用** | 09:36 调 10:00 开门的门店 → `{"code": 600057, "message": "门店可能已关闭或不在营业时间"}` | 先用返回的 `businessStatus` / 营业时段判断，未营业直接告知用户，不编造价格 |
| 4 | **菜单 `currentPrice` ≠ 能拿到的价** | 同一套餐 `currentPrice` 显示 14.1，但 `discountType` 是"随单购早餐卡优惠"；不随单购卡时 `calculate-price` 返回 **23.50**，差 **9.40 元** | 菜单价只用于展示"标价"，结论一律以 `calculate-price` 的 `price` 为准 |
| 5 | 菜单**随时间段变化** | 09:36 取到的是早餐菜单（10 分类 / 86 项），与正餐菜单不同 | 每次决策实时拉取，不缓存跨时段菜单 |
| 6 | `beCode` 的传参规则不一致 | 到店自取（`beType=1`）**不传** `beCode`；得来速（`beType=5`）/ 外送（`orderType=2`）**必传** | 按 `beType` 决定是否透传，不透传错误值 |
| 7 | **券有三个层次，不能混用** | `available-coupons` 有券（9.9 元冰美式等），`query-my-coupons` 返回"暂无可用优惠券"，`query-store-coupons` 返回 `[]` | 先 `auto-bind-coupons` 领券，再用 `query-store-coupons` 判定"这单能不能用"，最后交 `calculate-price` 验证 |
| 8 | 返回体前面常带一段"字段说明"文本 | 解析时先剥离说明段再取 JSON | 解析器先定位 `{"success"` 起点的 JSON 段 |

> 第 4 条是本项目存在的理由：**用户以为的价，和结账的价，可以差 9 块多。** 这个差不是"优惠多少"，而是"菜单展示口径与结算口径不一致"。没有 `calculate-price`，任何省钱结论都是猜的。

---

## 五、业务价值

### 5.1 解决的真问题

| 用户痛点 | 常见做法 | 本 Skill 的做法 |
|:--|:--|:--|
| 菜单价不等于实付价 | 照菜单报数 | 每次都用 `calculate-price` 算实付 |
| 券太多不会组合 | 让用户自己看券包 | 枚举候选组合 → 逐个真实计价 → 取最小 |
| 不知道要不要等明天 | 只说"今天有什么优惠" | `campaign-calendar` 比 48h 内活动，算"等 vs 买"的差额 |
| 领了券不等于能用 | 拿券包直接报优惠 | `query-store-coupons` 校验当前门店渠道可用性 |

### 5.2 真实算例（2026-10-17 09:36 实测）

用户想点：原味板烧鸡腿麦满分套餐 ×1 + 可乐 ×1

| 口径 | 金额 | 说明 |
|:--|--:|:--|
| 菜单展示的 `currentPrice` 合计 | ¥23.60 | 14.1 + 9.5，但 14.1 依赖"随单购早餐卡" |
| `calculate-price` 无券实付 | **¥33.00** | 23.50 + 9.50，这才是真实结账口径 |
| 差额 | **¥9.40** | 只看菜单会少报 9.4 元 |

同时 `campaign-calendar` 显示 10 月 8 日–21 日有「9.9 元早餐两件套」活动——是否该改点两件套、是否值得等，属于本 Skill 的预判输出。

### 5.3 与"点餐助手"的区别

点餐助手的目标是**把单下出去**；本 Skill 的目标是**在不下单的前提下，把"这单到底多少钱、哪套最省、要不要等"算清楚**。
下单接口默认不碰，是本项目的产品立场，也是安全边界。

---

## 六、安全与隐私

| 项 | 做法 |
|:--|:--|
| Token | 只存在 WorkBuddy 的 `mcp.json`；Skill 不读取、不写入、不回显、不落日志 |
| 位置与地址 | `delivery-query-addresses` 等仅在用户明确要求外送时调用，不持久化、不写入任何文件 |
| 资金动作 | `create-order` / `mall-create-order` / `party-order-create` / `draw-lottery` 默认禁用，需用户确认完整明细后二次确认 |
| 不编造 | 取不到实时数据时明确说"无法实算"，绝不拿标价冒充实付 |
| 仓库卫生 | 本仓库不含任何真实 Token、手机号、地址、真实姓名；配置示例只用环境变量占位 |

---

## 七、复现验证

1. 在 `open.mcd.cn` 申请 MCP Token；
2. 按 `mcp-config.example.json` 配置连接器，把 `${MCD_MCP_TOKEN}` 换成自己的 Token；
3. 依次调用：`now-time-info` → `query-nearby-stores`（city+keyword）→ `query-meals` → `calculate-price`；
4. 对比 `query-meals` 的 `currentPrice` 与 `calculate-price` 的 `price`，即可复现"标价 ≠ 实付"。

---

*实测环境：WorkBuddy（Windows），2026-10-17 09:35–09:37（GMT+8）。*
