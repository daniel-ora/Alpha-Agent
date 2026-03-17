# Agent 基金平台 PRD

MVP 版本 - Backend Relay + Gnosis Safe

## 1. 产品定位

- 这是一个面向 Agent 的预测市场基金平台。Agent 在 Polymarket 上交易，投资人基于透明的净值、持仓和历史业绩进行申购。
- 平台的核心不是代替 Agent 交易，而是提供资金托管、权限控制、基金会计和投资者交互。

## 2. MVP 核心原则

- 每个基金只有一个通过 Polymarket SafeFactory 部署的 Gnosis Safe，所有正式基金资产都在该 Safe 内。
- Agent 不持有 Owner Key，只能通过 Backend Trading Relay 提交交易意图，由后端代签并提交到 Polymarket CLOB。
- 提现、资产转出、参数修改、暂停和恢复等管理权限只属于 Owner Key（平台通过 AWS KMS 控制）。
- Safe 部署和 Token 授权通过 Polymarket Builder Relayer 完成，全程免 gas 费。


## 3. 用户角色

- **投资者**：浏览基金、查看基金详情、提交申购与赎回、查看自己的份额与待处理请求。
- **管理人 / Owner**：配置基金资料、管理 Safe、处理 deal cycle。
- **Agent**：运行策略并通过 Backend Trading Relay API 提交交易意图（market、side、size、price），不直接接触 CLOB。
- **运营 / Admin**：创建基金、通过 Relayer 初始化 Safe、处理异常和对账。

## 4. 账户与权限模型

| 模块 | 说明 |
|------|------|
| Fund Safe | 通过 Polymarket SafeFactory 部署的 Gnosis Safe，承载全部正式基金资产、NAV、shares、申购与赎回。使用标准 CompatibilityFallbackHandler（不可替换，否则 Polymarket 会 ban 地址）。 |
| Owner Key | 平台通过 AWS KMS 管理。用于签署 Polymarket 订单（signatureType=2, POLY_GNOSIS_SAFE）、withdraw、transfer out、配置变更。 |
| Backend Trading Relay | Agent 提交交易意图 → Backend 用 AWS KMS (Owner Key) 签署 EIP-712 订单 → 提交 Polymarket CLOB。 |
| Builder Relayer | 通过 Polymarket Relayer 免 gas 执行链上操作（部署 Safe、Token Approve、赎回 CTF token）。需要 Builder API Key（在 polymarket.com/settings 开发者页面创建，含 key / secret / passphrase 三组凭证）。 |


## 5. 交易架构（已验证）

```
Agent (交易意图: market, side, size, price)
    → Backend Trading Relay API
        → 构造 EIP-712 Order (maker=Safe, signer=OwnerEOA, signatureType=2)
        → AWS KMS 签名（Owner Key）
        → Polymarket CLOB API (POST /order)
        → CTF Exchange 链上结算
```

**关键约束（Polygon Mainnet 实测确认）：**
- 每笔 Polymarket 订单必须由 Owner Key 签署（signatureType=2, POLY_GNOSIS_SAFE）
- 不能替换 Safe 的 Fallback Handler（Polymarket 检测到非标准 handler 会 ban Safe 地址）
- 不支持 signatureType=3 (POLY_1271)
- Agent 无法独立签署订单，必须经过 Backend Relay

## 6. Onboarding 流程（免 Gas）

通过 Polymarket Builder Relayer 完成，Owner 钱包无需持有 POL：

| 步骤 | 方式 | Gas 费 |
|------|------|--------|
| 部署 Safe | Builder Relayer (SAFE-CREATE) | 0 |
| Token Approve (7 个) | Builder Relayer (SAFE) | 0 |
| 创建 CLOB API Key | CLOB API (L1 签名) | 0 |
| 提交订单 | CLOB API (L2 HMAC) | 0 |

**前置条件：** 需要在 polymarket.com/settings 创建 Builder API Key（key / secret / passphrase）。Unverified 等级每天 100 次 Relayer 调用；申请 Verified（邮件 builder@polymarket.com）可升至 3,000 次/天。

详见 `polymarket_relayer_flow.md`。

## 7. 基金会计与估值

- 正式 NAV 只基于 Safe 计算，公式为：Fund NAV = USDC 余额 + Σ(持仓数量 × fair_price) - 应计费用。
- 对于每个未结算 outcome token，使用 dealing window 内的 TWAP midpoint 作为 fair price。


## 8. 申购与赎回机制

- 采用固定批处理 Deal 机制，例如每周五 17:00-18:00。
- 一周内的申购 / 赎回请求先入队，到 cutoff 截止后统一估值。
- 申购公式：shares_minted = subscription_usdc / dealing_nav_per_share。
- 赎回公式：cash_due = shares_redeemed × dealing_nav_per_share。
- 赎回对用户只以 USDC 结算，基金内部负责卖出 Yes / No token 完成变现。
- 已结算市场的 CTF token 赎回通过 Builder Relayer 免 gas 执行。

## 9. 非功能与运营要求

- 需要同步订单、成交、持仓与价格，并保留同步状态与重试机制。

- 需要支持 deal cycle 的 freeze、preview、confirm、settle 流程。
- 需要提供估值输入数据与 NAV 快照，便于审计和排查。
- Backend Trading Relay 需记录完整交易审计日志（Agent 意图 → 签名 → CLOB 响应）。

## 10. MVP 范围

- **做**：基金列表、基金详情、右侧申购 / 赎回操作栏、我的投资、后台 fund 配置、deal cycle 处理、Backend Trading Relay、Builder Relayer onboarding。
- **不做**：完整公开 manager 自助注册、复杂社交 feed、多基金组合器、用户 in-kind redemption、Agent 直接交易（已验证不可行）、历史钱包认领。
