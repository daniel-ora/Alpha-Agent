# Agent 基金平台 PRD

MVP 版本 - 单 Safe + 受限 Trading Module

## 1. 产品定位

- 这是一个面向 Agent 的预测市场基金平台。Agent 在 Polymarket 上交易，投资人基于透明的净值、持仓、风险和历史战绩进行申购。
- 平台的核心不是代替 Agent 交易，而是提供资金托管、权限控制、基金会计和投资者交互。

## 2. MVP 核心原则

- 每个基金只有一个系统生成的 Safe，所有正式基金资产都在该 Safe 内。
- Agent 不是 Safe owner，只能通过自定义 `PolymarketTradingModule` 获得受限交易能力。
- 提现、资产转出、参数修改、暂停和恢复等管理权限只属于 owner / multisig。
- 平台不承担执行责任；Agent 自行处理与 Polymarket 的 API 交互、重试、撤单和 heartbeat。
- 老钱包可被认领并展示为管理人的历史战绩，但不并入正式基金 NAV 和 shares。

## 3. 用户角色

- **投资者**：浏览基金、查看基金详情、提交申购与赎回、查看自己的份额与待处理请求。
- **管理人 / Owner**：配置基金资料、管理 Safe 与 module、管理白名单与风险参数、处理 deal cycle。
- **Agent**：运行策略并通过 module 调用基金 Safe 的受限交易能力。
- **运营 / Admin**：创建基金、初始化 Safe、审核历史钱包认领、处理异常和对账。

## 4. 账户与权限模型

| 模块 | 说明 |
|------|------|
| Fund Safe | 系统生成的唯一正式基金账户，承载全部正式基金资产、NAV、shares、申购与赎回。 |
| Owner Path | 用于 withdraw、transfer out、pause、配置变更、owner / threshold 调整、紧急恢复。 |
| Agent Path | 仅允许通过 `PolymarketTradingModule` 调用 `placeOrder`、`cancelOrder` 等受限交易动作。 |
| Claimed Wallets | 被管理人认领的历史钱包，仅展示历史战绩，不进入基金会计。 |

## 5. Onchain 模块设计

- `PolymarketTradingModule` 是单 Safe 架构的核心链上模块。
- MVP 只暴露 `placeOrder(...)` 与 `cancelOrder(...)` 两类接口。
- 采用白名单模型：仅允许指定目标地址、指定函数选择器、白名单 market / token。
- 必须限制单笔最大 notional、每日 / 每周最大成交额、最大未平仓敞口和最大挂单数。
- 紧急模式下只能 cancel，不能开新仓。
- 模块必须禁止任意 transfer、任意 approve、delegatecall、任意 external call 和治理改写。

## 6. 基金会计与估值

- 正式 NAV 只基于新 Safe 计算，公式为：Fund NAV = USDC 余额 + Σ(持仓数量 × fair_price) - 应计费用。
- 对于每个未结算 outcome token，使用 dealing window 内的 TWAP midpoint 作为 fair price。
- 被认领的历史钱包只生成管理人历史权益曲线、回撤、胜率和持仓统计，不进入正式基金 NAV。

## 7. 申购与赎回机制

- 采用固定批处理 Deal 机制，例如每周五 17:00-18:00。
- 一周内的申购 / 赎回请求先入队，到 cutoff 截止后统一估值。
- 申购公式：shares_minted = subscription_usdc / dealing_nav_per_share。
- 赎回公式：cash_due = shares_redeemed × dealing_nav_per_share。
- 赎回对用户只以 USDC 结算，基金内部负责卖出 Yes / No token 完成变现。

## 8. 冷启动策略

- 支持老钱包认领，用于展示管理人的基金成立前历史战绩。
- 老钱包必须通过链上签名验证认领。
- 页面展示必须明确标识为"管理人历史战绩"，并与正式基金业绩严格分层。

## 9. 非功能与运营要求

- 需要同步订单、成交、持仓与价格，并保留同步状态与重试机制。
- 需要支持 pause / resume / cancel-only 等风险控制操作。
- 需要支持 deal cycle 的 freeze、preview、confirm、settle 流程。
- 需要提供估值输入数据与 NAV 快照，便于审计和排查。

## 10. MVP 范围

- **做**：基金列表、基金详情、右侧申购 / 赎回操作栏、我的投资、后台 fund 配置、deal cycle 处理。
- **不做**：完整公开 manager 自助注册、复杂社交 feed、多基金组合器、用户 in-kind redemption。
