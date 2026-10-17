# 麦当劳 MCP 工具参考（35 工具 · 实测版）

> 数据来源：2026-10-17 对 `https://mcp.mcd.cn` 真实调用 `tools/list` 的结果（Server `mcd-mcp` 1.0.0）。
> ⚠️ 网上流传的「33 个 / 24 个」清单为旧版或有误；两个派对工具的正确名是 `query-party-store-date`、`query-party-store-session`（旧写法 `query-partystore-*` 会调用失败）。
> 端点：Streamable HTTP + Bearer Token，限流 600 次/分钟。金额字段单位为**分**。

## A. 门店 / 地址（4）

| 工具 | 用途 | 关键参数 |
|:--|:--|:--|
| `query-nearby-stores` | 查可到店取餐门店 | `beType`(1/5)、`searchType`(1/2)、`city`、`keyword` |
| `delivery-query-stores` | 按地址查可配送门店 | `addressId`、`beType` |
| `delivery-query-addresses` | 查我的配送地址（**隐私敏感**） | 无 |
| `delivery-create-address` | 新建配送地址（**隐私敏感**） | `city`、`contactName`、`address`、`phone` 等 |

> 🔴 **实测坑**：`query-nearby-stores` 的 `city` 与 `keyword` **必须同时传**，只传一个返回 `600058 城市名或者关键词不能为空`；`searchType=1`（收藏餐厅）在无收藏时返回 `600050`。

## B. 菜单 / 营养（3）

| 工具 | 用途 | 关键参数 |
|:--|:--|:--|
| `query-meals` | 在售菜单 + 标价（**标榜层数据源**） | `storeCode`、`orderType`、`beType` |
| `query-meal-detail` | 套餐组成拆解 | `storeCode`、`orderType`、`beType`、`code` |
| `list-nutrition-foods` | 营养成分（能量/蛋白/脂肪/钠/钙） | 无 |

> 🔴 **实测坑**：菜单**随时间段变化**（早餐时段与正餐不同），不要缓存跨时段菜单；门店未营业时返回 `600057 门店可能已关闭或不在营业时间`。

## C. 优惠券（4）

| 工具 | 用途 | 关键参数 |
|:--|:--|:--|
| `available-coupons` | 可领取的麦麦省券（**领取前**） | 无 |
| `auto-bind-coupons` | 一键领取全部可领券 | 无 |
| `query-my-coupons` | 卡包里的券（**领取后**，不校验门店可用性） | `page`、`pageSize` |
| `query-store-coupons` | **当前门店+渠道**真正可用的券 | `storeCode`、`orderType`、`beType` |

> 🔴 **实测坑**：三者含义不同，不可互相替代。实测同一时刻：`available-coupons` 有券、`query-my-coupons` 返回"暂无可用优惠券"、`query-store-coupons` 返回 `[]`。

## D. 计价 / 下单 / 取消（3）

| 工具 | 用途 | 关键参数 |
|:--|:--|:--|
| `calculate-price` | 算价，含券后**真实实付**（**事实层核心**） | `storeCode`、`orderType`、`beType`、`items[]` |
| `create-order` | 创建订单，返回支付链接（⚠️ 真实出网+资金） | `storeCode`、`orderType`、`beType`、`items` 等 |
| `cancel-order` | 取消订单 | `orderId`、`cancelReasonCode` |

> `items[]` 结构：`{productCode, quantity, couponId?, couponCode?, roundList?}`。

## E. 订单查询（2）

| 工具 | 用途 | 关键参数 |
|:--|:--|:--|
| `query-order` | 订单详情 / 配送进度 | `orderId` |
| `order-list` | 历史订单（不含商城订单） | 无 |

## F. 积分账户（1）

| 工具 | 用途 | 关键参数 |
|:--|:--|:--|
| `query-my-account` | 可用/累计/冻结/即将过期积分 | 无 |

## G. 麦麦商城（5）

| 工具 | 用途 | 关键参数 |
|:--|:--|:--|
| `mall-points-products` | 积分可兑换商品列表 | `catRuleIds`（可选） |
| `mall-product-detail` | 商品详情与 SKU | `spuId` |
| `mall-create-order` | 商城兑换下单（⚠️ 扣积分） | `skuId`、`spuCategory`、`count` 等 |
| `mall-order-list` | 商城订单列表 | `lastId`、`size` |
| `mall-order-detail` | 商城订单详情 | `orderId` |

## H. 活动日历（1）

| 工具 | 用途 | 关键参数 |
|:--|:--|:--|
| `campaign-calendar` | 当月营销活动（进行中/往期/未来）—— **活动预判数据源** | `specifiedDate`（可选，yyyy-MM-dd） |

## I. 主题活动 / 派对（5）

| 工具 | 用途 | 关键参数 |
|:--|:--|:--|
| `query-party-city` | 可办活动的城市列表 | `spuId` |
| `query-party-store` | 指定城市的派对门店 | `code`、`spuId`、经纬度 |
| `query-party-store-date` | 门店可预约日期 | `spuId`、`storeCode` |
| `query-party-store-session` | 门店某日可约场次 | `spuId`、`storeCode`、`dateStr` |
| `party-order-create` | 派对/品鉴会下单（⚠️ 资金） | `partyType` 等 |

## J. 积分抽奖（3）

| 工具 | 用途 | 关键参数 |
|:--|:--|:--|
| `query-lottery-info` | 抽奖活动信息与奖品列表 | 无 |
| `draw-lottery` | 执行抽奖（⚠️ 消耗积分，须先展示消耗并确认） | 无 |
| `query-my-prizes` | 我的奖品 | `pageNum`、`pageSize` |

## K. 团餐（2）

| 工具 | 用途 | 关键参数 |
|:--|:--|:--|
| `query-meal-assistance` | 团餐助餐服务（仅 beType=6） | `storeCode` 等 |
| `query-promotions` | 团餐可用满减/满折规则（仅 beType=6） | `storeCode`、`orderType`、`beType` |

## L. 辅助（2）

| 工具 | 用途 | 关键参数 |
|:--|:--|:--|
| `now-time-info` | 服务器当前时间（**决策前必调**，防券过期误判） | 无 |
| `query-survey-coupon` | 按订单查满意度答卷与关联奖券 | `orderId` |

---

## 本 Skill 核心调用组合

| 步骤 | 工具 | 说明 |
|:--|:--|:--|
| ① 取时间 | `now-time-info` | 判断券是否过期、活动是否已结束 |
| ② 定位门店 | `query-nearby-stores` | `city` + `keyword` 同传；检查 `businessStatus` |
| ③ 取标榜层 | `query-meals` →（需要时）`query-meal-detail` | 拿 `currentPrice` / `originalPrice` / `discountType` |
| ④ 取券池 | `auto-bind-coupons` → `available-coupons` / `query-my-coupons` / `query-store-coupons` | 先领，再校验门店可用 |
| ⑤ 取事实层 | `calculate-price`（多组合各跑一次） | 以返回的 `price` 为准，不看菜单价 |
| ⑥ 活动预判 | `campaign-calendar` | 未来 48h 是否有更优活动 |
| ⑦ 输出 | （本地计算） | 最优组合 + 省多少 + 等不等 |
| ⑧ 用户确认后下单 | `create-order` | ⚠️ 必须二次确认 |
