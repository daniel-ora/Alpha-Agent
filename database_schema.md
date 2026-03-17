# Agent Fund 数据库设计（Supabase / PostgreSQL）

基于 Backend Relay 架构 + PRD 的 MVP 数据库设计，共 12 张表。

```
┌─────────────────────────────────────────────────┐
│  Platform Layer                                  │
│  ┌──────────┐  ┌───────────┐  ┌──────────────┐ │
│  │  users   │  │   funds   │  │ agent_keys   │ │
│  └──────────┘  └─────┬─────┘  └──────────────┘ │
│                      │                           │
├──────────────────────┼───────────────────────────┤
│  Trading Layer       │                           │
│  ┌───────────────┐  ┌┴─────────────┐            │
│  │ clob_creds    │  │ risk_configs  │            │
│  └───────────────┘  └──────────────┘            │
│  ┌──────────┐  ┌────────────┐                   │
│  │  orders  │  │ positions  │                   │
│  └──────────┘  └────────────┘                   │
│                                                  │
├──────────────────────────────────────────────────┤
│  Fund Accounting Layer                           │
│  ┌───────────────┐  ┌────────────────┐          │
│  │  deal_cycles  │  │ nav_snapshots  │          │
│  └───────────────┘  └────────────────┘          │
│  ┌────────────────┐  ┌─────────────────┐        │
│  │ subscriptions  │  │  redemptions    │        │
│  └────────────────┘  └─────────────────┘        │
│  ┌────────────────┐                              │
│  │  investments   │                              │
│  └────────────────┘                              │
└──────────────────────────────────────────────────┘
```

---

## Platform Layer

### 1. users — 所有用户

```sql
create table users (
  id              uuid primary key default gen_random_uuid(),
  wallet_address  text unique not null,
  role            text not null check (role in ('investor', 'manager', 'admin')),
  display_name    text,
  created_at      timestamptz default now()
);
```

### 2. funds — 基金

```sql
create table funds (
  id              uuid primary key default gen_random_uuid(),
  name            text not null,
  manager_id      uuid references users(id),
  status          text not null default 'draft'
                    check (status in ('draft', 'live', 'subscription_paused',
                      'redemption_paused', 'fully_paused', 'archived')),
  safe_address    text,
  owner_address   text,
  kms_key_id      text,
  onboarding      jsonb default '{}',
  created_at      timestamptz default now()
);
```

`onboarding` JSONB 示例：
```json
{
  "safe_deployed": true,
  "tokens_approved": true,
  "clob_registered": true,
  "risk_configured": true,
  "agent_key_issued": false
}
```

### 3. agent_keys — Agent API 密钥

```sql
create table agent_keys (
  id              uuid primary key default gen_random_uuid(),
  fund_id         uuid references funds(id) not null,
  key_hash        text not null,
  label           text,
  is_active       boolean default true,
  created_at      timestamptz default now(),
  revoked_at      timestamptz
);
```

- `key_hash`：bcrypt hash，原文只在创建时返回给 Agent 一次
- `label`：标识用途，如 `"agent-gpt-4o-prod"`

---

## Trading Layer

### 4. clob_credentials — CLOB API 凭证（加密存储）

```sql
create table clob_credentials (
  id              uuid primary key default gen_random_uuid(),
  fund_id         uuid references funds(id) unique not null,
  api_key         text not null,
  secret_enc      text not null,
  passphrase_enc  text not null,
  created_at      timestamptz default now()
);
```

- `secret_enc` / `passphrase_enc`：使用 pgcrypto 的 `pgp_sym_encrypt` 加密
- 解密密钥通过环境变量传入 Backend / Edge Function，不存储在数据库中

### 5. risk_configs — 风控参数

```sql
create table risk_configs (
  id                        uuid primary key default gen_random_uuid(),
  fund_id                   uuid references funds(id) unique not null,
  max_order_size            numeric,
  max_position_per_market   numeric,
  max_total_exposure        numeric,
  daily_volume_limit        numeric,
  order_rate_limit          int,
  market_whitelist          text[],
  updated_at                timestamptz default now()
);
```

- `market_whitelist`：允许的 tokenId 列表，用 `text[]` 而非独立表（变更低频，一次性读取）
- 所有金额单位为 USDC（6 decimals 对应的人类可读数值）

### 6. orders — 订单记录

```sql
create table orders (
  id              uuid primary key default gen_random_uuid(),
  fund_id         uuid references funds(id) not null,
  agent_key_id    uuid references agent_keys(id),
  clob_order_id   text,
  token_id        text not null,
  side            text not null check (side in ('BUY', 'SELL')),
  price           numeric not null,
  size            numeric not null,
  maker_amount    numeric,
  taker_amount    numeric,
  order_type      text default 'GTC',
  status          text not null
                    check (status in ('submitted', 'live', 'matched',
                      'cancelled', 'rejected', 'failed')),
  reject_reason   text,
  tx_hash         text,
  created_at      timestamptz default now(),
  updated_at      timestamptz default now()
);

create index idx_orders_fund_status on orders(fund_id, status);
create index idx_orders_fund_created on orders(fund_id, created_at desc);
```

### 7. positions — 持仓（链上同步）

```sql
create table positions (
  id              uuid primary key default gen_random_uuid(),
  fund_id         uuid references funds(id) not null,
  token_id        text not null,
  size            numeric not null,
  avg_entry_price numeric,
  current_price   numeric,
  last_synced_at  timestamptz,
  unique(fund_id, token_id)
);
```

- 由持仓同步服务定期从链上更新
- `current_price` 来自 CLOB 行情，用于 NAV 估算

---

## Fund Accounting Layer

### 8. deal_cycles — 申赎周期

```sql
create table deal_cycles (
  id              uuid primary key default gen_random_uuid(),
  fund_id         uuid references funds(id) not null,
  cycle_number    int not null,
  status          text not null default 'upcoming'
                    check (status in ('upcoming', 'open', 'frozen',
                      'nav_calculated', 'confirming', 'settling', 'settled', 'failed')),
  open_at         timestamptz,
  cutoff_at       timestamptz,
  nav_per_share   numeric,
  total_aum       numeric,
  settled_at      timestamptz,
  unique(fund_id, cycle_number)
);
```

### 9. subscriptions — 申购

```sql
create table subscriptions (
  id              uuid primary key default gen_random_uuid(),
  fund_id         uuid references funds(id) not null,
  user_id         uuid references users(id) not null,
  cycle_id        uuid references deal_cycles(id),
  usdc_amount     numeric not null,
  shares_minted   numeric,
  status          text not null default 'pending'
                    check (status in ('pending', 'queued', 'frozen',
                      'settled', 'cancelled', 'failed')),
  tx_hash         text,
  created_at      timestamptz default now()
);

create index idx_subscriptions_user on subscriptions(user_id, created_at desc);
create index idx_subscriptions_cycle on subscriptions(cycle_id, status);
```

### 10. redemptions — 赎回

```sql
create table redemptions (
  id              uuid primary key default gen_random_uuid(),
  fund_id         uuid references funds(id) not null,
  user_id         uuid references users(id) not null,
  cycle_id        uuid references deal_cycles(id),
  shares_amount   numeric not null,
  usdc_paid       numeric,
  status          text not null default 'pending'
                    check (status in ('pending', 'queued', 'frozen',
                      'nav_locked', 'settling', 'settled', 'cancelled', 'failed')),
  tx_hash         text,
  created_at      timestamptz default now()
);

create index idx_redemptions_user on redemptions(user_id, created_at desc);
create index idx_redemptions_cycle on redemptions(cycle_id, status);
```

### 11. investments — 投资者持仓（份额）

```sql
create table investments (
  id              uuid primary key default gen_random_uuid(),
  fund_id         uuid references funds(id) not null,
  user_id         uuid references users(id) not null,
  shares          numeric not null default 0,
  cost_basis      numeric default 0,
  updated_at      timestamptz default now(),
  unique(fund_id, user_id)
);
```

- `shares`：当前持有份额，申购后增加，赎回后减少
- `cost_basis`：累计投入 USDC，用于计算投资者收益

### 12. nav_snapshots — NAV 快照

```sql
create table nav_snapshots (
  id              uuid primary key default gen_random_uuid(),
  fund_id         uuid references funds(id) not null,
  nav_total       numeric not null,
  nav_per_share   numeric not null,
  total_shares    numeric not null,
  usdc_balance    numeric,
  positions_value numeric,
  snapshot_at     timestamptz not null
);

create index idx_nav_fund_time on nav_snapshots(fund_id, snapshot_at desc);
```

---

## Supabase 特有配置

### Row Level Security (RLS)

```sql
-- 投资者只能看自己的数据
alter table investments enable row level security;
create policy "users see own investments"
  on investments for select using (user_id = auth.uid());

alter table subscriptions enable row level security;
create policy "users see own subscriptions"
  on subscriptions for select using (user_id = auth.uid());

alter table redemptions enable row level security;
create policy "users see own redemptions"
  on redemptions for select using (user_id = auth.uid());

-- 基金数据公开可读
alter table funds enable row level security;
create policy "funds are public"
  on funds for select using (status != 'draft');

-- 交易数据按 fund_id 隔离（仅 Backend service role 访问）
alter table orders enable row level security;
alter table positions enable row level security;
alter table clob_credentials enable row level security;
```

### Realtime

对以下表开启 Realtime 推送：

- `orders` — 订单状态变化推送给前端
- `positions` — 持仓变化推送
- `nav_snapshots` — NAV 更新推送

### pgcrypto 加密

```sql
-- 加密 CLOB 凭证
create extension if not exists pgcrypto;

-- 写入时加密
insert into clob_credentials (fund_id, api_key, secret_enc, passphrase_enc)
values (
  $1,
  $2,
  pgp_sym_encrypt($3, current_setting('app.encryption_key')),
  pgp_sym_encrypt($4, current_setting('app.encryption_key'))
);

-- 读取时解密（仅在 Backend service role 中）
select
  api_key,
  pgp_sym_decrypt(secret_enc::bytea, current_setting('app.encryption_key')) as secret,
  pgp_sym_decrypt(passphrase_enc::bytea, current_setting('app.encryption_key')) as passphrase
from clob_credentials
where fund_id = $1;
```

---

## 表关系总览

```
users ──┬──< investments >──┬── funds
        ├──< subscriptions >─┤
        └──< redemptions >───┤
                              ├──── clob_credentials (1:1)
                              ├──── risk_configs (1:1)
                              ├──< agent_keys
                              ├──< orders
                              ├──< positions
                              ├──< deal_cycles ──< subscriptions
                              │                 ──< redemptions
                              └──< nav_snapshots
```

---

## 索引策略

| 表 | 索引 | 用途 |
|---|-------|------|
| orders | `(fund_id, status)` | 查询基金活跃订单 |
| orders | `(fund_id, created_at desc)` | 订单历史分页 |
| subscriptions | `(user_id, created_at desc)` | 投资者申购历史 |
| subscriptions | `(cycle_id, status)` | 周期批量结算 |
| redemptions | `(user_id, created_at desc)` | 投资者赎回历史 |
| redemptions | `(cycle_id, status)` | 周期批量结算 |
| nav_snapshots | `(fund_id, snapshot_at desc)` | 基金 NAV 历史曲线 |
| positions | `(fund_id, token_id)` UNIQUE | 持仓查询和更新 |
| investments | `(fund_id, user_id)` UNIQUE | 投资者份额查询 |
