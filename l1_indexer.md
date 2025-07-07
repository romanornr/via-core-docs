# Via L2 Bitcoin L1 Indexer System

## Overview

The Via L1 Indexer is a specialized Bitcoin blockchain indexing service that complements the existing L1 Watcher by providing dedicated indexing capabilities for deposits and withdrawals. This system was introduced to enhance the Via L2 architecture with more efficient and scalable Bitcoin transaction processing.

### Key Features

- **Dedicated Bitcoin L1 Indexing**: Specialized service for indexing Bitcoin transactions relevant to Via L2
- **Deposit/Withdrawal Tracking**: Comprehensive tracking of bridge deposits and withdrawals
- **Independent Database Schema**: Dedicated database tables optimized for indexing operations
- **Message Processing Pipeline**: Structured processing of Bitcoin inscription messages
- **Scalable Architecture**: Designed for high-throughput Bitcoin transaction processing

### Integration with Via Architecture

The L1 Indexer integrates seamlessly with the existing Via L2 system:

```mermaid
graph TB
    subgraph "Bitcoin L1"
        BTC[Bitcoin Blockchain]
        Inscriptions[Bitcoin Inscriptions]
    end
    
    subgraph "Via L2 Indexing Layer"
        L1Watcher[L1 Watcher]
        L1Indexer[L1 Indexer]
        IndexerDB[(Indexer Database)]
    end
    
    subgraph "Via L2 Core"
        Bridge[Bridge System]
        Sequencer[Sequencer]
        StateKeeper[State Keeper]
    end
    
    BTC --> L1Watcher
    BTC --> L1Indexer
    Inscriptions --> L1Indexer
    L1Indexer --> IndexerDB
    L1Indexer --> Bridge
    L1Watcher --> Bridge
    Bridge --> Sequencer
    Sequencer --> StateKeeper
```

## Architecture

### System Components

The L1 Indexer consists of several key components working together:

#### 1. Indexer Service (`via_indexer`)

The main indexer service responsible for:
- Bitcoin blockchain monitoring
- Transaction parsing and validation
- Message processing coordination
- Database operations management

#### 2. Data Access Layer (`via_indexer_dal`)

Provides database abstraction with:
- **ViaIndexerDal**: Metadata and state management
- **ViaTransactionsDal**: Transaction data operations
- **Connection Management**: Database connection pooling
- **Migration Support**: Schema versioning and updates

#### 3. Message Processors

Specialized processors for different message types:
- **Deposit Processor**: Handles L1→L2 deposit transactions
- **Withdrawal Processor**: Processes L2→L1 withdrawal transactions
- **Bridge Withdrawal Processor**: Manages bridge withdrawal inscriptions

### Component Interaction

```mermaid
sequenceDiagram
    participant BTC as Bitcoin Node
    participant IDX as L1 Indexer
    participant MP as Message Processors
    participant DB as Indexer Database
    participant BRIDGE as Bridge System

    BTC->>IDX: New Block Notification
    IDX->>BTC: Fetch Block Data
    BTC->>IDX: Block Transactions
    IDX->>MP: Process Transactions
    MP->>MP: Parse Messages
    MP->>DB: Store Deposits/Withdrawals
    MP->>BRIDGE: Notify Bridge System
    IDX->>DB: Update Metadata
```

## Database Schema

The L1 Indexer introduces a dedicated database schema optimized for indexing operations:

### Core Tables

#### 1. Deposits Table

Stores L1→L2 deposit transactions:

```sql
CREATE TABLE IF NOT EXISTS deposits (
    "priority_id" BIGINT NOT NULL,
    "tx_id" BYTEA NOT NULL,
    "block_number" BIGINT NOT NULL,
    "receiver" BYTEA NOT NULL,
    "value" BIGINT NOT NULL,
    "calldata" BYTEA,
    "canonical_tx_hash" BYTEA NOT NULL UNIQUE,
    "created_at" TIMESTAMP NOT NULL DEFAULT NOW(),
    PRIMARY KEY (tx_id)
);
```

**Field Descriptions:**
- `priority_id`: Sequential identifier for deposit ordering
- `tx_id`: Bitcoin transaction ID (32 bytes)
- `block_number`: Bitcoin block number containing the transaction
- `receiver`: L2 receiver address (20 bytes)
- `value`: Deposit amount in satoshis
- `calldata`: Optional transaction calldata
- `canonical_tx_hash`: Unique hash for L2 transaction
- `created_at`: Timestamp of record creation

#### 2. Bridge Withdrawals Table

Tracks bridge withdrawal transactions:

```sql
CREATE TABLE IF NOT EXISTS bridge_withdrawals (
    "id" SERIAL PRIMARY KEY,
    "tx_id" BYTEA NOT NULL UNIQUE,
    "l1_batch_reveal_tx_id" BYTEA NOT NULL,
    "fee" BIGINT NOT NULL,
    "vsize" BIGINT NOT NULL,
    "total_size" BIGINT NOT NULL,
    "withdrawals_count" BIGINT NOT NULL,
    "block_number" BIGINT NOT NULL UNIQUE,
    "created_at" TIMESTAMP NOT NULL DEFAULT NOW()
);
```

**Field Descriptions:**
- `id`: Auto-incrementing primary key
- `tx_id`: Bitcoin transaction ID
- `l1_batch_reveal_tx_id`: Associated L1 batch proof transaction
- `fee`: Transaction fee in satoshis
- `vsize`: Virtual transaction size
- `total_size`: Total transaction size in bytes
- `withdrawals_count`: Number of withdrawals in transaction
- `block_number`: Bitcoin block number
- `created_at`: Record creation timestamp

#### 3. Withdrawals Table

Individual withdrawal details:

```sql
CREATE TABLE IF NOT EXISTS withdrawals (
    "id" SERIAL PRIMARY KEY,
    "bridge_withdrawal_id" INTEGER NOT NULL,
    "tx_index" BIGINT NOT NULL,
    "receiver" VARCHAR NOT NULL,
    "value" BIGINT NOT NULL,
    "created_at" TIMESTAMP NOT NULL DEFAULT NOW(),
    FOREIGN KEY (bridge_withdrawal_id) REFERENCES bridge_withdrawals (id) ON DELETE CASCADE
);
```

**Field Descriptions:**
- `id`: Auto-incrementing primary key
- `bridge_withdrawal_id`: Foreign key to bridge_withdrawals table
- `tx_index`: Index of withdrawal within transaction
- `receiver`: Bitcoin address receiving the withdrawal
- `value`: Withdrawal amount in satoshis
- `created_at`: Record creation timestamp

#### 4. Indexer Metadata Table

Tracks indexer state and progress:

```sql
CREATE TABLE IF NOT EXISTS indexer_metadata (
    module VARCHAR NOT NULL UNIQUE,
    last_indexer_l1_block BIGINT NOT NULL,
    updated_at TIMESTAMP NOT NULL
);
```

**Field Descriptions:**
- `module`: Indexer module name (e.g., "deposit", "withdrawal")
- `last_indexer_l1_block`: Last processed Bitcoin block number
- `updated_at`: Last update timestamp

### Database Relationships

```mermaid
erDiagram
    DEPOSITS {
        bigint priority_id PK
        bytea tx_id
        bigint block_number
        bytea receiver
        bigint value
        bytea calldata
        bytea canonical_tx_hash UK
        timestamp created_at
    }
    
    BRIDGE_WITHDRAWALS {
        serial id PK
        bytea tx_id UK
        bytea l1_batch_reveal_tx_id
        bigint fee
        bigint vsize
        bigint total_size
        bigint withdrawals_count
        bigint block_number UK
        timestamp created_at
    }
    
    WITHDRAWALS {
        serial id PK
        integer bridge_withdrawal_id FK
        bigint tx_index
        varchar receiver
        bigint value
        timestamp created_at
    }
    
    INDEXER_METADATA {
        varchar module PK
        bigint last_indexer_l1_block
        timestamp updated_at
    }
    
    BRIDGE_WITHDRAWALS ||--o{ WITHDRAWALS : contains
```

## API Reference

### ViaIndexerDal

Core indexer metadata operations:

#### Initialize Indexer Metadata

```rust
pub async fn init_indexer_metadata(&mut self, module: &str, l1_block: u32) -> DalResult<()>
```

Initializes metadata for a specific indexer module.

**Parameters:**
- `module`: Module name (e.g., "deposit", "withdrawal")
- `l1_block`: Starting Bitcoin block number

**Example:**
```rust
let mut dal = connection.via_indexer_dal();
dal.init_indexer_metadata("deposit", 800000).await?;
```

#### Update Last Processed Block

```rust
pub async fn update_last_processed_l1_block(&mut self, module: &str, l1_block: u32) -> DalResult<()>
```

Updates the last processed block for a module.

**Parameters:**
- `module`: Module name
- `l1_block`: Last processed block number

#### Get Last Processed Block

```rust
pub async fn get_last_processed_l1_block(&mut self, module: &str) -> DalResult<u64>
```

Retrieves the last processed block number for a module.

**Returns:** Last processed block number (0 if not found)

### ViaTransactionsDal

Transaction data operations:

#### Insert Deposits

```rust
pub async fn insert_deposit_many(&mut self, deposits: Vec<Deposit>) -> DalResult<()>
```

Inserts multiple deposit records in a single transaction.

**Parameters:**
- `deposits`: Vector of deposit records

**Example:**
```rust
let deposits = vec![
    Deposit {
        priority_id: 1,
        tx_id: tx_id_bytes,
        block_number: 800001,
        receiver: receiver_bytes,
        value: 100000000, // 1 BTC in satoshis
        calldata: vec![],
        canonical_tx_hash: hash_bytes,
    }
];

let mut dal = connection.via_transactions_dal();
dal.insert_deposit_many(deposits).await?;
```

#### Check Deposit Existence

```rust
pub async fn deposit_exists(&mut self, tx_id: &[u8]) -> DalResult<bool>
```

Checks if a deposit with the given transaction ID exists.

#### Insert Withdrawal

```rust
pub async fn insert_withdraw(
    &mut self,
    bridge_withdrawal_param: BridgeWithdrawalParam,
    withdrawals: Vec<WithdrawalParam>,
) -> DalResult<()>
```

Inserts a bridge withdrawal with associated individual withdrawals.

**Example:**
```rust
let bridge_withdrawal = BridgeWithdrawalParam {
    tx_id: tx_id_bytes,
    l1_batch_reveal_tx_id: batch_tx_id,
    block_number: 800002,
    fee: 5000,
    vsize: 250,
    total_size: 300,
    withdrawals_count: 2,
};

let withdrawals = vec![
    WithdrawalParam {
        tx_index: 0,
        receiver: "bc1qxy2kgdygjrsqtzq2n0yrf2493p83kkfjhx0wlh".to_string(),
        value: 50000000,
    },
    WithdrawalParam {
        tx_index: 1,
        receiver: "bc1qw508d6qejxtdg4y5r3zarvary0c5xw7kv8f3t4".to_string(),
        value: 45000000,
    },
];

dal.insert_withdraw(bridge_withdrawal, withdrawals).await?;
```

## Message Processing

### Deposit Processing

The deposit processor handles L1→L2 transactions:

```rust
pub struct DepositProcessor {
    pool: ConnectionPool<Indexer>,
}

impl DepositProcessor {
    pub async fn process_l1_to_l2_message(
        &self,
        message: &L1ToL2Message,
        block_number: u32,
    ) -> anyhow::Result<()> {
        // Extract deposit information
        let deposit = Deposit {
            priority_id: self.get_next_priority_id().await?,
            tx_id: message.common.tx_id.to_vec(),
            block_number,
            receiver: message.input.receiver_l2_address.as_bytes().to_vec(),
            value: message.input.value as i64,
            calldata: message.input.calldata.clone(),
            canonical_tx_hash: self.calculate_canonical_hash(&message).await?,
        };

        // Store deposit
        let mut connection = self.pool.connection().await?;
        connection.via_transactions_dal()
            .insert_deposit_many(vec![deposit])
            .await?;

        Ok(())
    }
}
```

### Withdrawal Processing

The withdrawal processor handles L2→L1 transactions:

```rust
pub struct WithdrawalProcessor {
    pool: ConnectionPool<Indexer>,
}

impl WithdrawalProcessor {
    pub async fn process_bridge_withdrawal(
        &self,
        message: &BridgeWithdrawal,
        block_number: u32,
    ) -> anyhow::Result<()> {
        // Extract bridge withdrawal information
        let bridge_withdrawal = BridgeWithdrawalParam {
            tx_id: message.common.tx_id.to_vec(),
            l1_batch_reveal_tx_id: message.input.l1_batch_proof_reveal_tx_id.clone(),
            block_number: block_number as i64,
            fee: self.calculate_fee(&message.input)?,
            vsize: message.input.v_size,
            total_size: message.input.total_size,
            withdrawals_count: message.input.withdrawals.len() as i64,
        };

        // Extract individual withdrawals
        let withdrawals: Vec<WithdrawalParam> = message.input.withdrawals
            .iter()
            .enumerate()
            .map(|(index, (receiver, value))| WithdrawalParam {
                tx_index: index as i64,
                receiver: receiver.clone(),
                value: *value,
            })
            .collect();

        // Store withdrawal data
        let mut connection = self.pool.connection().await?;
        connection.via_transactions_dal()
            .insert_withdraw(bridge_withdrawal, withdrawals)
            .await?;

        Ok(())
    }
}
```

## Configuration

### Environment Variables

The L1 Indexer requires the following configuration:

#### Database Configuration

```toml
# Database connection for indexer
DATABASE_INDEXER_URL="postgresql://user:password@localhost:5432/via_indexer"
```

#### Bitcoin Client Configuration

```toml
# Bitcoin RPC configuration
VIA_BTC_CLIENT_RPC_URL="http://localhost:8332"
VIA_BTC_CLIENT_RPC_USER="rpcuser"
VIA_BTC_CLIENT_RPC_PASSWORD="rpcpassword"
VIA_BTC_CLIENT_NETWORK="regtest"
```

#### Indexer-Specific Settings

```toml
# Starting block for indexing (optional)
VIA_INDEXER_START_BLOCK=800000

# Batch size for processing transactions
VIA_INDEXER_BATCH_SIZE=100

# Polling interval in seconds
VIA_INDEXER_POLL_INTERVAL=10
```

### Configuration Structure

```rust
#[derive(Debug, Clone, PartialEq)]
pub struct ViaIndexerConfig {
    pub via_genesis_config: ViaGenesisConfig,
    pub via_btc_client_config: ViaBtcClientConfig,
    pub via_btc_watch_config: ViaBtcWatchConfig,
    pub observability_config: ObservabilityConfig,
    pub health_check: HealthCheckConfig,
    pub prometheus_config: PrometheusConfig,
    pub postgres_config: PostgresConfig,
    pub secrets: ViaSecrets,
}
```

## Deployment

### Docker Configuration

The L1 Indexer includes dedicated Docker support:

#### Dockerfile

```dockerfile
FROM rust:1.70 as builder

WORKDIR /app
COPY . .
RUN cargo build --release --bin via_indexer_bin

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/via_indexer_bin /usr/local/bin/via-indexer
EXPOSE 3000
CMD ["via-indexer"]
```

#### Docker Compose Integration

```yaml
services:
  via-indexer:
    image: via-l1-indexer:latest
    environment:
      - DATABASE_INDEXER_URL=postgresql://postgres:password@postgres:5432/via_indexer
      - VIA_BTC_CLIENT_RPC_URL=http://bitcoind:8332
      - VIA_BTC_CLIENT_RPC_USER=rpcuser
      - VIA_BTC_CLIENT_RPC_PASSWORD=rpcpassword
    depends_on:
      - postgres
      - bitcoind
    ports:
      - "3001:3000"
```

### CLI Deployment

Start the indexer using the CLI:

```bash
# Start indexer with default configuration
via-indexer --network regtest

# Start with custom configuration
via-indexer --network mainnet --config /path/to/config.toml

# Restart indexer service
via-restart-indexer
```

### Health Checks

The indexer provides health check endpoints:

```bash
# Check indexer health
curl http://localhost:3001/health

# Check indexer metrics
curl http://localhost:3001/metrics
```

## Monitoring and Metrics

### Prometheus Metrics

The L1 Indexer exposes the following metrics:

- `via_indexer_blocks_processed_total`: Total blocks processed
- `via_indexer_deposits_processed_total`: Total deposits processed
- `via_indexer_withdrawals_processed_total`: Total withdrawals processed
- `via_indexer_processing_duration_seconds`: Processing time per block
- `via_indexer_database_operations_total`: Database operation counters
- `via_indexer_errors_total`: Error counters by type

### Logging

Structured logging with configurable levels:

```toml
# Logging configuration
RUST_LOG="via_indexer=info,via_indexer_dal=debug"
```

Log categories:
- `via_indexer`: Main indexer service logs
- `via_indexer_dal`: Database access layer logs
- `via_indexer::deposit`: Deposit processing logs
- `via_indexer::withdrawal`: Withdrawal processing logs

## Troubleshooting

### Common Issues

#### 1. Database Connection Issues

**Symptom:** Indexer fails to start with database connection errors

**Solution:**
```bash
# Check database connectivity
psql $DATABASE_INDEXER_URL -c "SELECT 1;"

# Verify database schema
psql $DATABASE_INDEXER_URL -c "\dt"

# Run migrations if needed
sqlx migrate run --database-url $DATABASE_INDEXER_URL
```

#### 2. Bitcoin RPC Connection Issues

**Symptom:** Cannot connect to Bitcoin node

**Solution:**
```bash
# Test Bitcoin RPC connection
curl -u rpcuser:rpcpassword \
  -d '{"jsonrpc":"1.0","id":"test","method":"getblockchaininfo","params":[]}' \
  -H 'content-type: text/plain;' \
  http://localhost:8332/

# Check Bitcoin node status
bitcoin-cli -rpcuser=rpcuser -rpcpassword=rpcpassword getblockchaininfo
```

#### 3. Indexer Lag Issues

**Symptom:** Indexer falling behind Bitcoin blockchain

**Solutions:**
- Increase `VIA_INDEXER_BATCH_SIZE` for better throughput
- Optimize database queries and indexes
- Scale horizontally with multiple indexer instances
- Monitor system resources (CPU, memory, disk I/O)

#### 4. Data Consistency Issues

**Symptom:** Missing or duplicate transactions

**Solutions:**
```sql
-- Check for gaps in processed blocks
SELECT 
    block_number,
    LAG(block_number) OVER (ORDER BY block_number) as prev_block,
    block_number - LAG(block_number) OVER (ORDER BY block_number) as gap
FROM (
    SELECT DISTINCT block_number FROM deposits 
    UNION 
    SELECT DISTINCT block_number FROM bridge_withdrawals
) blocks
WHERE block_number - LAG(block_number) OVER (ORDER BY block_number) > 1;

-- Check for duplicate transactions
SELECT tx_id, COUNT(*) 
FROM deposits 
GROUP BY tx_id 
HAVING COUNT(*) > 1;
```

### Performance Optimization

#### Database Optimization

```sql
-- Add indexes for common queries
CREATE INDEX CONCURRENTLY idx_deposits_block_number ON deposits(block_number);
CREATE INDEX CONCURRENTLY idx_deposits_receiver ON deposits(receiver);
CREATE INDEX CONCURRENTLY idx_withdrawals_receiver ON withdrawals(receiver);
CREATE INDEX CONCURRENTLY idx_bridge_withdrawals_block_number ON bridge_withdrawals(block_number);

-- Analyze table statistics
ANALYZE deposits;
ANALYZE bridge_withdrawals;
ANALYZE withdrawals;
ANALYZE indexer_metadata;
```

#### Configuration Tuning

```toml
# Optimize for high throughput
VIA_INDEXER_BATCH_SIZE=500
VIA_INDEXER_POLL_INTERVAL=5
VIA_INDEXER_MAX_CONCURRENT_REQUESTS=10

# Database connection pool settings
DATABASE_MAX_CONNECTIONS=20
DATABASE_MIN_CONNECTIONS=5
DATABASE_ACQUIRE_TIMEOUT=30
```

## Integration Examples

### Querying Deposit Data

```rust
use via_indexer_dal::{ConnectionPool, IndexerDal, Indexer};

async fn get_recent_deposits(
    pool: &ConnectionPool<Indexer>,
    limit: i64,
) -> anyhow::Result<Vec<Deposit>> {
    let mut connection = pool.connection().await?;
    
    let deposits = sqlx::query_as!(
        Deposit,
        r#"
        SELECT priority_id, tx_id, block_number, receiver, value, calldata, canonical_tx_hash
        FROM deposits 
        ORDER BY block_number DESC, priority_id DESC 
        LIMIT $1
        "#,
        limit
    )
    .fetch_all(connection.as_mut())
    .await?;
    
    Ok(deposits)
}
```

### Monitoring Indexer Progress

```rust
async fn get_indexer_status(
    pool: &ConnectionPool<Indexer>,
) -> anyhow::Result<IndexerStatus> {
    let mut connection = pool.connection().await?;
    let mut dal = connection.via_indexer_dal();
    
    let deposit_block = dal.get_last_processed_l1_block("deposit").await?;
    let withdrawal_block = dal.get_last_processed_l1_block("withdrawal").await?;
    
    Ok(IndexerStatus {
        deposit_last_block: deposit_block,
        withdrawal_last_block: withdrawal_block,
        is_synced: deposit_block == withdrawal_block,
    })
}
```

The L1 Indexer system provides a robust, scalable foundation for Bitcoin transaction indexing within the Via L2 ecosystem, enabling efficient deposit and withdrawal processing while maintaining data integrity and performance.