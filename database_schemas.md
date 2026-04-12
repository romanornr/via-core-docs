# Via Core Database Schemas Documentation

## Overview

The Via Core system uses multiple database schemas to manage different aspects of the L2 rollup system. This document provides comprehensive documentation for all database schemas used across the Via Core components.

## Core System Database Schema

The main Via Core database contains tables for managing L2 operations, batches, transactions, and system state.

### L1 Batches Table

Stores information about L1 batches that have been committed to Bitcoin.

```sql
CREATE TABLE l1_batches (
    number BIGINT PRIMARY KEY,
    l1_tx_count INTEGER NOT NULL,
    l2_tx_count INTEGER NOT NULL,
    timestamp BIGINT NOT NULL,
    l2_to_l1_logs BYTEA[],
    l2_to_l1_messages BYTEA[],
    bloom BYTEA NOT NULL,
    priority_ops_onchain_data BYTEA[],
    hash BYTEA,
    parent_hash BYTEA,
    commitment BYTEA,
    compressed_write_logs BYTEA,
    compressed_contracts BYTEA,
    eth_prove_tx_id INTEGER,
    eth_commit_tx_id INTEGER,
    eth_execute_tx_id INTEGER,
    merkle_root_hash BYTEA,
    l2_to_l1_logs_tree_size INTEGER,
    skip_proof BOOLEAN NOT NULL DEFAULT FALSE,
    aux_data_hash BYTEA,
    meta_parameters_hash BYTEA,
    protocol_version INTEGER,
    compressed_state_diffs BYTEA,
    system_logs BYTEA[] NOT NULL DEFAULT '{}',
    compressed_initial_writes BYTEA,
    l2_l1_compressed_messages BYTEA,
    l2_l1_merkle_root BYTEA,
    rollup_last_leaf_index BIGINT,
    zkporter_is_available BOOLEAN,
    bootloader_code_hash BYTEA,
    default_aa_code_hash BYTEA,
    base_fee_per_gas NUMERIC(80) NOT NULL,
    l1_gas_price BIGINT NOT NULL,
    l2_fair_gas_price BIGINT NOT NULL,
    base_system_contracts_hashes JSONB NOT NULL,
    system_logs_hash_tree_root BYTEA,
    events_queue_commitment BYTEA,
    bootloader_initial_content_commitment BYTEA,
    pubdata_input BYTEA
);
```

### Transactions Table

Stores L2 transactions and their execution details.

```sql
CREATE TABLE transactions (
    hash BYTEA PRIMARY KEY,
    is_priority BOOLEAN NOT NULL,
    initiator_address BYTEA NOT NULL,
    nonce BIGINT,
    signature BYTEA,
    gas_limit NUMERIC(80),
    max_fee_per_gas NUMERIC(80),
    max_priority_fee_per_gas NUMERIC(80),
    gas_per_pubdata_limit NUMERIC(80),
    input BYTEA,
    data JSONB NOT NULL,
    received_at TIMESTAMP NOT NULL,
    in_mempool BOOLEAN NOT NULL DEFAULT TRUE,
    l1_batch_number BIGINT REFERENCES l1_batches(number),
    l1_batch_tx_index INTEGER,
    miniblock_number BIGINT,
    index_in_block INTEGER,
    error TEXT,
    effective_gas_price NUMERIC(80),
    contract_address BYTEA,
    l1_tx_mint NUMERIC(80),
    l1_tx_refund_recipient BYTEA,
    upgrade_id INTEGER,
    execution_info JSONB NOT NULL DEFAULT '{}'::jsonb,
    refunded_gas BIGINT NOT NULL DEFAULT 0,
    gas_used_for_tx BIGINT
);
```

### Storage Logs Table

Stores storage changes for smart contracts.

```sql
CREATE TABLE storage_logs (
    hashed_key BYTEA NOT NULL,
    address BYTEA NOT NULL,
    key BYTEA NOT NULL,
    value BYTEA NOT NULL,
    operation_number INTEGER NOT NULL,
    tx_hash BYTEA NOT NULL REFERENCES transactions(hash),
    miniblock_number BIGINT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    PRIMARY KEY (hashed_key, miniblock_number)
);
```

### Events Table

Stores smart contract events emitted during transaction execution.

```sql
CREATE TABLE events (
    miniblock_number BIGINT NOT NULL,
    tx_hash BYTEA NOT NULL REFERENCES transactions(hash),
    tx_index_in_block INTEGER NOT NULL,
    address BYTEA NOT NULL,
    event_index_in_block INTEGER NOT NULL,
    event_index_in_tx INTEGER NOT NULL,
    topic1 BYTEA NOT NULL,
    topic2 BYTEA,
    topic3 BYTEA,
    topic4 BYTEA,
    value BYTEA NOT NULL,
    tx_initiator_address BYTEA,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    PRIMARY KEY (miniblock_number, event_index_in_block)
);
```

## L1 Indexer Database Schema

> **CORRECTION (2026-04-11):** The previous version of this section contained entirely fabricated table definitions
> (deposits with vout/confirmation tracking, bridge_withdrawals with JSONB signatures, withdrawals with proof_hash,
> a key-value indexer_metadata, and a utxos tracking table). None of those tables exist in the codebase.
> The actual schemas below are from `via_indexer/lib/via_indexer_dal/migrations/`.

The L1 Indexer maintains a **separate database** (`via_indexer_dal`) for tracking Bitcoin L1 deposits and withdrawals.

### Deposits Table

Tracks Bitcoin-to-L2 deposits.

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

### Withdrawals Table

Tracks L2-to-Bitcoin withdrawal records.

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

Tracks indexer cursor per module.

```sql
-- via_indexer/lib/via_indexer_dal/migrations/20250604192021_indexer_metadata.up.sql
CREATE TABLE IF NOT EXISTS indexer_metadata (
    module VARCHAR NOT NULL UNIQUE,
    last_indexer_l1_block BIGINT NOT NULL,
    updated_at TIMESTAMP NOT NULL
);
```

Note: There is no UTXO tracking table in the database. Bridge UTXO management is handled in-memory by the `UtxoManager` in `via_verifier/lib/via_musig2/src/utxo_manager.rs`, which fetches UTXOs from the Bitcoin node via RPC on each iteration.

## Verifier Database Schema

> **CORRECTION (2026-04-11):** The previous version fabricated `verifier_votes`, `signing_sessions`, and
> `verifier_registry` tables. None of those exist. The actual verifier database schema is in
> `via_verifier/lib/verifier_dal/migrations/`. MuSig2 signing sessions are managed in-memory
> by the coordinator, not persisted in a database table.

The verifier uses a **separate database** (`verifier_dal`) with its own migration history.

### Votable Transactions Table

Tracks L1 batches that verifiers need to vote on.

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

### Verifier Votes Table

Stores per-verifier votes referencing votable transactions.

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

ALTER TABLE via_votes ADD FOREIGN KEY (votable_transaction_id)
    REFERENCES via_votable_transactions (id) ON DELETE CASCADE ON UPDATE NO ACTION;
```

### Verifier Transactions Table

Tracks priority transactions observed by the verifier.

```sql
-- via_verifier/lib/verifier_dal/migrations/20250207164238_via_add_transactions_dal.up.sql
CREATE TABLE IF NOT EXISTS via_transactions (
    "priority_id" BIGINT NOT NULL,
    "tx_id" BYTEA NOT NULL,
    "receiver" VARCHAR NOT NULL,
    "value" BIGINT NOT NULL,
    "calldata" BYTEA,
    "canonical_tx_hash" BYTEA NOT NULL,
    "status" BOOLEAN,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    PRIMARY KEY (tx_id)
);
```

Note: There is no `signing_sessions` or `verifier_registry` table. MuSig2 signing is coordinated in-memory by the coordinator service. Verifier registration is tracked via Bitcoin governance inscriptions and the `via_wallets` table, not a dedicated registry table.

## Protocol Version Management Schema

Tables for managing protocol versions and upgrades.

### Protocol Versions Table

Stores all protocol versions and their execution status.

```sql
CREATE TABLE protocol_versions (
    id INTEGER PRIMARY KEY,
    timestamp BIGINT NOT NULL,
    recursion_scheduler_level_vk_hash BYTEA NOT NULL,
    recursion_node_level_vk_hash BYTEA NOT NULL,
    recursion_leaf_level_vk_hash BYTEA NOT NULL,
    recursion_circuits_set_vks_hash BYTEA NOT NULL,
    bootloader_code_hash BYTEA NOT NULL,
    default_account_code_hash BYTEA NOT NULL,
    verifier_address BYTEA NOT NULL,
    upgrade_tx_hash BYTEA,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

### Protocol Patches Table

Stores protocol patches and their application status.

```sql
CREATE TABLE protocol_patches (
    minor_version INTEGER NOT NULL,
    patch INTEGER NOT NULL,
    snark_wrapper_vk_hash BYTEA NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    PRIMARY KEY (minor_version, patch)
);
```

## Database Maintenance and Optimization

### Recommended Indexes

Additional indexes for performance optimization:

```sql
-- Transaction hash lookups
CREATE INDEX CONCURRENTLY idx_transactions_hash_partial ON transactions(hash) WHERE in_mempool = FALSE;

-- L1 batch queries
CREATE INDEX CONCURRENTLY idx_l1_batches_timestamp ON l1_batches(timestamp);
CREATE INDEX CONCURRENTLY idx_l1_batches_protocol_version ON l1_batches(protocol_version);

-- Event queries
CREATE INDEX CONCURRENTLY idx_events_address_topic1 ON events(address, topic1);
CREATE INDEX CONCURRENTLY idx_events_tx_hash ON events(tx_hash);

-- Storage log queries
CREATE INDEX CONCURRENTLY idx_storage_logs_address_key ON storage_logs(address, key);
```

### Partitioning Strategy

For high-volume tables, consider partitioning:

```sql
-- Partition events table by miniblock_number
CREATE TABLE events_partitioned (
    LIKE events INCLUDING ALL
) PARTITION BY RANGE (miniblock_number);

-- Create monthly partitions
CREATE TABLE events_y2024m01 PARTITION OF events_partitioned
    FOR VALUES FROM (1000000) TO (2000000);
```

### Maintenance Procedures

Regular maintenance procedures for optimal performance:

```sql
-- Vacuum and analyze tables
VACUUM ANALYZE transactions;
VACUUM ANALYZE l1_batches;
VACUUM ANALYZE events;
VACUUM ANALYZE storage_logs;

-- Update table statistics
ANALYZE transactions;
ANALYZE l1_batches;
ANALYZE via_data_availability;
ANALYZE via_votes;

-- Reindex if necessary
REINDEX INDEX CONCURRENTLY idx_transactions_hash;
REINDEX INDEX CONCURRENTLY idx_events_address_topic1;
```

## Backup and Recovery

### Backup Strategy

```bash
# Full database backup
pg_dump -h localhost -U postgres -d via_core > via_core_backup.sql

# L1 Indexer database backup
pg_dump -h localhost -U postgres -d via_l1_indexer > via_l1_indexer_backup.sql

# Compressed backup with custom format
pg_dump -h localhost -U postgres -d via_core -Fc > via_core_backup.dump
```

### Point-in-Time Recovery

```bash
# Enable WAL archiving in postgresql.conf
archive_mode = on
archive_command = 'cp %p /path/to/archive/%f'

# Create base backup
pg_basebackup -h localhost -U postgres -D /path/to/backup -Ft -z -P

# Restore to specific point in time
pg_ctl stop -D /path/to/data
rm -rf /path/to/data/*
tar -xzf /path/to/backup/base.tar.gz -C /path/to/data
# Configure recovery.conf and restart
```

## Monitoring and Metrics

### Database Health Queries

```sql
-- Check database size
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size
FROM pg_tables 
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- Check index usage
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- Check slow queries
SELECT 
    query,
    calls,
    total_time,
    mean_time,
    rows
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;
```

### Performance Metrics

Key metrics to monitor:

- **Transaction throughput**: Transactions per second
- **L1 batch processing time**: Time to process and commit batches
- **Indexer lag**: Blocks behind Bitcoin tip
- **Database connection pool usage**: Active vs. available connections
- **Query performance**: Slow query identification and optimization
- **Storage growth**: Database size growth over time
- **Replication lag**: If using read replicas

This comprehensive database schema documentation provides the foundation for understanding and maintaining the Via Core database systems across all components.