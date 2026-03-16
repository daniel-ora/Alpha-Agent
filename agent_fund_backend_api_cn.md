**Agent 基金平台后端 API 设计**

MVP 接口分组与资源结构

1. API 分组原则
===============

-   对外展示接口使用 \`/api/funds/\...\`。

-   当前用户资产接口使用 \`/api/me/\...\`。

-   内部后台与运营接口使用 \`/api/admin/\...\`。

-   认证接口单独使用 \`/api/auth/\...\`。

2. 认证与用户
=============

  **方法**   **路径**                     **用途**             **权限**
  ---------- ---------------------------- -------------------- ----------
  POST       /api/auth/wallet/challenge   钱包登录 challenge   公开
  POST       /api/auth/wallet/verify      验证签名并登录       公开
  POST       /api/auth/logout             退出登录             登录用户
  GET        /api/auth/me                 获取当前用户信息     登录用户

3. 基金列表与详情
=================

  **方法**   **路径**                                 **用途**                           **权限**
  ---------- ---------------------------------------- ---------------------------------- ----------
  GET        /api/funds                               基金列表、筛选与排序               公开
  GET        /api/funds/{fundId}                      基金详情基础信息                   公开
  GET        /api/funds/{fundId}/performance          NAV 与回撤图表数据                 公开
  GET        /api/funds/{fundId}/holdings             当前持仓表                         公开
  GET        /api/funds/{fundId}/recent-activity      最近交易与时间线                   公开
  GET        /api/funds/{fundId}/deal-cycle           右侧操作栏需要的 deal cycle 信息   公开
  GET        /api/funds/{fundId}/historical-wallets   管理人历史钱包与历史战绩           公开

4. 申购接口
===========

  **方法**   **路径**                                               **用途**               **权限**
  ---------- ------------------------------------------------------ ---------------------- ----------
  POST       /api/funds/{fundId}/subscriptions/quote                申购弹窗预估 shares    登录用户
  POST       /api/funds/{fundId}/subscriptions                      提交申购申请           登录用户
  GET        /api/funds/{fundId}/subscriptions/{requestId}          查看单笔申购状态       登录用户
  GET        /api/me/subscriptions                                  查看我的全部申购记录   登录用户
  POST       /api/funds/{fundId}/subscriptions/{requestId}/cancel   截止前取消申购         登录用户

5. 赎回接口
===========

  **方法**   **路径**                                             **用途**               **权限**
  ---------- ---------------------------------------------------- ---------------------- ----------
  POST       /api/funds/{fundId}/redemptions/quote                赎回弹窗预估 USDC      登录用户
  POST       /api/funds/{fundId}/redemptions                      提交赎回申请           登录用户
  GET        /api/funds/{fundId}/redemptions/{requestId}          查看单笔赎回状态       登录用户
  GET        /api/me/redemptions                                  查看我的全部赎回记录   登录用户
  POST       /api/funds/{fundId}/redemptions/{requestId}/cancel   截止前取消赎回         登录用户

6. 我的投资
===========

  **方法**   **路径**                          **用途**                         **权限**
  ---------- --------------------------------- -------------------------------- ----------
  GET        /api/me/positions                 当前用户全部基金持仓             登录用户
  GET        /api/me/fund-positions/{fundId}   单只基金持仓与估值               登录用户
  GET        /api/me/investment-summary        累计申购 / 赎回 / 当前估值汇总   登录用户
  GET        /api/me/activity                  用户投资活动时间线               登录用户

7. 历史钱包认领
===============

  **方法**   **路径**                               **用途**                     **权限**
  ---------- -------------------------------------- ---------------------------- -----------------
  POST       /api/manager/wallet-claims/challenge   生成历史钱包签名 challenge   manager / admin
  POST       /api/manager/wallet-claims/verify      验证签名并完成钱包认领       manager / admin
  GET        /api/manager/wallet-claims             查看认领列表与状态           manager / admin
  DELETE     /api/manager/wallet-claims/{claimId}   删除 / 撤销认领              manager / admin

8. Fund / Manager 后台
======================

  **方法**   **路径**                           **用途**                 **权限**
  ---------- ---------------------------------- ------------------------ ----------
  POST       /api/admin/funds                   创建 fund                admin
  PATCH      /api/admin/funds/{fundId}          更新 fund 基础资料       admin
  GET        /api/admin/funds/{fundId}          后台查看 fund 配置详情   admin
  POST       /api/admin/funds/{fundId}/safe     绑定或初始化 Safe        admin
  GET        /api/admin/funds/{fundId}/safe     查看 Safe 信息           admin
  POST       /api/admin/funds/{fundId}/status   更新 fund 状态           admin

9. Module 与风险参数
====================

  **方法**   **路径**                                              **用途**                  **权限**
  ---------- ----------------------------------------------------- ------------------------- ----------
  GET        /api/admin/funds/{fundId}/module-config               读取 module 配置          admin
  PATCH      /api/admin/funds/{fundId}/module-config               更新 module 配置          admin
  GET        /api/admin/funds/{fundId}/market-whitelist            读取市场白名单            admin
  POST       /api/admin/funds/{fundId}/market-whitelist            新增白名单 market         admin
  DELETE     /api/admin/funds/{fundId}/market-whitelist/{itemId}   删除白名单 market         admin
  GET        /api/admin/funds/{fundId}/risk-limits                 读取风险限额              admin
  PATCH      /api/admin/funds/{fundId}/risk-limits                 更新风险限额              admin
  POST       /api/admin/funds/{fundId}/pause                       暂停 fund / module        admin
  POST       /api/admin/funds/{fundId}/resume                      恢复 fund / module        admin
  POST       /api/admin/funds/{fundId}/cancel-only                 切换为 cancel-only 模式   admin

10. Deal Cycle 与 NAV
=====================

  **方法**   **路径**                                                        **用途**                       **权限**
  ---------- --------------------------------------------------------------- ------------------------------ ----------
  GET        /api/admin/funds/{fundId}/deal-cycles                           查看 deal cycle 列表           admin
  GET        /api/admin/funds/{fundId}/deal-cycles/{cycleId}                 查看 cycle 详情                admin
  POST       /api/admin/funds/{fundId}/deal-cycles/{cycleId}/freeze          锁定本周期请求                 admin
  POST       /api/admin/funds/{fundId}/deal-cycles/{cycleId}/calculate-nav   计算 dealing NAV               admin
  POST       /api/admin/funds/{fundId}/deal-cycles/{cycleId}/preview         生成批处理预览                 admin
  POST       /api/admin/funds/{fundId}/deal-cycles/{cycleId}/confirm         确认本次 cycle                 admin
  POST       /api/admin/funds/{fundId}/deal-cycles/{cycleId}/settle          写入 NAV / shares / 结算结果   admin
  GET        /api/admin/funds/{fundId}/nav-snapshots                         查看 NAV 快照                  admin
  GET        /api/admin/funds/{fundId}/valuation-inputs                      查看估值输入与 TWAP 序列       admin

11. 同步与系统状态
==================

  **方法**   **路径**                                **用途**                            **权限**
  ---------- --------------------------------------- ----------------------------------- -------------
  GET        /api/admin/funds/{fundId}/sync-status   订单 / 成交 / 持仓 / 价格同步状态   admin
  POST       /api/admin/funds/{fundId}/resync        触发重同步                          admin
  GET        /api/admin/funds/{fundId}/jobs          查看后台任务状态                    admin
  GET        /api/health                             系统健康检查                        内部 / 监控

12. 建议的状态枚举
==================

  **模块**              **说明**
  --------------------- -------------------------------------------------------------------------------------
  subscription_status   pending / queued / frozen / settled / cancelled / failed
  redemption_status     pending / queued / frozen / nav_locked / settling / settled / cancelled / failed
  deal_cycle_status     upcoming / open / frozen / nav_calculated / confirmed / settling / settled / failed
  fund_status           draft / live / subscription_paused / redemption_paused / fully_paused / archived

13. MVP 最小必需接口集
======================

-   投资者侧：基金列表、基金详情、performance、holdings、deal-cycle、subscription quote / submit、redemption quote / submit、我的持仓、我的申购、我的赎回、历史钱包展示。

-   后台侧：创建 fund、更新 fund、module config 读写、deal cycle preview / confirm、sync status。
