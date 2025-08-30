# Via L2 Transaction Pool / Mempool

## Overview

The Transaction Pool (Mempool) is a critical component of the Via L2 Bitcoin ZK-Rollup architecture, responsible for buffering and managing pending L2 transactions before they are included in blocks by the Sequencer. The Mempool serves as an intermediary layer between the RPC API (where transactions are submitted) and the State Keeper (which processes transactions into blocks).

## Architecture

The Mempool implementation consists of several key components that work together to manage the transaction lifecycle:

### Core Components

1. **MempoolStore**: The central data structure that stores and manages pending transactions.
2. **MempoolGuard**: A thread-safe wrapper around MempoolStore that provides synchronized access.
3. **MempoolIO**: An interface between the State Keeper and the Mempool that handles transaction retrieval and batch creation.
4. **MempoolFetcher**: A component that synchronizes transactions from the database into the in-memory Mempool.

### Component Relationships

```
                  ┌─────────────────┐
                  │     RPC API     │
                  └────────┬────────┘
                           │ Submit Transactions
                           ▼
┌─────────────────────────────────────────────┐
│                  Mempool                    │
│  ┌─────────────┐      ┌──────────────────┐  │
│  │MempoolFetcher│◄────┤   Database DAL   │  │
│  └──────┬──────┘      └──────────────────┘  │
│         │                                   │
│         ▼                                   │
│  ┌─────────────┐      ┌──────────────────┐  │
│  │MempoolGuard ├─────►│   MempoolStore   │  │
│  └──────┬──────┘      └──────────────────┘  │
│         │                                   │
│         ▼                                   │
│  ┌─────────────┐                            │
│  │  MempoolIO  │                            │
│  └──────┬──────┘                            │
└─────────┼────────────────────────────────────┘
          │ Provide Transactions
          ▼
┌─────────────────────┐
│    State Keeper     │
└─────────────────────┘
```

## Implementation Details

### Data Structures

The Mempool uses several key data structures to efficiently manage transactions:

1. **MempoolStore**: The main container for pending transactions with the following fields:
   ```rust
   pub struct MempoolStore {
       /// Pending L1 transactions
       l1_transactions: HashMap<PriorityOpId, L1Tx>,
       /// Pending L2 transactions grouped by initiator address
       l2_transactions_per_account: HashMap<Address, AccountTransactions>,
       /// Global priority queue for L2 transactions. Used for scoring
       l2_priority_queue: BTreeSet<MempoolScore>,
       /// Next priority operation
       next_priority_id: PriorityOpId,
       stashed_accounts: Vec<Address>,
       /// Number of L2 transactions in the mempool.
       size: u64,
       capacity: u64,
   }
   ```
### Implementation updates (commits 0037–0044)

- New crate: [`core/lib/via_mempool`](core/lib/via_mempool/Cargo.toml) provides the mempool implementation used by State Keeper.
- Data structure adjustments in [`rust MempoolStore`](core/lib/via_mempool/src/mempool_store.rs:83):
  - L1 transactions now stored in `BTreeMap<PriorityOpId, L1Tx>` for ordered retrieval.
  - Field renamed to track last processed priority op: `last_priority_id` replaces `next_priority_id`.
- Public re-exports in [`rust lib.rs`](core/lib/via_mempool/src/lib.rs:51) expose `MempoolStore`, `MempoolStats`, `MempoolInfo`, and `L2TxFilter` to integrators (e.g., State Keeper).

2. **AccountTransactions**: Manages transactions for a specific account:
   ```rust
   pub(crate) struct AccountTransactions {
       /// transactions that belong to given account keyed by transaction nonce
       transactions: HashMap<Nonce, L2Tx>,
       /// account nonce in mempool
       /// equals to committed nonce in db + number of transactions sent to state keeper
       nonce: Nonce,
   }
   ```

3. **MempoolScore**: Used for transaction prioritization:
   ```rust
   pub struct MempoolScore {
       pub account: Address,
       pub received_at_ms: u64,
       pub fee_data: Fee,
   }
   ```

4. **L2TxFilter**: Used to filter transactions based on fee requirements:
   ```rust
   pub struct L2TxFilter {
       /// Batch fee model input. It typically includes things like L1 gas price, L2 fair fee, etc.
       pub fee_input: BatchFeeInput,
       /// Effective fee price for the transaction. The price of 1 gas in wei.
       pub fee_per_gas: u64,
       /// Effective pubdata price in gas for transaction. The number of gas per 1 pubdata byte.
       pub gas_per_pubdata: u32,
   }
   ```

### Transaction Flow

#### 1. Transaction Submission and Storage

Transactions are submitted through the RPC API and stored in the database. The `MempoolFetcher` periodically synchronizes these transactions from the database into the in-memory `MempoolStore`:

```rust
// From mempool_actor.rs
pub async fn run(mut self, stop_receiver: watch::Receiver<bool>) -> anyhow::Result<()> {
    // ...
    loop {
        // ...
        let mempool_info = self.mempool.get_mempool_info();
        // ...
        let transactions = storage
            .transactions_dal()
            .sync_mempool(
                &mempool_info.stashed_accounts,
                &mempool_info.purged_accounts,
                l2_tx_filter.gas_per_pubdata,
                l2_tx_filter.fee_per_gas,
                self.sync_batch_size,
            )
            .await
            .context("failed syncing mempool")?;
        let nonces = get_transaction_nonces(&mut storage, &transactions).await?;
        // ...
        self.mempool.insert(transactions, nonces);
        // ...
    }
    // ...
}
```

#### 2. Transaction Validation

Transactions undergo several validation checks:

1. **Nonce Validation**: Transactions are checked to ensure they have the correct nonce for the account:
   ```rust
   // From types.rs (AccountTransactions)
   pub fn insert(&mut self, transaction: L2Tx) -> InsertionMetadata {
       // ...
       // skip insertion if transaction is nonce is too old
       if nonce < self.nonce {
           return metadata;
       }
       // ...
   }
   ```

2. **Fee Validation**: Transactions must meet minimum fee requirements:
   ```rust
   // From types.rs (MempoolScore)
   pub fn matches_filter(&self, filter: &L2TxFilter) -> bool {
       self.fee_data.max_fee_per_gas >= U256::from(filter.fee_per_gas)
           && self.fee_data.gas_per_pubdata_limit >= U256::from(filter.gas_per_pubdata)
   }
   ```

3. **Gas Limit Validation**: Transactions with excessive gas limits are rejected:
   ```rust
   // From mempool.rs (MempoolIO)
   if tx.gas_limit() > self.max_allowed_tx_gas_limit {
       tracing::warn!(
           "Found tx with too big gas limit in state keeper, hash: {:?}, gas_limit: {}",
           tx.hash(),
           tx.gas_limit()
       );
       self.reject(&tx, UnexecutableReason::Halt(Halt::TooBigGasLimit))
           .await?;
       continue;
   }
   ```

#### 3. Transaction Ordering and Prioritization

Transactions are ordered primarily based on their received timestamp, with older transactions having higher priority:

```rust
// From types.rs (MempoolScore)
impl Ord for MempoolScore {
    fn cmp(&self, other: &MempoolScore) -> Ordering {
        match self.received_at_ms.cmp(&other.received_at_ms).reverse() {
            Ordering::Equal => {}
            ordering => return ordering,
        }
        self.account.cmp(&other.account)
    }
}
```

This ordering ensures that transactions are generally processed in a first-come, first-served manner, with a tiebreaker based on the account address.

#### 4. Transaction Selection for Blocks

The `MempoolIO` component selects transactions for inclusion in blocks based on the current fee requirements and other criteria:

```rust
// From mempool.rs (MempoolIO)
async fn wait_for_next_tx(
    &mut self,
    max_wait: Duration,
) -> anyhow::Result<Option<Transaction>> {
    let started_at = Instant::now();
    while started_at.elapsed() <= max_wait {
        let get_latency = KEEPER_METRICS.get_tx_from_mempool.start();
        let maybe_tx = self.mempool.next_transaction(&self.filter);
        get_latency.observe();

        if let Some(tx) = maybe_tx {
            // Validation checks...
            return Ok(Some(tx));
        } else {
            tokio::time::sleep(self.delay_interval).await;
            continue;
        }
    }
    Ok(None)
}
```

### Concurrency and Thread Safety

The Mempool implementation uses a mutex-based approach to ensure thread safety:

```rust
// From types.rs (MempoolGuard)
#[derive(Debug, Clone)]
pub struct MempoolGuard(Arc<Mutex<MempoolStore>>);

impl MempoolGuard {
    // ...
    pub fn next_transaction(&mut self, filter: &L2TxFilter) -> Option<Transaction> {
        self.0
            .lock()
            .expect("failed to acquire mempool lock")
            .next_transaction(filter)
    }
    // ...
}
```

This design allows multiple components to safely interact with the Mempool without race conditions.

### Mempool Size Management

The Mempool has a configurable capacity limit and implements garbage collection to prevent unbounded growth:

```rust
// From mempool_store.rs (MempoolStore)
fn gc(&mut self) -> Vec<Address> {
    if self.size >= self.capacity {
        let index: HashSet<_> = self
            .l2_priority_queue
            .iter()
            .map(|pointer| pointer.account)
            .collect();
        let transactions = std::mem::take(&mut self.l2_transactions_per_account);
        let (kept, drained) = transactions
            .into_iter()
            .partition(|(address, _)| index.contains(address));
        self.l2_transactions_per_account = kept;
        self.size = self
            .l2_transactions_per_account
            .iter()
            .fold(0, |agg, (_, tnxs)| agg + tnxs.len() as u64);
        return drained.into_keys().collect();
    }
    vec![]
}
```

Additionally, there's an optional mechanism to remove stuck transactions after a configurable timeout:

```rust
// From mempool_actor.rs (MempoolFetcher)
if let Some(stuck_tx_timeout) = self.stuck_tx_timeout {
    let removed_txs = storage
        .transactions_dal()
        .remove_stuck_txs(stuck_tx_timeout)
        .await
        .context("failed removing stuck transactions")?;
    tracing::info!("Number of stuck txs was removed: {removed_txs}");
}
```

## Interactions with Other Components

### 1. RPC/API Layer

The RPC/API layer submits transactions to the database, which are then picked up by the `MempoolFetcher` and added to the Mempool.

### 2. State Keeper

The State Keeper interacts with the Mempool through the `MempoolIO` interface, which provides transactions for inclusion in blocks and handles transaction rejections:

```rust
// From mempool.rs (MempoolIO)
async fn reject(
    &mut self,
    rejected: &Transaction,
    reason: UnexecutableReason,
) -> anyhow::Result<()> {
    // ...
    // Reset the nonces in the mempool, but don't insert the transaction back.
    self.mempool.rollback(rejected);

    // Mark tx as rejected in the storage.
    let mut storage = self.pool.connection_tagged("state_keeper").await?;
    // ...
    storage
        .transactions_dal()
        .mark_tx_as_rejected(rejected.hash(), &format!("rejected: {reason}"))
        .await?;
    Ok(())
}
```

### 3. Database Layer

The Mempool interacts with the database through the Data Access Layer (DAL) to:
- Fetch new transactions
- Update transaction statuses
- Retrieve account nonces
- Remove stuck transactions

## Configuration Parameters

The Mempool behavior can be customized through several configuration parameters:

```rust
// From mempool_actor.rs (tests)
const TEST_MEMPOOL_CONFIG: MempoolConfig = MempoolConfig {
    sync_interval_ms: 10,      // How often to sync with the database
    sync_batch_size: 100,      // Maximum number of transactions to fetch in one sync
    capacity: 100,             // Maximum number of transactions in the mempool
    stuck_tx_timeout: 0,       // Timeout for stuck transactions
    remove_stuck_txs: false,   // Whether to remove stuck transactions
    delay_interval: 10,        // Delay between transaction processing attempts
};
```

## Conclusion

The Via L2 Transaction Pool / Mempool is a sophisticated component that efficiently manages pending transactions, ensuring they are properly validated, prioritized, and delivered to the State Keeper for inclusion in blocks. Its design addresses key challenges such as transaction ordering, fee-based filtering, concurrency, and memory management, making it a critical part of the Via L2 Bitcoin ZK-Rollup architecture.