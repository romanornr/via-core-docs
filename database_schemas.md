# Via Core Database Schemas Documentation

## Overview

Via runs three separate PostgreSQL databases, each with its own sqlx migration history:

| Database | Migrations directory | DAL crate |
|---|---|---|
| Main node DB | `core/lib/dal/migrations/` | `core/lib/dal/src/` |
| Verifier DB | `via_verifier/lib/verifier_dal/migrations/` | `via_verifier/lib/verifier_dal/src/` |
| L1 Indexer DB | `via_indexer/lib/via_indexer_dal/migrations/` | `via_indexer/lib/via_indexer_dal/src/` |

All DDL below is copied verbatim from the migration files. Each SQL block is headed by a comment naming its source migration. Where later migrations alter a table, those ALTERs are shown after the original CREATE.

## Main Node Database (core/lib/dal)

The main node database is the inherited zksync-era schema (232 up migrations) plus Via-specific tables added by migrations whose filenames contain `via`.

### Inherited zksync-era schema

> **CORRECTION (2026-07-02):** The previous version of this section showed "final state" DDL for
> `l1_batches`, `transactions`, `storage_logs`, and `events` that does not appear in any migration file
> (it was reconstructed from memory of upstream zksync-era). These tables are real, but their current
> shape is the product of the init migration plus roughly twenty ALTER migrations each, so no single
> verbatim CREATE TABLE reflecting the final state exists in the repo.

Verified facts about the inherited schema:

- `core/lib/dal/migrations/20211026134308_init.up.sql` creates the original tables, including `storage`, `eth_txs`, `eth_txs_history`, `blocks`, `transactions`, `storage_logs`, `contracts`, `events`, `factory_deps`, and `tokens`.
- `core/lib/dal/migrations/20220827110416_miniblocks.up.sql` contains `ALTER TABLE blocks RENAME TO l1_batches;`, which is where the `l1_batches` table gets its name.
- `l1_batches` is subsequently modified by 21 later migrations and `transactions` by 20, so consult the migration history (or a live database) for the current column set rather than any single file.
- `protocol_versions` is created by `20230427083744_protocol_versions_table.up.sql` and `protocol_patches` by `20240527082006_add_protocol_vk_patches_table.up.sql` (both shown verbatim below).

As a verified sample, the original (2021) forms of two inherited tables, as created by the init migration. Note that later migrations alter these tables, so this is their initial shape, not their current one:

```sql
-- core/lib/dal/migrations/20211026134308_init.up.sql (original form; altered by later migrations)
CREATE TABLE storage_logs
(
    id BIGSERIAL PRIMARY KEY,
    raw_key BYTEA NOT NULL,
    address BYTEA NOT NULL,
    key BYTEA NOT NULL,
    value BYTEA NOT NULL,
    operation_number INT NOT NULL,
    tx_hash BYTEA NOT NULL,
    block_number BIGINT NOT NULL REFERENCES blocks (number) ON DELETE CASCADE,

    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL
);
```

```sql
-- core/lib/dal/migrations/20211026134308_init.up.sql (original form; altered by later migrations)
CREATE TABLE events
(
    id BIGSERIAL PRIMARY KEY,
    block_number BIGINT NOT NULL REFERENCES blocks (number) ON DELETE CASCADE,
    tx_hash BYTEA NOT NULL,
    tx_index_in_block INT NOT NULL,
    address BYTEA NOT NULL,

    event_index_in_block INT NOT NULL,
    event_index_in_tx INT NOT NULL,

    topic1 BYTEA NOT NULL,
    topic2 BYTEA NOT NULL,
    topic3 BYTEA NOT NULL,
    topic4 BYTEA NOT NULL,

    value BYTEA NOT NULL,

    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL
);
```

#### Protocol Versions Table (inherited, main DB)

```sql
-- core/lib/dal/migrations/20230427083744_protocol_versions_table.up.sql
CREATE TABLE IF NOT EXISTS protocol_versions (
    id INT PRIMARY KEY,
    timestamp BIGINT NOT NULL,
    recursion_scheduler_level_vk_hash BYTEA NOT NULL,
    recursion_node_level_vk_hash BYTEA NOT NULL,
    recursion_leaf_level_vk_hash BYTEA NOT NULL,
    recursion_circuits_set_vks_hash BYTEA NOT NULL,
    bootloader_code_hash BYTEA NOT NULL,
    default_account_code_hash BYTEA NOT NULL,
    verifier_address BYTEA NOT NULL,
    upgrade_tx_hash BYTEA REFERENCES transactions (hash),
    created_at TIMESTAMP NOT NULL
);
```

#### Protocol Patches Table (inherited, main DB)

```sql
-- core/lib/dal/migrations/20240527082006_add_protocol_vk_patches_table.up.sql
CREATE TABLE protocol_patches (
    minor INTEGER NOT NULL REFERENCES protocol_versions(id),
    patch INTEGER NOT NULL,
    recursion_scheduler_level_vk_hash BYTEA NOT NULL,
    recursion_node_level_vk_hash      BYTEA NOT NULL,
    recursion_leaf_level_vk_hash      BYTEA NOT NULL,
    recursion_circuits_set_vks_hash   BYTEA NOT NULL,
    created_at TIMESTAMP NOT NULL,
    PRIMARY KEY (minor, patch)
);
```

The same migration then relaxes the vk hash columns on `protocol_versions`:

```sql
-- core/lib/dal/migrations/20240527082006_add_protocol_vk_patches_table.up.sql
ALTER TABLE protocol_versions ALTER COLUMN recursion_scheduler_level_vk_hash DROP NOT NULL;
ALTER TABLE protocol_versions ALTER COLUMN recursion_node_level_vk_hash DROP NOT NULL;
ALTER TABLE protocol_versions ALTER COLUMN recursion_leaf_level_vk_hash DROP NOT NULL;
ALTER TABLE protocol_versions ALTER COLUMN recursion_circuits_set_vks_hash DROP NOT NULL;
```

### Via-specific tables (main DB)

#### Bitcoin Inscription Request Tables

Used by the BTC sender to build, sign, and track commit/reveal inscription transactions on Bitcoin. DAL: `core/lib/dal/src/via_btc_sender_dal.rs` and `core/lib/dal/src/via_blocks_dal.rs`.

```sql
-- core/lib/dal/migrations/20240906134623_add_via_btc_inscription_requests.up.sql
CREATE TABLE "via_btc_inscriptions_request" (
  "id" BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  "l1_batch_number" BIGINT NOT NULL,
  "request_type" varchar NOT NULL,
  "inscription_message" BYTEA,
  "predicted_fee" BIGINT,
  "confirmed_inscriptions_request_history_id" BIGINT UNIQUE,
  "created_at" timestamp NOT NULL DEFAULT 'now()',
  "updated_at" timestamp NOT NULL
);

CREATE TABLE "via_btc_inscriptions_request_history" (
  "id" BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  "commit_tx_id" BYTEA UNIQUE NOT NULL,
  "reveal_tx_id" BYTEA UNIQUE NOT NULL,
  "inscription_request_id" BIGINT NOT NULL,
  "signed_commit_tx" BYTEA NOT NULL,
  "signed_reveal_tx" BYTEA NOT NULL,
  "actual_fees" BIGINT NOT NULL,
  "confirmed_at" timestamp DEFAULT null,
  "sent_at_block" BIGINT NOT NULL,
  "created_at" timestamp NOT NULL DEFAULT 'now()',
  "updated_at" timestamp NOT NULL
);

CREATE TABLE "via_l1_batch_inscription_request" (
  "l1_batch_number" BIGINT UNIQUE NOT NULL,
  "commit_l1_batch_inscription_id" BIGINT UNIQUE NOT NULL,
  "commit_proof_inscription_id" BIGINT UNIQUE,
  "is_finalized" BOOLEAN,
  "created_at" timestamp NOT NULL DEFAULT 'now()',
  "updated_at" timestamp NOT NULL
);
ALTER TABLE "via_btc_inscriptions_request" ADD FOREIGN KEY ("confirmed_inscriptions_request_history_id") REFERENCES "via_btc_inscriptions_request_history" ("id") ON DELETE CASCADE ON UPDATE NO ACTION;;
ALTER TABLE "via_btc_inscriptions_request" ADD FOREIGN KEY ("l1_batch_number") REFERENCES "l1_batches" ("number") ON DELETE CASCADE ON UPDATE NO ACTION;
ALTER TABLE "via_btc_inscriptions_request_history" ADD FOREIGN KEY ("inscription_request_id") REFERENCES "via_btc_inscriptions_request" ("id") ON DELETE CASCADE ON UPDATE NO ACTION;
ALTER TABLE "via_l1_batch_inscription_request" ADD FOREIGN KEY ("l1_batch_number") REFERENCES "l1_batches" ("number") ON DELETE CASCADE ON UPDATE NO ACTION;
ALTER TABLE "via_l1_batch_inscription_request" ADD FOREIGN KEY ("commit_l1_batch_inscription_id") REFERENCES "via_btc_inscriptions_request" ("id") ON DELETE CASCADE ON UPDATE NO ACTION;
ALTER TABLE "via_l1_batch_inscription_request" ADD FOREIGN KEY ("commit_proof_inscription_id") REFERENCES "via_btc_inscriptions_request" ("id") ON DELETE CASCADE ON UPDATE NO ACTION;
```

#### Data Availability Table

Tracks pubdata and proof blobs dispatched to the DA layer (Celestia). DAL: `core/lib/dal/src/via_data_availability_dal.rs`.

```sql
-- core/lib/dal/migrations/20240916094925_create_via_data_availability.up.sql
CREATE TABLE IF NOT EXISTS via_data_availability
(
    l1_batch_number BIGINT NOT NULL REFERENCES l1_batches (number) ON DELETE CASCADE,
    is_proof        BOOLEAN NOT NULL,

    blob_id         TEXT      NOT NULL, -- blob here is an abstract term, unrelated to any DA implementation
    inclusion_data  BYTEA,
    sent_at         TIMESTAMP NOT NULL,

    created_at      TIMESTAMP NOT NULL,
    updated_at      TIMESTAMP NOT NULL,

    PRIMARY KEY (l1_batch_number, is_proof) -- for ensuring uniqueness of the combination of l1_batch_number and is_proof
);
```

A later migration adds an `index` column so a batch can be split into multiple blob chunks:

```sql
-- core/lib/dal/migrations/20251030121202_via_da_chunk_blob.up.sql
-- 1. Add the new index column with default 0
ALTER TABLE via_data_availability
ADD COLUMN index INTEGER NOT NULL DEFAULT 0;

-- 2. Drop the old primary key constraint
ALTER TABLE via_data_availability
DROP CONSTRAINT via_data_availability_pkey;

-- 3. Add the new primary key including the index column
ALTER TABLE via_data_availability
ADD PRIMARY KEY ("l1_batch_number", "is_proof", "index");
```

Query excerpt from `core/lib/dal/src/via_data_availability_dal.rs`:

```sql
INSERT INTO
via_data_availability (
    l1_batch_number, is_proof, index, blob_id, sent_at, created_at, updated_at
)
VALUES
($1, FALSE, $2, $3, $4, NOW(), NOW())
ON CONFLICT DO NOTHING
```

#### Via Votes Table (main DB)

The sequencer's view of verifier votes per batch. Note this has a different shape than the verifier database's `via_votes` table. DAL: `core/lib/dal/src/via_votes_dal.rs`.

```sql
-- core/lib/dal/migrations/20241209150000_create_via_votes.up.sql
CREATE TABLE IF NOT EXISTS via_votes (
    l1_batch_number BIGINT NOT NULL,
    proof_reveal_tx_id BYTEA NOT NULL,
    verifier_address TEXT NOT NULL,
    vote BOOLEAN NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    PRIMARY KEY(l1_batch_number, proof_reveal_tx_id, verifier_address)
);

ALTER TABLE "via_votes" ADD FOREIGN KEY ("l1_batch_number") REFERENCES "l1_batches" ("number") ON DELETE CASCADE ON UPDATE NO ACTION;
```

#### Via Indexer Metadata Table (main DB)

Cursor for the node's own Bitcoin indexing. DAL: `core/lib/dal/src/via_indexer_dal.rs`.

```sql
-- core/lib/dal/migrations/20250414100204_via_indexer_metadata.up.sql
CREATE TABLE IF NOT EXISTS via_indexer_metadata (
    module VARCHAR UNIQUE NOT NULL,
    last_indexer_l1_block BIGINT NOT NULL,
    updated_at TIMESTAMP NOT NULL
);
```

#### Via Wallets Table (main DB)

System wallet addresses (by role) discovered from governance inscriptions. DAL: `core/lib/dal/src/via_wallet_dal.rs`.

```sql
-- core/lib/dal/migrations/20250731184639_via_wallet_migration.up.sql
CREATE TABLE via_wallets (
    id BIGSERIAL PRIMARY KEY,
    role VARCHAR NOT NULL,
    address VARCHAR NOT NULL,
    tx_hash VARCHAR NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT unique_tx_hash_address_role UNIQUE (tx_hash, address, role)
);

CREATE INDEX idx_via_wallets_role_created_at ON via_wallets(role, created_at DESC);
```

#### L1 Blocks and Reorg Tables (main DB)

Track observed Bitcoin block hashes and detected reorgs; the same migration also adds `l1_block_number` to `via_wallets`. DAL: `core/lib/dal/src/via_l1_block_dal.rs`.

```sql
-- core/lib/dal/migrations/20250905110631_via_l1_blocks.up.sql
BEGIN;
    ALTER TABLE via_wallets
    ADD COLUMN l1_block_number BIGINT NOT NULL DEFAULT 0;

    CREATE INDEX idx_via_wallets_l1_block_number
    ON via_wallets(l1_block_number);
COMMIT;

CREATE TABLE via_l1_blocks (
    "number" BIGINT UNIQUE NOT NULL,
    "hash" VARCHAR UNIQUE NOT NULL
);

CREATE TABLE via_l1_reorg (
    "l1_block_number" BIGINT UNIQUE NOT NULL,
    "l1_batch_number" BIGINT UNIQUE NOT NULL,
    "created_at" TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Verifier Database Schema

> **CORRECTION (2026-04-11):** The previous version fabricated `verifier_votes`, `signing_sessions`, and
> `verifier_registry` tables. None of those exist. The actual verifier database schema is in
> `via_verifier/lib/verifier_dal/migrations/`. MuSig2 signing sessions are managed in-memory
> by the coordinator, not persisted in a database table.

The verifier uses a separate database (`verifier_dal`) with its own migration history (10 up migrations as of this writing).

### Verifier Inscription Request Tables

The verifier variant of the inscription tables. It differs from the main DB variant: no `l1_batch_number` column on the request table, `commit_tx_id`/`reveal_tx_id` are `varchar` instead of `BYTEA`, and there is a `via_l1_batch_vote_inscription_request` table instead of `via_l1_batch_inscription_request`. DAL: `via_verifier/lib/verifier_dal/src/via_btc_sender_dal.rs`.

```sql
-- via_verifier/lib/verifier_dal/migrations/20240906134623_add_via_btc_inscription_requests.up.sql
CREATE TABLE "via_btc_inscriptions_request" (
  "id" BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  "request_type" varchar NOT NULL,
  "inscription_message" BYTEA,
  "predicted_fee" bigint,
  "confirmed_inscriptions_request_history_id" bigint UNIQUE,
  "created_at" timestamp NOT NULL DEFAULT 'now()',
  "updated_at" timestamp NOT NULL
);

CREATE TABLE "via_btc_inscriptions_request_history" (
  "id" BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  "commit_tx_id" varchar UNIQUE NOT NULL,
  "reveal_tx_id" varchar UNIQUE NOT NULL,
  "inscription_request_id" bigint NOT NULL,
  "signed_commit_tx" BYTEA NOT NULL,
  "signed_reveal_tx" BYTEA NOT NULL,
  "actual_fees" bigint NOT NULL,
  "confirmed_at" timestamp DEFAULT null,
  "sent_at_block" bigint NOT NULL,
  "created_at" timestamp NOT NULL DEFAULT 'now()',
  "updated_at" timestamp NOT NULL
);

CREATE TABLE "via_l1_batch_vote_inscription_request" (
  "votable_transaction_id" bigint UNIQUE NOT NULL,
  "vote_l1_batch_inscription_id" bigint UNIQUE NOT NULL,
  "created_at" timestamp NOT NULL DEFAULT 'now()',
  "updated_at" timestamp NOT NULL
);

ALTER TABLE "via_btc_inscriptions_request_history" ADD FOREIGN KEY ("inscription_request_id") REFERENCES "via_btc_inscriptions_request" ("id") ON DELETE CASCADE ON UPDATE NO ACTION;
ALTER TABLE "via_btc_inscriptions_request" ADD FOREIGN KEY ("confirmed_inscriptions_request_history_id") REFERENCES "via_btc_inscriptions_request_history" ("id");
ALTER TABLE "via_l1_batch_vote_inscription_request" ADD FOREIGN KEY ("vote_l1_batch_inscription_id") REFERENCES "via_btc_inscriptions_request" ("id");
```

### Votable Transactions Table

Tracks L1 batches that verifiers need to vote on. DAL: `via_verifier/lib/verifier_dal/src/via_votes_dal.rs`.

```sql
-- via_verifier/lib/verifier_dal/migrations/20250112053854_create_via_votes.up.up.sql
CREATE TABLE IF NOT EXISTS via_votable_transactions (
    id BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    l1_batch_number BIGINT NOT NULL,
    l1_batch_hash BYTEA UNIQUE NOT NULL,
    prev_l1_batch_hash BYTEA NOT NULL,
    proof_blob_id VARCHAR UNIQUE NOT NULL,
    proof_reveal_tx_id BYTEA UNIQUE NOT NULL,
    pubdata_blob_id VARCHAR UNIQUE NOT NULL,
    pubdata_reveal_tx_id VARCHAR UNIQUE NOT NULL,
    da_identifier VARCHAR NOT NULL,
    bridge_tx_id BYTEA,
    is_finalized BOOLEAN,
    l1_batch_status BOOLEAN,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_via_votable_transactions_l1_batch_hash ON via_votable_transactions(l1_batch_hash);
CREATE INDEX idx_via_votable_transactions_finalized ON via_votable_transactions (is_finalized) WHERE is_finalized IS NOT NULL;
CREATE INDEX idx_via_votable_transactions_status ON via_votable_transactions (l1_batch_status) WHERE l1_batch_status IS NOT NULL;
CREATE INDEX idx_via_votable_transactions_batch_tx ON via_votable_transactions (bridge_tx_id) WHERE bridge_tx_id IS NOT NULL;
```

Later migrations change this table:

```sql
-- via_verifier/lib/verifier_dal/migrations/20250714093607_bridge_tx.up.sql (excerpt)
-- Drop the bridge_tx_id column
ALTER TABLE via_votable_transactions DROP COLUMN bridge_tx_id;
```

```sql
-- via_verifier/lib/verifier_dal/migrations/20250730193407_fix_votable_transaction.up.sql
-- Allow duplicate batch numbers, but only one can be in canonical chain
CREATE UNIQUE INDEX unique_canonical_batch
ON via_votable_transactions (l1_batch_number)
WHERE is_finalized = TRUE OR is_finalized IS NULL;
```

So in the current schema `via_votable_transactions` has no `bridge_tx_id` column (and the `idx_via_votable_transactions_batch_tx` index on it is gone with the column).

Query excerpt from `via_verifier/lib/verifier_dal/src/via_votes_dal.rs`:

```sql
INSERT INTO
via_votable_transactions (
    l1_batch_number,
    l1_batch_hash,
    prev_l1_batch_hash,
    proof_reveal_tx_id,
    da_identifier,
    proof_blob_id,
    pubdata_reveal_tx_id,
    pubdata_blob_id
)
VALUES
($1, $2, $3, $4, $5, $6, $7, $8)
ON CONFLICT (l1_batch_hash) DO NOTHING
```

### Verifier Votes Table

Stores per-verifier votes referencing votable transactions. DAL: `via_verifier/lib/verifier_dal/src/via_votes_dal.rs`.

```sql
-- via_verifier/lib/verifier_dal/migrations/20250112053854_create_via_votes.up.up.sql
CREATE TABLE IF NOT EXISTS via_votes (
    id BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    votable_transaction_id BIGINT NOT NULL,
    verifier_address TEXT NOT NULL,
    vote BOOLEAN NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE (votable_transaction_id, verifier_address)
);

ALTER TABLE "via_votes" ADD FOREIGN KEY ("votable_transaction_id") REFERENCES "via_votable_transactions" ("id") ON DELETE CASCADE ON UPDATE NO ACTION;
```

Query excerpt from `via_verifier/lib/verifier_dal/src/via_votes_dal.rs`:

```sql
INSERT INTO
via_votes (votable_transaction_id, verifier_address, vote)
VALUES
($1, $2, $3)
ON CONFLICT (votable_transaction_id, verifier_address) DO NOTHING
```

### Verifier Transactions Table

Tracks priority (deposit) transactions observed by the verifier. DAL: `via_verifier/lib/verifier_dal/src/via_transactions_dal.rs`.

```sql
-- via_verifier/lib/verifier_dal/migrations/20250207164238_via_add_transactions_dal.up.sql
CREATE TABLE IF NOT EXISTS via_transactions (
    "priority_id" BIGINT NOT NULL,
    "tx_id" BYTEA NOT NULL,
    "receiver" VARCHAR NOT NULL,
    "value" BIGINT NOT NULL,
    "calldata" BYTEA,
    "canonical_tx_hash" BYTEA NOT NULL,
    "status"  BOOLEAN,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    PRIMARY KEY (tx_id)
);

CREATE INDEX idx_via_transactions_priority ON via_transactions(priority_id);
```

A later migration adds L1 tracking columns:

```sql
-- via_verifier/lib/verifier_dal/migrations/20250905110631_via_l1_blocks.up.sql (excerpt)
BEGIN;
    ALTER TABLE via_transactions
    ADD COLUMN l1_block_number BIGINT NOT NULL DEFAULT 0;

    ALTER TABLE via_transactions
    ADD COLUMN l1_batch_number BIGINT DEFAULT NULL;

    CREATE INDEX idx_via_transactions_l1_block_number
    ON via_transactions(l1_block_number);
COMMIT;
```

### Verifier Protocol Version Tables

The verifier keeps its own trimmed-down protocol version tables (a different shape than the main DB variants). DAL: `via_verifier/lib/verifier_dal/src/via_protocol_versions_dal.rs`.

```sql
-- via_verifier/lib/verifier_dal/migrations/20250315203029_protocol_version.up.sql
CREATE TABLE IF NOT EXISTS protocol_versions (
    id INT PRIMARY KEY,
    bootloader_code_hash BYTEA NOT NULL,
    default_account_code_hash BYTEA NOT NULL,
    upgrade_tx_hash BYTEA UNIQUE NOT NULL,
    recursion_scheduler_level_vk_hash BYTEA NOT NULL,
    executed BOOLEAN NOT NULL,
    created_at TIMESTAMP NOT NULL
);

CREATE TABLE protocol_patches (
    minor INTEGER NOT NULL REFERENCES protocol_versions(id),
    patch INTEGER NOT NULL,
    created_at TIMESTAMP NOT NULL,
    PRIMARY KEY (minor, patch)
);
```

### Verifier Indexer Metadata Table

DAL: `via_verifier/lib/verifier_dal/src/via_indexer_dal.rs`.

```sql
-- via_verifier/lib/verifier_dal/migrations/20250414100204_via_indexer_metadata.up.sql
CREATE TABLE IF NOT EXISTS via_indexer_metadata (
    module VARCHAR UNIQUE NOT NULL,
    last_indexer_l1_block BIGINT NOT NULL,
    updated_at TIMESTAMP NOT NULL
);
```

### Bridge Transaction Table (created, then dropped)

`via_bridge_tx` was introduced to hold bridge transaction hashes per votable transaction (migrating data out of the dropped `bridge_tx_id` column), then removed two months later in favor of the withdrawal tables below. It does not exist in the current schema.

```sql
-- via_verifier/lib/verifier_dal/migrations/20250714093607_bridge_tx.up.sql
CREATE TABLE via_bridge_tx (
    "id" BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    "votable_tx_id" BIGINT REFERENCES via_votable_transactions(id),
    "hash" BYTEA,
    "data" BYTEA DEFAULT NULL,
    "index" BIGINT NOT NULL DEFAULT 0,
    "created_at" TIMESTAMP NOT NULL DEFAULT NOW(),
    "updated_at" TIMESTAMP NOT NULL DEFAULT NOW(),
    CONSTRAINT unique_votable_index UNIQUE ("votable_tx_id", "index")
);
```

```sql
-- via_verifier/lib/verifier_dal/migrations/20250920102508_witdrawals.up.sql (excerpt)
DROP TABLE via_bridge_tx;
```

### Verifier Wallets Table

Same shape as the main DB `via_wallets`. DAL: `via_verifier/lib/verifier_dal/src/via_wallet_dal.rs`.

```sql
-- via_verifier/lib/verifier_dal/migrations/20250731184639_via_wallet_migration.up.sql
CREATE TABLE via_wallets (
    id BIGSERIAL PRIMARY KEY,
    role VARCHAR NOT NULL,
    address VARCHAR NOT NULL,
    tx_hash VARCHAR NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT unique_tx_hash_address_role UNIQUE (tx_hash, address, role)
);

CREATE INDEX idx_via_wallets_role_created_at ON via_wallets(role, created_at DESC);
```

The `20250905110631_via_l1_blocks.up.sql` migration later adds `l1_block_number BIGINT NOT NULL DEFAULT 0` and an index on it, exactly as in the main DB.

### Verifier L1 Blocks and Reorg Tables

DAL: `via_verifier/lib/verifier_dal/src/via_l1_block_dal.rs`.

```sql
-- via_verifier/lib/verifier_dal/migrations/20250905110631_via_l1_blocks.up.sql (excerpt)
CREATE TABLE via_l1_blocks (
    "number" BIGINT UNIQUE NOT NULL,
    "hash" VARCHAR UNIQUE NOT NULL
);

CREATE TABLE via_l1_reorg (
    "l1_block_number" BIGINT UNIQUE NOT NULL,
    "l1_batch_number" BIGINT UNIQUE NOT NULL,
    "created_at" TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Withdrawal Tables

Track L2-to-Bitcoin withdrawals and the bridge transactions that execute them. Note the migration filename is misspelled (`witdrawals`) in the repo. DAL: `via_verifier/lib/verifier_dal/src/withdrawals_dal.rs`.

```sql
-- via_verifier/lib/verifier_dal/migrations/20250920102508_witdrawals.up.sql
DROP TABLE via_bridge_tx;

CREATE TABLE IF NOT EXISTS via_l1_batch_bridge_withdrawals (
    "proof_reveal_tx_id" BYTEA NOT NULL UNIQUE,
    "created_at" TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS via_bridge_withdrawals (
    "id" BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    "tx_id" BYTEA NOT NULL UNIQUE,
    "executed" BOOLEAN NOT NULL DEFAULT FALSE,
    "updated_at" TIMESTAMP NOT NULL DEFAULT NOW(),
    "created_at" TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS via_withdrawals (
    "id" VARCHAR UNIQUE NOT NULL,
    "bridge_withdrawal_id" BIGINT REFERENCES via_bridge_withdrawals(id) ON DELETE CASCADE DEFAULT NULL,
    "l2_tx_hash" VARCHAR NOT NULL,
    "l2_tx_log_index" BIGINT NOT NULL,
    "receiver" VARCHAR NOT NULL,
    "value" BIGINT NOT NULL,
    "updated_at" TIMESTAMP NOT NULL DEFAULT NOW(),
    "created_at" TIMESTAMP NOT NULL DEFAULT NOW()
);
```

Query excerpt from `via_verifier/lib/verifier_dal/src/withdrawals_dal.rs`:

```sql
INSERT INTO
via_withdrawals (id, l2_tx_hash, l2_tx_log_index, receiver, value)
VALUES
($1, $2, $3, $4, $5)
ON CONFLICT (id)
DO UPDATE SET
value = excluded.value,
l2_tx_hash = CASE
    WHEN excluded.l2_tx_hash <> '' THEN excluded.l2_tx_hash
    ELSE via_withdrawals.l2_tx_hash
END,
updated_at = NOW();
```

Note: There is no `signing_sessions` or `verifier_registry` table. MuSig2 signing is coordinated in-memory by the coordinator service. Verifier registration is tracked via Bitcoin governance inscriptions and the `via_wallets` table, not a dedicated registry table.

## L1 Indexer Database Schema

> **CORRECTION (2026-04-11):** The previous version of this section contained entirely fabricated table definitions
> (deposits with vout/confirmation tracking, bridge_withdrawals with JSONB signatures, withdrawals with proof_hash,
> a key-value indexer_metadata, and a utxos tracking table). None of those tables exist in the codebase.
> The actual schemas below are from `via_indexer/lib/via_indexer_dal/migrations/`.

The L1 Indexer maintains a separate database (`via_indexer_dal`) with three migrations and four tables. DAL crate: `via_indexer/lib/via_indexer_dal/src/`.

### Deposits Table

Tracks Bitcoin-to-L2 deposits. DAL: `via_indexer/lib/via_indexer_dal/src/via_transactions_dal.rs`.

```sql
-- via_indexer/lib/via_indexer_dal/migrations/20250604191948_deposit_withdraw.up.sql
CREATE TABLE IF NOT EXISTS deposits (
    "priority_id" BIGINT NOT NULL,
    "tx_id" BYTEA NOT NULL,
    "block_number" BIGINT NOT NULL,
    "sender" VARCHAR NOT NULL,
    "receiver" VARCHAR NOT NULL,
    "value" BIGINT NOT NULL,
    "calldata" BYTEA,
    "canonical_tx_hash" BYTEA NOT NULL UNIQUE,
    "created_at" BIGINT NOT NULL,
    PRIMARY KEY (tx_id)
);
```

Query excerpt from `via_indexer/lib/via_indexer_dal/src/via_transactions_dal.rs`:

```sql
INSERT INTO
deposits (
    priority_id,
    tx_id,
    block_number,
    sender,
    receiver,
    value,
    calldata,
    canonical_tx_hash,
    created_at
)
VALUES
($1, $2, $3, $4, $5, $6, $7, $8, $9)
ON CONFLICT (tx_id) DO NOTHING
```

### Withdrawals Table

Tracks L2-to-Bitcoin withdrawal records. DAL: `via_indexer/lib/via_indexer_dal/src/via_transactions_dal.rs`.

```sql
-- via_indexer/lib/via_indexer_dal/migrations/20250604191948_deposit_withdraw.up.sql
CREATE TABLE IF NOT EXISTS withdrawals (
    "id" VARCHAR UNIQUE NOT NULL,
    "tx_id" BYTEA NOT NULL,
    "l2_tx_log_index" BIGINT NOT NULL,
    "receiver" VARCHAR NOT NULL,
    "value" BIGINT NOT NULL,
    "block_number" BIGINT NOT NULL,
    "timestamp" BIGINT NOT NULL,
    "created_at" TIMESTAMP NOT NULL DEFAULT NOW()
);
```

### Indexer Metadata Table

Tracks the indexer cursor per module. DAL: `via_indexer/lib/via_indexer_dal/src/via_indexer_dal.rs`.

```sql
-- via_indexer/lib/via_indexer_dal/migrations/20250604192021_indexer_metadata.up.sql
CREATE TABLE IF NOT EXISTS indexer_metadata (
    module VARCHAR NOT NULL UNIQUE,
    last_indexer_l1_block BIGINT NOT NULL,
    updated_at TIMESTAMP NOT NULL
);
```

### Indexer Wallets Table

Same shape as the `via_wallets` tables in the other two databases. DAL: `via_indexer/lib/via_indexer_dal/src/via_wallet_dal.rs`.

```sql
-- via_indexer/lib/via_indexer_dal/migrations/20250731184639_via_wallet_migration.up.sql
CREATE TABLE via_wallets (
    id BIGSERIAL PRIMARY KEY,
    role VARCHAR NOT NULL,
    address VARCHAR NOT NULL,
    tx_hash VARCHAR NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT unique_tx_hash_address_role UNIQUE (tx_hash, address, role)
);

CREATE INDEX idx_via_wallets_role_created_at ON via_wallets(role, created_at DESC);
```

Note: There is no UTXO tracking table in any of the databases. Bridge UTXO management is handled in-memory by the `UtxoManager` in `via_verifier/lib/via_musig2/src/utxo_manager.rs`, which fetches UTXOs from the Bitcoin node via RPC on each iteration.

## Operational notes

> **CORRECTION (2026-07-02):** The previous version of this document ended with "Recommended Indexes",
> "Partitioning Strategy", "Maintenance Procedures", "Backup and Recovery", and "Monitoring" sections
> containing invented index names (for example `idx_transactions_hash`, `idx_events_address_topic1`),
> invented database names, and generic PostgreSQL advice not grounded in this repo. Those sections
> have been removed. Indexes that actually exist are shown inline with their tables above, in the
> migration that creates them.
