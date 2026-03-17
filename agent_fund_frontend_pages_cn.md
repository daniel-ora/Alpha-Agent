# Agent 基金平台前端页面设计

MVP 页面结构与关键交互

## 1. 页面地图

| 模块 | 说明 |
|------|------|
| 基金市场页 | 用于浏览、筛选和排序基金列表。 |
| 基金详情页 | 核心决策页面，包含业绩、持仓、历史战绩、右侧申购 / 赎回操作栏。 |
| 我的投资页 | 展示用户持有份额、估值、待处理申购 / 赎回与历史记录。 |
| 管理后台总览 | 内部运营使用，查看 AUM、NAV、净流入净流出、待处理 deal cycle。 |
| Fund 配置页 | 内部配置 Safe、基金资料。 |
| Onboarding 页 | 内部执行 Safe 部署、Token Approve、CLOB 注册（全程免 gas）。 |
| Deal Cycle 处理页 | 内部执行 freeze、preview、confirm、settle。 |
| 交易监控页 | 内部查看 Agent 交易活动、审计日志。 |

## 2. 基金市场页

- 以基金卡片列表为核心，不做社交 feed。
- 每张卡片展示：基金名、manager、策略标签、当前 NAV、近 7D / 30D 收益、最大回撤、AUM、是否开放申购。
- 支持筛选：开放状态、收益排序、回撤排序、AUM 排序、策略类型。
- 卡片 CTA：查看详情、申购。

## 3. 基金详情页

- 这是最核心页面，左侧是信息内容，右侧是固定操作栏。
- Hero 区显示：基金名、一句话策略、manager、inception date、当前 NAV、近 7D / 30D / inception 收益、AUM。
- 中部展示：NAV 曲线、回撤曲线、持仓表、最近交易时间线。
- 展示基金正式业绩数据。

## 4. 右侧操作栏

- 显示当前申购状态、下次 Deal Window、最近一次 NAV、最低申购金额、用户当前持有份额。
- 提供两个主按钮：申购、赎回。
- 底部展示规则摘要：批处理成交、统一 dealing NAV、TWAP midpoint 估值、USDC 赎回结算。
- 底部展示"安全与权限结构"：资金托管于 Gnosis Safe、Agent 通过 Backend Relay 交易（不持有私钥）、withdraw 仅属于平台 Owner Key。

## 5. 申购弹窗交互

- 点击"申购"后打开弹窗，不跳转页面。
- 输入 USDC 金额，实时显示预估 shares、最近一次 NAV、下一次 deal window。
- 明确提示：本次不会立即成交，最终按本周期统一 dealing NAV 配 shares。
- 确认后进入"已提交到本周申购队列"的成功态，并提示可去"我的投资"查看状态。

## 6. 赎回弹窗交互

- 点击"赎回"后打开弹窗。
- 展示当前持有 shares、可赎回份额、预估可得 USDC。
- 明确提示：最终赎回金额按本周期统一 NAV 计算，并以 USDC 结算。
- 提交后展示"已提交到本周赎回队列"，后续会在 deal cycle 完成后支付 USDC。

## 7. 我的投资页

- 展示用户持有的基金列表、当前份额、当前估值、累计申购、累计赎回。
- 展示待处理申购与赎回请求，含状态：pending / queued / settled / cancelled 等。
- 支持点击某基金进入详情页继续操作。

## 8. 管理后台页面

- **总览页**：AUM、NAV、份额总量、本周净流入 / 净流出、待处理申购 / 赎回、Agent 当日交易量。
- **Fund 配置页**：基金资料、Safe 地址、onboarding 状态、开放状态。
- **Onboarding 页**：Safe 部署（免 gas）→ Token Approve（免 gas）→ CLOB 注册 → 状态检查。每步显示进度和交易 ID。前置条件需配置 Builder API Key（key / secret / passphrase）。
- **Deal Cycle 处理页**：freeze、calculate NAV、preview、confirm、settle。
- **交易监控页**：Agent 提交的交易意图列表、CLOB 订单状态、完整审计日志。


## 9. Agent / Manager Onboarding 页面策略

- MVP 阶段不做公开自助注册页面。
- 先走 API + 内部运营后台创建 fund、通过 Relayer 部署 Safe 和 Approve、注册 CLOB。
- 后续可增加轻量 manager 页面，仅用于维护基金简介、头像、策略说明和页面预览。
