# Economy & Inventory Schema (MVP)

This document extends the base schema in `PROJECT.txt` with the minimum tables needed for the Sprint 2 economy loop (shop, inventory, harvest, and auditing).

## 1) `catalog_items`

Server-authoritative item catalog. Client can cache this data for display, but server values are source of truth.

```sql
CREATE TABLE catalog_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  item_code VARCHAR(64) NOT NULL UNIQUE,              -- e.g. seed_mint, crop_mint, egg_basic
  item_type VARCHAR(32) NOT NULL,                     -- seed | crop | furniture | egg | consumable
  display_name VARCHAR(128) NOT NULL,

  soft_price BIGINT NOT NULL DEFAULT 0 CHECK (soft_price >= 0),
  hard_price INT NOT NULL DEFAULT 0 CHECK (hard_price >= 0),

  growth_time_seconds INT,                            -- nullable unless item_type='seed'
  hatch_time_seconds INT,                             -- nullable unless item_type='egg'

  discord_sku VARCHAR(128),                           -- optional mapping for IAP bundles
  metadata JSONB NOT NULL DEFAULT '{}'::jsonb,

  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

  CONSTRAINT catalog_item_type_chk
    CHECK (item_type IN ('seed', 'crop', 'furniture', 'egg', 'consumable'))
);

CREATE INDEX idx_catalog_items_type_active ON catalog_items (item_type, is_active);
CREATE INDEX idx_catalog_items_sku ON catalog_items (discord_sku) WHERE discord_sku IS NOT NULL;
```

### Notes
- `item_code` is unique and stable for client/server protocol use.
- Price and timer values are stored only here (authoritative server config persisted in DB).

---

## 2) `inventory_items`

Per-user inventory stacks for seeds/crops/furniture/eggs and owned item metadata.

```sql
CREATE TABLE inventory_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  catalog_item_id UUID NOT NULL REFERENCES catalog_items(id) ON DELETE RESTRICT,

  quantity BIGINT NOT NULL DEFAULT 0 CHECK (quantity >= 0),
  stackable BOOLEAN NOT NULL DEFAULT TRUE,

  metadata JSONB NOT NULL DEFAULT '{}'::jsonb,        -- durability, cosmetic variant, quality rolls
  acquired_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

  CONSTRAINT uq_inventory_user_catalog UNIQUE (user_id, catalog_item_id)
);

CREATE INDEX idx_inventory_user ON inventory_items (user_id);
CREATE INDEX idx_inventory_user_updated ON inventory_items (user_id, updated_at DESC);
CREATE INDEX idx_inventory_catalog_lookup ON inventory_items (catalog_item_id, user_id);
```

### Notes
- Unique `(user_id, catalog_item_id)` enforces one row per user/item stack.
- For truly non-stackable objects later, move to a dedicated ownership table with one row per instance.

---

## 3) `currency_transactions`

Append-only ledger for all coins/gems changes. User balance (`users.coins`, `users.gems`) is a materialized fast-balance; this table is audit source of truth.

```sql
CREATE TABLE currency_transactions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,

  currency_type VARCHAR(16) NOT NULL,                 -- coins | gems
  amount_delta BIGINT NOT NULL,                       -- signed (+earn, -spend)
  balance_after BIGINT NOT NULL CHECK (balance_after >= 0),

  reason_code VARCHAR(64) NOT NULL,                   -- shop_purchase, harvest_reward, admin_grant, etc.
  reference_type VARCHAR(64),                         -- order, harvest, iap_receipt, migration
  reference_id UUID,

  metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

  CONSTRAINT currency_type_chk CHECK (currency_type IN ('coins', 'gems')),
  CONSTRAINT non_zero_delta_chk CHECK (amount_delta <> 0)
);

CREATE INDEX idx_currency_tx_user_time ON currency_transactions (user_id, created_at DESC);
CREATE INDEX idx_currency_tx_reason_time ON currency_transactions (reason_code, created_at DESC);
CREATE INDEX idx_currency_tx_reference ON currency_transactions (reference_type, reference_id)
  WHERE reference_type IS NOT NULL AND reference_id IS NOT NULL;
```

### Notes
- Never update/delete rows except legal/privacy workflows.
- `reason_code` should be constrained by application enum.

---

## 4) `harvest_events` (optional audit table)

Audit trail for crop harvest attempts and outcomes.

```sql
CREATE TABLE harvest_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,

  house_item_id UUID REFERENCES house_items(id) ON DELETE SET NULL,
  crop_catalog_item_id UUID REFERENCES catalog_items(id) ON DELETE SET NULL,

  planted_at TIMESTAMPTZ,
  attempted_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

  outcome VARCHAR(32) NOT NULL,                       -- success | not_ready | invalid_tile | rejected
  coins_awarded BIGINT NOT NULL DEFAULT 0 CHECK (coins_awarded >= 0),
  items_awarded JSONB NOT NULL DEFAULT '[]'::jsonb,   -- [{"item_code":"crop_mint","qty":2}]
  metadata JSONB NOT NULL DEFAULT '{}'::jsonb
);

CREATE INDEX idx_harvest_user_time ON harvest_events (user_id, attempted_at DESC);
CREATE INDEX idx_harvest_house_item_time ON harvest_events (house_item_id, attempted_at DESC);
CREATE INDEX idx_harvest_outcome_time ON harvest_events (outcome, attempted_at DESC);
```

---

## 5) Atomicity requirements (purchase & harvest)

All state changes for purchase and harvest must happen inside **one DB transaction**:

1. Lock target user balance row (`SELECT ... FOR UPDATE`).
2. Validate price/readiness on the server using authoritative catalog/timer data.
3. Update user balance (`users.coins` / `users.gems`).
4. Insert append-only ledger record into `currency_transactions`.
5. Upsert inventory stack in `inventory_items`.
6. (Harvest) update/reset `house_items.state` and optionally insert `harvest_events`.
7. Commit once all statements succeed; otherwise rollback everything.

Isolation recommendation: `READ COMMITTED` with explicit row locks is sufficient for MVP. If concurrent multi-action races still appear, upgrade operation to `SERIALIZABLE` retry loop.

---

## 6) Data retention policy

- `currency_transactions`: retain indefinitely (financial/economy audit table).
- `harvest_events`: keep **90 days hot** in primary DB; archive older partitions to cold storage (or delete after export in MVP if storage constrained).
- Add monthly partitioning when event volume grows (`created_at` / `attempted_at` partition key).

---

## 7) Example safe purchase transaction (debit + item grant)

Example: buy 3 `seed_mint` with coins.

```sql
BEGIN;

-- 1) Lock user row to serialize balance changes.
SELECT id, coins
FROM users
WHERE id = $1
FOR UPDATE;

-- 2) Fetch catalog row and compute total price server-side.
WITH item AS (
  SELECT id, soft_price
  FROM catalog_items
  WHERE item_code = 'seed_mint' AND is_active = TRUE
)
SELECT id, soft_price FROM item;

-- Application checks: user.coins >= item.soft_price * 3

-- 3) Debit balance.
UPDATE users
SET coins = coins - ($2::BIGINT),
    updated_at = NOW()
WHERE id = $1;

-- 4) Ledger append.
INSERT INTO currency_transactions (
  user_id, currency_type, amount_delta, balance_after,
  reason_code, reference_type, metadata
)
SELECT
  u.id,
  'coins',
  -$2::BIGINT,
  u.coins,
  'shop_purchase',
  'catalog_item',
  jsonb_build_object('item_code', 'seed_mint', 'qty', 3)
FROM users u
WHERE u.id = $1;

-- 5) Inventory upsert.
INSERT INTO inventory_items (user_id, catalog_item_id, quantity, metadata)
SELECT $1, c.id, 3, '{}'::jsonb
FROM catalog_items c
WHERE c.item_code = 'seed_mint'
ON CONFLICT (user_id, catalog_item_id)
DO UPDATE SET
  quantity = inventory_items.quantity + EXCLUDED.quantity,
  updated_at = NOW();

COMMIT;
```

If any check fails (insufficient funds, inactive catalog item, row not found), abort and `ROLLBACK` so no partial debit/grant can occur.
