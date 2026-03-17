# Agent 基金平台后端 API 设计

MVP 接口分组与资源结构

## 1. API 分组原则

- 对外展示接口使用 `/api/funds/...`。
- 当前用户资产接口使用 `/api/me/...`。
- Agent 交易接口使用 `/api/agent/...`。
- 内部后台与运营接口使用 `/api/admin/...`。
- 认证接口单独使用 `/api/auth/...`。

## 2. 认证与用户

| 方法 | 路径 | 用途 | 权限 |
|------|------|------|------|
| POST | /api/auth/wallet/challenge | 钱包登录 challenge | 公开 |
| POST | /api/auth/wallet/verify | 验证签名并登录 | 公开 |
| POST | /api/auth/logout | 退出登录 | 登录用户 |
| GET | /api/auth/me | 获取当前用户信息 | 登录用户 |

## 3. 基金列表与详情

| 方法 | 路径 | 用途 | 权限 |
|------|------|------|------|
| GET | /api/funds | 基金列表、筛选与排序 | 公开 |
| GET | /api/funds/{fundId} | 基金详情基础信息 | 公开 |
| GET | /api/funds/{fundId}/performance | NAV 与回撤图表数据 | 公开 |
| GET | /api/funds/{fundId}/holdings | 当前持仓表 | 公开 |
| GET | /api/funds/{fundId}/recent-activity | 最近交易与时间线 | 公开 |
| GET | /api/funds/{fundId}/deal-cycle | 右侧操作栏需要的 deal cycle 信息 | 公开 |


## 4. 申购接口

| 方法 | 路径 | 用途 | 权限 |
|------|------|------|------|
| POST | /api/funds/{fundId}/subscriptions/quote | 申购弹窗预估 shares | 登录用户 |
| POST | /api/funds/{fundId}/subscriptions | 提交申购申请 | 登录用户 |
| GET | /api/funds/{fundId}/subscriptions/{requestId} | 查看单笔申购状态 | 登录用户 |
| GET | /api/me/subscriptions | 查看我的全部申购记录 | 登录用户 |
| POST | /api/funds/{fundId}/subscriptions/{requestId}/cancel | 截止前取消申购 | 登录用户 |

## 5. 赎回接口

| 方法 | 路径 | 用途 | 权限 |
|------|------|------|------|
| POST | /api/funds/{fundId}/redemptions/quote | 赎回弹窗预估 USDC | 登录用户 |
| POST | /api/funds/{fundId}/redemptions | 提交赎回申请 | 登录用户 |
| GET | /api/funds/{fundId}/redemptions/{requestId} | 查看单笔赎回状态 | 登录用户 |
| GET | /api/me/redemptions | 查看我的全部赎回记录 | 登录用户 |
| POST | /api/funds/{fundId}/redemptions/{requestId}/cancel | 截止前取消赎回 | 登录用户 |

## 6. 我的投资

| 方法 | 路径 | 用途 | 权限 |
|------|------|------|------|
| GET | /api/me/positions | 当前用户全部基金持仓 | 登录用户 |
| GET | /api/me/fund-positions/{fundId} | 单只基金持仓与估值 | 登录用户 |
| GET | /api/me/investment-summary | 累计申购 / 赎回 / 当前估值汇总 | 登录用户 |
| GET | /api/me/activity | 用户投资活动时间线 | 登录用户 |

## 7. Agent Trading Relay（Backend 代签交易）

Agent 通过此接口提交交易意图，Backend 用 Owner Key (KMS) 签署订单并提交到 Polymarket CLOB。

| 方法 | 路径 | 用途 | 权限 |
|------|------|------|------|
| POST | /api/agent/funds/{fundId}/orders | 提交交易意图（market, side, size, price） | agent |
| DELETE | /api/agent/funds/{fundId}/orders/{orderId} | 取消订单 | agent |
| DELETE | /api/agent/funds/{fundId}/orders | 取消全部订单 | agent |
| GET | /api/agent/funds/{fundId}/orders | 查看当前挂单 | agent |
| GET | /api/agent/funds/{fundId}/orders/{orderId} | 查看单笔订单状态 | agent |
| GET | /api/agent/funds/{fundId}/positions | 查看当前持仓 | agent |
| GET | /api/agent/funds/{fundId}/balance | 查看 USDC.e 余额 | agent |
| GET | /api/agent/funds/{fundId}/markets | 查看可交易市场 | agent |

**POST /api/agent/funds/{fundId}/orders 请求体：**

```json
{
    "tokenId": "75467129...",
    "side": "BUY",
    "size": "10.0",
    "price": "0.55",
    "orderType": "GTC",
    "negRisk": false
}
```

**Backend 处理流程：**

1. 验证 Agent 身份和 fund 授权
2. 计算 makerAmount / takerAmount（6 位小数）
3. 构造 EIP-712 Order（maker=Safe, signer=OwnerEOA, signatureType=2）
4. AWS KMS 签署 orderHash（Owner Key）
5. 提交到 Polymarket CLOB API（附 Builder API Key 做订单归属）
6. 返回 orderId 和状态

## 8. Fund / Manager 后台

| 方法 | 路径 | 用途 | 权限 |
|------|------|------|------|
| POST | /api/admin/funds | 创建 fund | admin |
| PATCH | /api/admin/funds/{fundId} | 更新 fund 基础资料 | admin |
| GET | /api/admin/funds/{fundId} | 后台查看 fund 配置详情 | admin |
| GET | /api/admin/funds/{fundId}/safe | 查看 Safe 信息 | admin |

## 9. Onboarding（Safe 部署与配置）

通过 Polymarket Builder Relayer 免 gas 执行。需要平台级 Builder API Key（key / secret / passphrase，在 polymarket.com/settings 开发者页面创建）。

| 方法 | 路径 | 用途 | 权限 |
|------|------|------|------|
| POST | /api/admin/funds/{fundId}/onboarding/deploy-safe | 通过 Relayer 部署 Safe（免 gas） | admin |
| POST | /api/admin/funds/{fundId}/onboarding/approve-tokens | 通过 Relayer 执行 7 个 Token Approve（免 gas） | admin |
| POST | /api/admin/funds/{fundId}/onboarding/register-clob | 创建 CLOB API Key | admin |
| GET | /api/admin/funds/{fundId}/onboarding/status | 查看 onboarding 进度 | admin |

**凭证管理：** 每个基金 onboarding 产生的凭证需安全存储：

| 凭证 | 来源 | 用途 | 存储位置 |
|------|------|------|---------|
| Owner Key | HD 派生 (BIP-44) | 签署订单、Safe 管理 | AWS KMS |
| Builder API Key (key/secret/passphrase) | polymarket.com 开发者页面 | Relayer 免 gas 操作、订单归属 | 后端配置 / Secrets Manager |
| CLOB API Key (apiKey/secret/passphrase) | CLOB /auth/api-key | L2 HMAC 认证提交订单 | 后端加密存储 |

## 10. Deal Cycle 与 NAV

| 方法 | 路径 | 用途 | 权限 |
|------|------|------|------|
| GET | /api/admin/funds/{fundId}/deal-cycles | 查看 deal cycle 列表 | admin |
| GET | /api/admin/funds/{fundId}/deal-cycles/{cycleId} | 查看 cycle 详情 | admin |
| POST | /api/admin/funds/{fundId}/deal-cycles/{cycleId}/freeze | 锁定本周期请求 | admin |
| POST | /api/admin/funds/{fundId}/deal-cycles/{cycleId}/calculate-nav | 计算 dealing NAV | admin |
| POST | /api/admin/funds/{fundId}/deal-cycles/{cycleId}/preview | 生成批处理预览 | admin |
| POST | /api/admin/funds/{fundId}/deal-cycles/{cycleId}/confirm | 确认本次 cycle | admin |
| POST | /api/admin/funds/{fundId}/deal-cycles/{cycleId}/settle | 写入 NAV / shares / 结算结果 | admin |
| GET | /api/admin/funds/{fundId}/nav-snapshots | 查看 NAV 快照 | admin |
| GET | /api/admin/funds/{fundId}/valuation-inputs | 查看估值输入与 TWAP 序列 | admin |

## 11. 同步与系统状态

| 方法 | 路径 | 用途 | 权限 |
|------|------|------|------|
| GET | /api/admin/funds/{fundId}/sync-status | 订单 / 成交 / 持仓 / 价格同步状态 | admin |
| POST | /api/admin/funds/{fundId}/resync | 触发重同步 | admin |
| GET | /api/admin/funds/{fundId}/jobs | 查看后台任务状态 | admin |
| GET | /api/admin/funds/{fundId}/trading-audit-log | 查看交易审计日志 | admin |
| GET | /api/health | 系统健康检查 | 内部 / 监控 |

## 12. 建议的状态枚举

| 模块 | 说明 |
|------|------|
| subscription_status | pending / queued / frozen / settled / cancelled / failed |
| redemption_status | pending / queued / frozen / nav_locked / settling / settled / cancelled / failed |
| deal_cycle_status | upcoming / open / frozen / nav_calculated / confirmed / settling / settled / failed |
| fund_status | draft / onboarding / live / subscription_paused / redemption_paused / fully_paused / archived |
| order_status | submitted / live / matched / cancelled / failed |
| onboarding_status | not_started / safe_deploying / safe_deployed / approving / approved / clob_registered / completed |

## 13. MVP 最小必需接口集

- **投资者侧**：基金列表、基金详情、performance、holdings、deal-cycle、subscription quote / submit、redemption quote / submit、我的持仓、我的申购、我的赎回。
- **Agent 侧**：提交订单、取消订单、查看挂单、查看持仓、查看余额、查看可交易市场。
- **后台侧**：创建 fund、更新 fund、onboarding（deploy-safe / approve-tokens / register-clob）、deal cycle preview / confirm、sync status、trading audit log。
