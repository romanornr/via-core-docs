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

The L1 Indexer maintains a separate database for tracking Bitcoin L1 transactions and bridge operations.

### Deposits Table

Tracks Bitcoin deposits to the bridge address.

```sql
CREATE TABLE deposits (
    id SERIAL PRIMARY KEY,
    txid VARCHAR(64) NOT NULL UNIQUE,
    vout INTEGER NOT NULL,
    amount BIGINT NOT NULL,
    address VARCHAR(100) NOT NULL,
    l2_receiver_address BYTEA,
    block_height INTEGER,
    block_hash VARCHAR(64),
    confirmed BOOLEAN DEFAULT FALSE,
    processed BOOLEAN DEFAULT FALSE,
    validation_status VARCHAR(20) DEFAULT 'pending',
    validation_error TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    CONSTRAINT unique_deposit UNIQUE (txid, vout)
);

-- Indexes for efficient querying
CREATE INDEX idx_deposits_block_height ON deposits(block_height);
CREATE INDEX idx_deposits_confirmed ON deposits(confirmed);
CREATE INDEX idx_deposits_processed ON deposits(processed);
CREATE INDEX idx_deposits_address ON deposits(address);
CREATE INDEX idx_deposits_l2_receiver ON deposits(l2_receiver_address);
```

### Bridge Withdrawals Table

Tracks withdrawal transactions processed by the bridge.

```sql
CREATE TABLE bridge_withdrawals (
    id SERIAL PRIMARY KEY,
    txid VARCHAR(64) NOT NULL UNIQUE,
    inscription_data TEXT,
    inscription_content_type VARCHAR(50),
    withdrawal_requests JSONB,
    input_count INTEGER NOT NULL DEFAULT 1,
    output_count INTEGER NOT NULL,
    total_amount BIGINT NOT NULL,
    fee_amount BIGINT,
    block_height INTEGER,
    block_hash VARCHAR(64),
    confirmed BOOLEAN DEFAULT FALSE,
    processed BOOLEAN DEFAULT FALSE,
    musig2_signature BYTEA,
    verifier_signatures JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Indexes for efficient querying
CREATE INDEX idx_bridge_withdrawals_block_height ON bridge_withdrawals(block_height);
CREATE INDEX idx_bridge_withdrawals_confirmed ON bridge_withdrawals(confirmed);
CREATE INDEX idx_bridge_withdrawals_processed ON bridge_withdrawals(processed);
CREATE INDEX idx_bridge_withdrawals_inscription ON bridge_withdrawals USING gin(inscription_data gin_trgm_ops);
```

### Withdrawals Table

Tracks individual withdrawal requests within bridge transactions.

```sql
CREATE TABLE withdrawals (
    id SERIAL PRIMARY KEY,
    bridge_withdrawal_id INTEGER REFERENCES bridge_withdrawals(id),
    txid VARCHAR(64) NOT NULL,
    vout INTEGER NOT NULL,
    amount BIGINT NOT NULL,
    recipient_address VARCHAR(100) NOT NULL,
    l2_batch_number BIGINT,
    withdrawal_index INTEGER,
    proof_hash BYTEA,
    block_height INTEGER,
    confirmed BOOLEAN DEFAULT FALSE,
    processed BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    CONSTRAINT unique_withdrawal UNIQUE (txid, vout)
);

-- Indexes for efficient querying
CREATE INDEX idx_withdrawals_bridge_withdrawal ON withdrawals(bridge_withdrawal_id);
CREATE INDEX idx_withdrawals_recipient ON withdrawals(recipient_address);
CREATE INDEX idx_withdrawals_l2_batch ON withdrawals(l2_batch_number);
CREATE INDEX idx_withdrawals_block_height ON withdrawals(block_height);
```

### Indexer Metadata Table

Stores indexer state and configuration metadata.

```sql
CREATE TABLE indexer_metadata (
    key VARCHAR(50) PRIMARY KEY,
    value TEXT NOT NULL,
    value_type VARCHAR(20) DEFAULT 'string',
    description TEXT,
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Insert default metadata
INSERT INTO indexer_metadata (key, value, value_type, description) VALUES
('last_processed_block', '0', 'integer', 'Last Bitcoin block height processed by the indexer'),
('indexer_version', '1.0.0', 'string', 'Version of the L1 indexer'),
('start_block_height', '0', 'integer', 'Starting block height for indexing'),
('confirmation_blocks', '6', 'integer', 'Number of confirmation blocks required'),
('bridge_address', '', 'string', 'Bitcoin bridge address being monitored'),
('network', 'regtest', 'string', 'Bitcoin network (mainnet/testnet/regtest)'),
('indexer_status', 'stopped', 'string', 'Current status of the indexer'),
('last_error', '', 'string', 'Last error encountered by the indexer'),
('total_deposits_processed', '0', 'integer', 'Total number of deposits processed'),
('total_withdrawals_processed', '0', 'integer', 'Total number of withdrawals processed');
```

### UTXO Tracking Table

Tracks available UTXOs for bridge operations.

```sql
CREATE TABLE utxos (
    id SERIAL PRIMARY KEY,
    txid VARCHAR(64) NOT NULL,
    vout INTEGER NOT NULL,
    amount BIGINT NOT NULL,
    script_pubkey BYTEA NOT NULL,
    address VARCHAR(100),
    block_height INTEGER NOT NULL,
    spent BOOLEAN DEFAULT FALSE,
    spent_in_txid VARCHAR(64),
    spent_in_vin INTEGER,
    spent_block_height INTEGER,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    CONSTRAINT unique_utxo UNIQUE (txid, vout)
);

-- Indexes for efficient UTXO selection
CREATE INDEX idx_utxos_unspent ON utxos(spent) WHERE spent = FALSE;
CREATE INDEX idx_utxos_amount ON utxos(amount) WHERE spent = FALSE;
CREATE INDEX idx_utxos_address ON utxos(address) WHERE spent = FALSE;
CREATE INDEX idx_utxos_block_height ON utxos(block_height);
```

## Verifier Database Schema

The verifier system uses additional tables for managing verification processes and voting.

### Verifier Votes Table

Stores votes from verifiers on L1 batches.

```sql
CREATE TABLE verifier_votes (
    id SERIAL PRIMARY KEY,
    l1_batch_number BIGINT NOT NULL,
    verifier_address BYTEA NOT NULL,
    vote_type VARCHAR(20) NOT NULL, -- 'approve', 'reject'
    signature BYTEA NOT NULL,
    timestamp TIMESTAMP NOT NULL DEFAULT NOW(),
    block_height BIGINT,
    CONSTRAINT unique_verifier_vote UNIQUE (l1_batch_number, verifier_address)
);

-- Indexes for efficient querying
CREATE INDEX idx_verifier_votes_batch ON verifier_votes(l1_batch_number);
CREATE INDEX idx_verifier_votes_verifier ON verifier_votes(verifier_address);
CREATE INDEX idx_verifier_votes_type ON verifier_votes(vote_type);
```

### Signing Sessions Table

Tracks MuSig2 signing sessions for withdrawal transactions.

```sql
CREATE TABLE signing_sessions (
    id SERIAL PRIMARY KEY,
    session_id VARCHAR(64) NOT NULL UNIQUE,
    unsigned_tx_hash BYTEA NOT NULL,
    withdrawal_batch_id INTEGER,
    coordinator_address BYTEA NOT NULL,
    required_signers INTEGER NOT NULL,
    current_signers INTEGER DEFAULT 0,
    status VARCHAR(20) DEFAULT 'pending', -- 'pending', 'signing', 'completed', 'failed'
    nonce_commitments JSONB,
    partial_signatures JSONB,
    final_signature BYTEA,
    signed_tx_hash BYTEA,
    broadcast_txid VARCHAR(64),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP
);

-- Indexes for efficient querying
CREATE INDEX idx_signing_sessions_status ON signing_sessions(status);
CREATE INDEX idx_signing_sessions_coordinator ON signing_sessions(coordinator_address);
CREATE INDEX idx_signing_sessions_expires ON signing_sessions(expires_at);
```

### Verifier Registry Table

Maintains the registry of active verifiers and their public keys.

```sql
CREATE TABLE verifier_registry (
    id SERIAL PRIMARY KEY,
    verifier_address BYTEA NOT NULL UNIQUE,
    public_key BYTEA NOT NULL,
    bitcoin_address VARCHAR(100) NOT NULL,
    status VARCHAR(20) DEFAULT 'active', -- 'active', 'inactive', 'slashed'
    stake_amount NUMERIC(80),
    registration_block BIGINT,
    last_activity TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Indexes for efficient querying
CREATE INDEX idx_verifier_registry_status ON verifier_registry(status);
CREATE INDEX idx_verifier_registry_bitcoin_address ON verifier_registry(bitcoin_address);
```

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
ANALYZE deposits;
ANALYZE bridge_withdrawals;

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