# Via L2 Transaction Pool / Mempool

## Overview

The Transaction Pool (Mempool) buffers and manages pending L2 transactions before they are included in blocks by the sequencer. It sits between the RPC API (where transactions are submitted and persisted to Postgres) and the State Keeper (which pulls transactions from the in-memory pool and executes them into blocks).

The machinery is inherited from zksync-era, with a Via-specific fork of the mempool crate. Two copies exist in the tree:

- `core/lib/mempool` (`zksync_mempool`): the original zksync-era crate, kept for the inherited `core/node/state_keeper`.
- `core/lib/via_mempool` (`via_mempool`): the Via fork, used by `core/node/via_state_keeper`. This is the crate that runs in the Via sequencer.

The Via fork differs from upstream in how L1 priority operations are stored and consumed, because Via priority operations come from Bitcoin deposits (parsed by `core/node/via_btc_watch`) rather than from an Ethereum priority queue with contiguous serial ids. See the "Bitcoin priority operations" section below.

## Architecture

### Core Components

1. **MempoolStore** (`core/lib/via_mempool/src/mempool_store.rs`): the central in-memory data structure that stores and manages pending transactions.
2. **MempoolGuard** (`core/node/via_state_keeper/src/types.rs`): a thread-safe `Arc<Mutex<MempoolStore>>` wrapper that provides synchronized access.
3. **MempoolIO** (`core/node/via_state_keeper/src/io/mempool.rs`): the `StateKeeperIO` implementation that feeds the state keeper from the mempool and decides batch parameters.
4. **MempoolFetcher** (`core/node/via_state_keeper/src/mempool_actor.rs`): a background task that synchronizes transactions from the database into the in-memory `MempoolStore`.

The wiring lives in `core/node/node_framework/src/implementations/layers/via_state_keeper/mempool_io.rs`.

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

Bitcoin deposits enter this picture on the database side: `via_btc_watch` parses deposit transactions from Bitcoin blocks and persists them as L1 priority operations, which `MempoolFetcher` then loads like any other pending transaction.

## The via_mempool crate

The public surface of the crate:

```rust
// core/lib/via_mempool/src/lib.rs
mod mempool_store;
#[cfg(test)]
mod tests;
mod types;

pub use crate::{
    mempool_store::{MempoolInfo, MempoolStats, MempoolStore},
    types::L2TxFilter,
};
```

### MempoolStore

Compared to the upstream `zksync_mempool::MempoolStore` (which keeps `l1_transactions: HashMap<PriorityOpId, L1Tx>` and a `next_priority_id` counter), the Via fork stores L1 transactions in a `BTreeMap` ordered by priority id and tracks the `last_priority_id` it has handed out:

```rust
// core/lib/via_mempool/src/mempool_store.rs
#[derive(Debug)]
pub struct MempoolInfo {
    pub stashed_accounts: Vec<Address>,
    pub purged_accounts: Vec<Address>,
}

#[derive(Debug)]
pub struct MempoolStats {
    pub l1_transaction_count: usize,
    pub l2_transaction_count: u64,
    pub l2_priority_queue_size: usize,
}

#[derive(Debug)]
pub struct MempoolStore {
    /// Pending L1 transactions
    l1_transactions: BTreeMap<PriorityOpId, L1Tx>,
    /// Pending L2 transactions grouped by initiator address
    l2_transactions_per_account: HashMap<Address, AccountTransactions>,
    /// Global priority queue for L2 transactions. Used for scoring
    l2_priority_queue: BTreeSet<MempoolScore>,
    /// Last priority operation
    last_priority_id: PriorityOpId,
    stashed_accounts: Vec<Address>,
    /// Number of L2 transactions in the mempool.
    size: u64,
    capacity: u64,
}

impl MempoolStore {
    pub fn new(last_priority_id: PriorityOpId, capacity: u64) -> Self {
        Self {
            l1_transactions: BTreeMap::new(),
            l2_transactions_per_account: HashMap::new(),
            l2_priority_queue: BTreeSet::new(),
            last_priority_id,
            stashed_accounts: vec![],
            size: 0,
            capacity,
        }
    }
```

The `BTreeMap` matters because Via priority ids are not contiguous counters: they are bit-packed Bitcoin coordinates, and the mempool must return them in ascending id order regardless of insertion order.

### AccountTransactions

Per-account L2 transactions are keyed by nonce, and each entry carries the `TransactionTimeRangeConstraint` it was submitted with:

```rust
// core/lib/via_mempool/src/types.rs
/// Pending mempool transactions of account
#[derive(Debug)]
pub(crate) struct AccountTransactions {
    /// transactions that belong to given account keyed by transaction nonce
    transactions: HashMap<Nonce, (L2Tx, TransactionTimeRangeConstraint)>,
    /// account nonce in mempool
    /// equals to committed nonce in db + number of transactions sent to state keeper
    nonce: Nonce,
}

impl AccountTransactions {
    pub fn new(nonce: Nonce) -> Self {
        Self {
            transactions: HashMap::new(),
            nonce,
        }
    }

    /// Inserts new transaction for given account. Returns insertion metadata
    pub fn insert(
        &mut self,
        transaction: L2Tx,
        constraint: TransactionTimeRangeConstraint,
    ) -> InsertionMetadata {
        let mut metadata = InsertionMetadata::default();
        let nonce = transaction.common_data.nonce;
        // skip insertion if transaction is old
        if nonce < self.nonce {
            return metadata;
        }
        let new_score = Self::score_for_transaction(&transaction);
        let previous_score = self
            .transactions
            .insert(nonce, (transaction, constraint))
            .map(|x| Self::score_for_transaction(&x.0));
        metadata.is_new = previous_score.is_none();
        if nonce == self.nonce {
            metadata.new_score = Some(new_score);
            metadata.previous_score = previous_score;
        }
        metadata
    }

    /// Returns next transaction to be included in block, its time range constraint and optional
    /// score of its successor. Panics if no such transaction exists
    pub fn next(&mut self) -> (L2Tx, TransactionTimeRangeConstraint, Option<MempoolScore>) {
        let transaction = self
            .transactions
            .remove(&self.nonce)
            .expect("missing transaction in mempool");
        self.nonce += 1;
        let score = self
            .transactions
            .get(&self.nonce)
            .map(|(tx, _c)| Self::score_for_transaction(tx));
        (transaction.0, transaction.1, score)
    }

    /// Handles transaction rejection. Returns optional score of its successor and time range
    /// constraint that the transaction has been added to the mempool with
    pub fn reset(
        &mut self,
        transaction: &Transaction,
    ) -> Option<(MempoolScore, TransactionTimeRangeConstraint)> {
        // current nonce for the group needs to be reset
        let tx_nonce = transaction
            .nonce()
            .expect("nonce is not set for L2 transaction");
        self.nonce = self.nonce.min(tx_nonce);
        self.transactions
            .get(&(tx_nonce + 1))
            .map(|(tx, c)| (Self::score_for_transaction(tx), c.clone()))
    }

    pub fn len(&self) -> usize {
        self.transactions.len()
    }

    fn score_for_transaction(transaction: &L2Tx) -> MempoolScore {
        MempoolScore {
            account: transaction.initiator_account(),
            received_at_ms: transaction.received_timestamp_ms,
            fee_data: transaction.common_data.fee.clone(),
        }
    }
}
```

### MempoolScore and L2TxFilter

```rust
// core/lib/via_mempool/src/types.rs
/// Mempool score of transaction. Used to prioritize L2 transactions in mempool
/// Currently trivial ordering is used based on received at timestamp
#[derive(Eq, PartialEq, Clone, Debug, Hash)]
pub struct MempoolScore {
    pub account: Address,
    pub received_at_ms: u64,
    // Not used for actual scoring, but state keeper would request
    // transactions that have acceptable fee values (so transactions
    // with fee too low would be ignored until prices go down).
    pub fee_data: Fee,
}

impl MempoolScore {
    /// Checks whether transaction matches requirements provided by state keeper.
    pub fn matches_filter(&self, filter: &L2TxFilter) -> bool {
        self.fee_data.max_fee_per_gas >= U256::from(filter.fee_per_gas)
            && self.fee_data.gas_per_pubdata_limit >= U256::from(filter.gas_per_pubdata)
    }
}

impl Ord for MempoolScore {
    fn cmp(&self, other: &MempoolScore) -> Ordering {
        match self.received_at_ms.cmp(&other.received_at_ms).reverse() {
            Ordering::Equal => {}
            ordering => return ordering,
        }
        self.account.cmp(&other.account)
    }
}

impl PartialOrd for MempoolScore {
    fn partial_cmp(&self, other: &MempoolScore) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

#[derive(Debug, Default)]
pub(crate) struct InsertionMetadata {
    pub new_score: Option<MempoolScore>,
    pub previous_score: Option<MempoolScore>,
    pub is_new: bool,
}

/// Structure that can be used by state keeper to describe
/// criteria for transaction it wants to fetch.
#[derive(Debug, Default, PartialEq, Eq)]
pub struct L2TxFilter {
    /// Batch fee model input. It typically includes things like L1 gas price, L2 fair fee, etc.
    pub fee_input: BatchFeeInput,
    /// Effective fee price for the transaction. The price of 1 gas in wei.
    pub fee_per_gas: u64,
    /// Effective pubdata price in gas for transaction. The number of gas per 1 pubdata byte.
    pub gas_per_pubdata: u32,
}
```

`received_at_ms` is compared in reverse, so the greatest `MempoolScore` in the `BTreeSet` belongs to the oldest transaction; `next_transaction` scans from that end (`rfind`), giving first-come, first-served ordering with the account address as a tiebreaker.

## Bitcoin priority operations

Via L1 priority operations do not come from Ethereum. `via_btc_watch` parses deposits from Bitcoin blocks, and each deposit gets a deterministic priority id packed from its Bitcoin coordinates:

```rust
// core/lib/types/src/l1/priority_id.rs
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct ViaPriorityOpId(pub u64);

/// VIA priorityId
/// 28 bits for block (268M blocks = 5,100 years when block time = 10min)
/// 20 bits for tx_index (1M transactions per block)
/// 16 bits for vout (65k outputs per transaction)
impl ViaPriorityOpId {
    // Constants for bit field sizes
    const BLOCK_BITS: u32 = 28;
    const TX_INDEX_BITS: u32 = 20;
    const VOUT_BITS: u32 = 16;

    // Bit masks
    const BLOCK_MASK: u64 = (1u64 << Self::BLOCK_BITS) - 1; // 0xFFFFFFF
    const TX_INDEX_MASK: u64 = (1u64 << Self::TX_INDEX_BITS) - 1; // 0xFFFFF
    const VOUT_MASK: u64 = (1u64 << Self::VOUT_BITS) - 1; // 0xFFFF

    // Bit positions
    const TX_INDEX_SHIFT: u32 = Self::VOUT_BITS;
    const BLOCK_SHIFT: u32 = Self::TX_INDEX_BITS + Self::VOUT_BITS;

    // Maximum values
    pub const MAX_BLOCK_NUMBER: u64 = Self::BLOCK_MASK;
    pub const MAX_TX_INDEX: u64 = Self::TX_INDEX_MASK;
    pub const MAX_VOUT: u64 = Self::VOUT_MASK;

    /// Creates a new PriorityOpId from components
    pub fn new(block_number: u64, tx_index: u64, vout: u64) -> Self {
        debug_assert!(
            block_number <= Self::MAX_BLOCK_NUMBER,
            "Block number {} exceeds maximum {}",
            block_number,
            Self::MAX_BLOCK_NUMBER
        );
        debug_assert!(
            tx_index <= Self::MAX_TX_INDEX,
            "TX index {} exceeds maximum {}",
            tx_index,
            Self::MAX_TX_INDEX
        );
        debug_assert!(
            vout <= Self::MAX_VOUT,
            "VOut {} exceeds maximum {}",
            vout,
            Self::MAX_VOUT
        );

        Self(
            ((block_number & Self::BLOCK_MASK) << Self::BLOCK_SHIFT)
                | ((tx_index & Self::TX_INDEX_MASK) << Self::TX_INDEX_SHIFT)
                | (vout & Self::VOUT_MASK),
        )
    }

    /// Extracts the block number
    pub fn block_number(&self) -> u64 {
        (self.0 >> Self::BLOCK_SHIFT) & Self::BLOCK_MASK
    }

    /// Extracts the transaction index
    pub fn tx_index(&self) -> u64 {
        (self.0 >> Self::TX_INDEX_SHIFT) & Self::TX_INDEX_MASK
    }

    /// Extracts the output vout
    pub fn vout(&self) -> u64 {
        self.0 & Self::VOUT_MASK
    }

    /// Gets the raw u64 value
    pub fn raw(&self) -> u64 {
        self.0
    }
}
```

A parsed deposit produces the `PriorityOpId` used as the L1 transaction's `serial_id`:

```rust
// core/lib/types/src/l1/via_l1.rs
#[derive(Debug, Clone)]

pub struct ViaL1Deposit {
    pub l2_receiver_address: Address,
    pub amount: u64,
    pub calldata: Vec<u8>,
    pub l1_block_number: u64,
    pub tx_index: usize,
    pub output_vout: usize,
}
```

```rust
// core/lib/types/src/l1/via_l1.rs
    pub fn priority_id(&self) -> PriorityOpId {
        PriorityOpId(
            ViaPriorityOpId::new(
                self.l1_block_number,
                self.tx_index as u64,
                self.output_vout as u64,
            )
            .raw(),
        )
    }
```

`From<ViaL1Deposit> for L1Tx` in the same file sets `common_data.serial_id: deposit.priority_id()`, so ids arrive in the mempool already carrying their Bitcoin ordering. This is why `MempoolStore` uses `BTreeMap<PriorityOpId, L1Tx>`: consecutive deposits have gaps between ids, and ordered retrieval via `pop_first` replaces the upstream assumption of a contiguous `next_priority_id` counter.

## Transaction Flow

### 1. Insertion into the store

`MempoolStore::insert` distinguishes L1 priority operations from L2 transactions. L1 transactions whose `serial_id` is at or below `last_priority_id` (already handed out) are skipped:

```rust
// core/lib/via_mempool/src/mempool_store.rs
    /// Inserts batch of new transactions to mempool
    /// `initial_nonces` provides current committed nonce information to mempool
    /// variable is used only if account is not present in mempool yet and we have to bootstrap it
    /// in other cases mempool relies on state keeper and its internal state to keep that info up to date
    pub fn insert(
        &mut self,
        transactions: Vec<(Transaction, TransactionTimeRangeConstraint)>,
        initial_nonces: HashMap<Address, Nonce>,
    ) {
        for (transaction, constraint) in transactions {
            let Transaction {
                common_data,
                execute,
                received_timestamp_ms,
                raw_bytes,
            } = transaction;
            match common_data {
                ExecuteTransactionCommon::L1(data) => {
                    if self.last_priority_id > data.serial_id {
                        continue;
                    }
                    tracing::trace!("inserting L1 transaction {}", data.serial_id);
                    self.l1_transactions.insert(
                        data.serial_id,
                        L1Tx {
                            execute,
                            common_data: data,
                            received_timestamp_ms,
                        },
                    );
                }
                ExecuteTransactionCommon::L2(data) => {
                    tracing::trace!("inserting L2 transaction {}", data.nonce);
                    self.insert_l2_transaction(
                        L2Tx {
                            execute,
                            common_data: data,
                            received_timestamp_ms,
                            raw_bytes,
                        },
                        constraint,
                        &initial_nonces,
                    );
                }
                ExecuteTransactionCommon::ProtocolUpgrade(_) => {
                    panic!("Protocol upgrade tx is not supposed to be inserted into mempool");
                }
            }
        }
    }
```

```rust
// core/lib/via_mempool/src/mempool_store.rs
    fn insert_l2_transaction(
        &mut self,
        transaction: L2Tx,
        constraint: TransactionTimeRangeConstraint,
        initial_nonces: &HashMap<Address, Nonce>,
    ) {
        let account = transaction.initiator_account();

        let metadata = match self.l2_transactions_per_account.entry(account) {
            hash_map::Entry::Occupied(mut txs) => txs.get_mut().insert(transaction, constraint),
            hash_map::Entry::Vacant(entry) => {
                let account_nonce = initial_nonces.get(&account).cloned().unwrap_or(Nonce(0));
                entry
                    .insert(AccountTransactions::new(account_nonce))
                    .insert(transaction, constraint)
            }
        };
        if let Some(score) = metadata.previous_score {
            self.l2_priority_queue.remove(&score);
        }
        if let Some(score) = metadata.new_score {
            self.l2_priority_queue.insert(score);
        }
        if metadata.is_new {
            self.size += 1;
        }
    }
```

### 2. Database synchronization (MempoolFetcher)

Transactions submitted through the RPC API land in Postgres. The `MempoolFetcher` loop pulls them into the in-memory store using `TransactionsDal::sync_mempool`, after first removing stuck transactions (if configured) and resetting the DB-side mempool marker:

```rust
// core/node/via_state_keeper/src/mempool_actor.rs
/// Creates a mempool filter for L2 transactions based on the current L1 gas price.
/// The filter is used to filter out transactions from the mempool that do not cover expenses
/// to process them.
pub async fn l2_tx_filter(
    batch_fee_input_provider: &dyn BatchFeeModelInputProvider,
    vm_version: VmVersion,
) -> anyhow::Result<L2TxFilter> {
    let fee_input = batch_fee_input_provider.get_batch_fee_input().await?;
    let (base_fee, gas_per_pubdata) = derive_base_fee_and_gas_per_pubdata(fee_input, vm_version);
    Ok(L2TxFilter {
        fee_input,
        fee_per_gas: base_fee,
        gas_per_pubdata: gas_per_pubdata as u32,
    })
}

#[derive(Debug)]
pub struct MempoolFetcher {
    mempool: MempoolGuard,
    pool: ConnectionPool<Core>,
    batch_fee_input_provider: Arc<dyn BatchFeeModelInputProvider>,
    sync_interval: Duration,
    sync_batch_size: usize,
    stuck_tx_timeout: Option<Duration>,
    #[cfg(test)]
    transaction_hashes_sender: mpsc::UnboundedSender<Vec<H256>>,
}
```

```rust
// core/node/via_state_keeper/src/mempool_actor.rs
    pub async fn run(mut self, stop_receiver: watch::Receiver<bool>) -> anyhow::Result<()> {
        let mut storage = self.pool.connection_tagged("via_state_keeper").await?;
        if let Some(stuck_tx_timeout) = self.stuck_tx_timeout {
            let removed_txs = storage
                .transactions_dal()
                .remove_stuck_txs(stuck_tx_timeout)
                .await
                .context("failed removing stuck transactions")?;
            tracing::info!("Number of stuck txs was removed: {removed_txs}");
        }
        storage.transactions_dal().reset_mempool().await?;
        drop(storage);

        loop {
            if *stop_receiver.borrow() {
                tracing::info!("Stop signal received, mempool is shutting down");
                break;
            }
            let latency = KEEPER_METRICS.mempool_sync.start();
            let mut storage = self.pool.connection_tagged("via_state_keeper").await?;
            let mempool_info = self.mempool.get_mempool_info();
            let protocol_version = storage
                .blocks_dal()
                .pending_protocol_version()
                .await
                .context("failed getting pending protocol version")?;

            let l2_tx_filter = l2_tx_filter(
                self.batch_fee_input_provider.as_ref(),
                protocol_version.into(),
            )
            .await
            .context("failed creating L2 transaction filter")?;

            let transactions_with_constraints = storage
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

            let transactions: Vec<_> = transactions_with_constraints
                .iter()
                .map(|(t, _c)| t)
                .collect();

            let nonces = get_transaction_nonces(&mut storage, &transactions).await?;
            drop(storage);

            #[cfg(test)]
            {
                let transaction_hashes = transactions.iter().map(|x| x.hash()).collect();
                self.transaction_hashes_sender.send(transaction_hashes).ok();
            }
            let all_transactions_loaded = transactions.len() < self.sync_batch_size;
            self.mempool.insert(transactions_with_constraints, nonces);
            latency.observe();

            if all_transactions_loaded {
                tokio::time::sleep(self.sync_interval).await;
            }
        }
        Ok(())
    }
```

Account nonces for accounts not yet tracked by the mempool are read straight from storage slots:

```rust
// core/node/via_state_keeper/src/mempool_actor.rs
/// Loads nonces for all distinct `transactions` initiators from the storage.
async fn get_transaction_nonces(
    storage: &mut Connection<'_, Core>,
    transactions: &[&Transaction],
) -> anyhow::Result<HashMap<Address, Nonce>> {
    let (nonce_keys, address_by_nonce_key): (Vec<_>, HashMap<_, _>) = transactions
        .iter()
        .map(|tx| {
            let address = tx.initiator_account();
            let nonce_key = get_nonce_key(&address).hashed_key();
            (nonce_key, (nonce_key, address))
        })
        .unzip();

    let nonce_values = storage
        .storage_web3_dal()
        .get_values(&nonce_keys)
        .await
        .context("failed getting nonces from storage")?;

    Ok(nonce_values
        .into_iter()
        .map(|(nonce_key, nonce_value)| {
            // `unwrap()` is safe by construction.
            let be_u32_bytes: [u8; 4] = nonce_value[28..].try_into().unwrap();
            let nonce = Nonce(u32::from_be_bytes(be_u32_bytes));
            (address_by_nonce_key[&nonce_key], nonce)
        })
        .collect())
}
```

### 3. Transaction selection

L1 priority operations always take precedence: `next_transaction` pops the lowest pending priority id before considering any L2 transaction. L2 candidates are taken from the priority queue end that holds the oldest matching score, and accounts whose head transaction no longer meets the fee filter are stashed:

```rust
// core/lib/via_mempool/src/mempool_store.rs
    /// Returns `true` if there is a transaction in the mempool satisfying the filter.
    pub fn has_next(&self, filter: &L2TxFilter) -> bool {
        self.l1_transactions.iter().next().is_some()
            || self
                .l2_priority_queue
                .iter()
                .rfind(|el| el.matches_filter(filter))
                .is_some()
    }

    /// Returns next transaction for execution from mempool
    pub fn next_transaction(
        &mut self,
        filter: &L2TxFilter,
    ) -> Option<(Transaction, TransactionTimeRangeConstraint)> {
        if let Some((_, transaction)) = self.l1_transactions.pop_first() {
            self.last_priority_id = transaction.common_data.serial_id;
            // L1 transactions can't use block.timestamp in AA and hence do not need to have a constraint
            return Some((
                transaction.into(),
                TransactionTimeRangeConstraint::default(),
            ));
        }

        let mut removed = 0;
        // We want to fetch the next transaction that would match the fee requirements.
        let tx_pointer = self
            .l2_priority_queue
            .iter()
            .rfind(|el| el.matches_filter(filter))?
            .clone();

        // Stash all observed transactions that don't meet criteria
        for stashed_pointer in self
            .l2_priority_queue
            .split_off(&tx_pointer)
            .into_iter()
            .skip(1)
        {
            removed += self
                .l2_transactions_per_account
                .remove(&stashed_pointer.account)
                .expect("mempool: dangling pointer in priority queue")
                .len();

            self.stashed_accounts.push(stashed_pointer.account);
        }
        // insert pointer to the next transaction if it exists
        let (transaction, constraint, score) = self
            .l2_transactions_per_account
            .get_mut(&tx_pointer.account)
            .expect("mempool: dangling pointer in priority queue")
            .next();

        if let Some(score) = score {
            self.l2_priority_queue.insert(score);
        }
        self.size = self
            .size
            .checked_sub((removed + 1) as u64)
            .expect("mempool size can't be negative");
        Some((transaction.into(), constraint))
    }
```

### 4. State keeper side (MempoolIO)

The state keeper consumes the mempool through `MempoolIO`:

```rust
// core/node/via_state_keeper/src/io/mempool.rs
/// Mempool-based sequencer for the state keeper.
/// Receives transactions from the database through the mempool filtering logic.
/// Decides which batch parameters should be used for the new batch.
/// This is an IO for the main server application.
#[derive(Debug)]
pub struct MempoolIO {
    mempool: MempoolGuard,
    pool: ConnectionPool<Core>,
    timeout_sealer: TimeoutSealer,
    l2_block_max_payload_size_sealer: L2BlockMaxPayloadSizeSealer,
    filter: L2TxFilter,
    l1_batch_params_provider: L1BatchParamsProvider,
    fee_account: Address,
    validation_computational_gas_limit: u32,
    max_allowed_tx_gas_limit: U256,
    delay_interval: Duration,
    // Used to keep track of gas prices to set accepted price per pubdata byte in blocks.
    batch_fee_input_provider: Arc<dyn BatchFeeModelInputProvider>,
    chain_id: L2ChainId,
    l2_da_validator_address: Option<Address>,
    pubdata_type: L1BatchCommitmentMode,
}
```

`wait_for_next_tx` polls the mempool and applies two last-line-of-defense checks, gas limit and `block.timestamp` range, rejecting violators:

```rust
// core/node/via_state_keeper/src/io/mempool.rs
    async fn wait_for_next_tx(
        &mut self,
        max_wait: Duration,
        l2_block_timestamp: u64,
    ) -> anyhow::Result<Option<Transaction>> {
        let started_at = Instant::now();
        while started_at.elapsed() <= max_wait {
            let get_latency = KEEPER_METRICS.get_tx_from_mempool.start();
            let maybe_tx = self.mempool.next_transaction(&self.filter);
            get_latency.observe();

            if let Some((tx, constraint)) = maybe_tx {
                // Reject transactions with too big gas limit. They are also rejected on the API level, but
                // we need to secure ourselves in case some tx will somehow get into mempool.
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

                // Reject transactions that violate block.timestamp constraints. Such transactions should be
                // rejected at the API level, but we need to protect ourselves in case if a transaction
                // goes outside of the allowed range while being in the mempool
                let matches_range = constraint
                    .timestamp_asserter_range
                    .map_or(true, |x| x.contains(&l2_block_timestamp));

                if !matches_range {
                    self.reject(
                        &tx,
                        UnexecutableReason::Halt(Halt::FailedBlockTimestampAssertion),
                    )
                    .await?;
                    continue;
                }

                return Ok(Some(tx));
            } else {
                tokio::time::sleep(self.delay_interval).await;
                continue;
            }
        }
        Ok(None)
    }
```

### 5. Rollback and rejection

A rollback (batch restarted after a rejected transaction) reinserts the transaction; a rejection resets the account nonce without reinsertion and marks the transaction as rejected in Postgres:

```rust
// core/node/via_state_keeper/src/io/mempool.rs
    async fn rollback(&mut self, tx: Transaction) -> anyhow::Result<()> {
        // Reset nonces in the mempool.
        let constraint = self.mempool.rollback(&tx);
        // Insert the transaction back.
        self.mempool.insert(vec![(tx, constraint)], HashMap::new());
        Ok(())
    }

    async fn reject(
        &mut self,
        rejected: &Transaction,
        reason: UnexecutableReason,
    ) -> anyhow::Result<()> {
        anyhow::ensure!(
            !rejected.is_l1(),
            "L1 transactions should not be rejected: {reason}"
        );

        // Reset the nonces in the mempool, but don't insert the transaction back.
        self.mempool.rollback(rejected);

        // Mark tx as rejected in the storage.
        let mut storage = self.pool.connection_tagged("state_keeper").await?;

        KEEPER_METRICS.inc_rejected_txs(reason.as_metric_label());

        tracing::warn!(
            "Transaction {} is rejected with error: {reason}",
            rejected.hash()
        );
        storage
            .transactions_dal()
            .mark_tx_as_rejected(rejected.hash(), &format!("rejected: {reason}"))
            .await?;
        Ok(())
    }
```

Inside the store, `rollback` handles the L1 and L2 cases separately. For an L1 transaction it winds `last_priority_id` back so the priority op can be re-fetched:

```rust
// core/lib/via_mempool/src/mempool_store.rs
    /// When a state_keeper starts the block over after a rejected transaction,
    /// we have to rollback the nonces/ids in the mempool and
    /// reinsert the transactions from the block back into mempool.
    pub fn rollback(&mut self, tx: &Transaction) -> TransactionTimeRangeConstraint {
        // rolling back the nonces and priority ids
        match &tx.common_data {
            ExecuteTransactionCommon::L1(data) => {
                // reset next priority id
                self.last_priority_id = self.last_priority_id.min(data.serial_id);
                TransactionTimeRangeConstraint::default()
            }
            ExecuteTransactionCommon::L2(_) => {
                if let Some((score, constraint)) = self
                    .l2_transactions_per_account
                    .get_mut(&tx.initiator_account())
                    .expect("account is not available in mempool")
                    .reset(tx)
                {
                    self.l2_priority_queue.remove(&score);
                    return constraint;
                }
                TransactionTimeRangeConstraint::default()
            }
            ExecuteTransactionCommon::ProtocolUpgrade(_) => {
                panic!("Protocol upgrade tx is not supposed to be in mempool");
            }
        }
    }
```

## Concurrency and Thread Safety

`MempoolGuard` wraps the store in `Arc<Mutex<...>>` so the fetcher, the IO, and the metrics collector can share it:

```rust
// core/node/via_state_keeper/src/types.rs
#[derive(Debug, Clone)]
pub struct MempoolGuard(Arc<Mutex<MempoolStore>>);

impl MempoolGuard {
    pub async fn from_storage(storage_processor: &mut Connection<'_, Core>, capacity: u64) -> Self {
        let next_priority_id = storage_processor
            .transactions_dal()
            .next_priority_id()
            .await;
        Self::new(next_priority_id, capacity)
    }

    pub(super) fn new(next_priority_id: PriorityOpId, capacity: u64) -> Self {
        let store = MempoolStore::new(next_priority_id, capacity);
        Self(Arc::new(Mutex::new(store)))
    }

    pub fn insert(
        &mut self,
        transactions: Vec<(Transaction, TransactionTimeRangeConstraint)>,
        nonces: HashMap<Address, Nonce>,
    ) {
        self.0
            .lock()
            .expect("failed to acquire mempool lock")
            .insert(transactions, nonces);
    }

    pub fn has_next(&self, filter: &L2TxFilter) -> bool {
        self.0
            .lock()
            .expect("failed to acquire mempool lock")
            .has_next(filter)
    }

    pub fn next_transaction(
        &mut self,
        filter: &L2TxFilter,
    ) -> Option<(Transaction, TransactionTimeRangeConstraint)> {
        self.0
            .lock()
            .expect("failed to acquire mempool lock")
            .next_transaction(filter)
    }

    pub fn rollback(&mut self, rejected: &Transaction) -> TransactionTimeRangeConstraint {
        self.0
            .lock()
            .expect("failed to acquire mempool lock")
            .rollback(rejected)
    }

    pub fn get_mempool_info(&mut self) -> MempoolInfo {
        self.0
            .lock()
            .expect("failed to acquire mempool lock")
            .get_mempool_info()
    }

    pub fn register_metrics(&self) {
        StateKeeperGauges::register(Arc::downgrade(&self.0));
    }
}
```

On startup, `MempoolGuard::from_storage` seeds `last_priority_id` from `TransactionsDal::next_priority_id()`.

## Mempool Size Management

The store has a configurable capacity. Garbage collection runs inside `get_mempool_info()` (called every fetcher iteration): when the pool is at or over capacity, it keeps only accounts that still have a pointer in the priority queue and purges the rest, reporting them as `purged_accounts` so the fetcher can re-sync them from the database later:

```rust
// core/lib/via_mempool/src/mempool_store.rs
    pub fn get_mempool_info(&mut self) -> MempoolInfo {
        MempoolInfo {
            stashed_accounts: std::mem::take(&mut self.stashed_accounts),
            purged_accounts: self.gc(),
        }
    }

    pub fn stats(&self) -> MempoolStats {
        MempoolStats {
            l1_transaction_count: self.l1_transactions.len(),
            l2_transaction_count: self.size,
            l2_priority_queue_size: self.l2_priority_queue.len(),
        }
    }

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

Additionally, when `remove_stuck_txs` is enabled, `MempoolFetcher::run` calls `TransactionsDal::remove_stuck_txs(stuck_tx_timeout)` once at startup (shown in the `run` body above) to purge transactions that have been sitting in the database mempool for too long.

## Metrics

`MempoolGuard::register_metrics` registers a `vise` collector that reads `MempoolStore::stats()` on every scrape:

```rust
// core/node/via_state_keeper/src/metrics.rs
/// State keeper-related gauges exposed via a collector.
#[derive(Debug, Metrics)]
#[metrics(prefix = "via_server_state_keeper")]
pub(super) struct StateKeeperGauges {
    /// Current number of L1 transactions in the mempool.
    mempool_l1_size: Gauge<usize>,
    /// Current number of L2 transactions in the mempool.
    mempool_l2_size: Gauge<u64>,
    /// Current size of the L2 priority queue.
    l2_priority_queue_size: Gauge<usize>,
}

impl StateKeeperGauges {
    pub(super) fn register(pool_ref: Weak<Mutex<MempoolStore>>) {
        #[vise::register]
        static COLLECTOR: vise::Collector<Option<StateKeeperGauges>> = vise::Collector::new();

        let res = COLLECTOR.before_scrape(move || {
            pool_ref.upgrade().map(|pool| {
                let stats = pool.lock().expect("failed to acquire mempool lock").stats();
                drop(pool); // Don't prevent the pool to be dropped

                let gauges = StateKeeperGauges::default();
                gauges.mempool_l1_size.set(stats.l1_transaction_count);
                gauges.mempool_l2_size.set(stats.l2_transaction_count);
                gauges
                    .l2_priority_queue_size
                    .set(stats.l2_priority_queue_size);
                gauges
            })
        });
        if res.is_err() {
            tracing::warn!(
                "Mempool registered for metrics multiple times; this is a logical error"
            );
        }
    }
}
```

The same metrics module also defines `KEEPER_METRICS.mempool_sync` (fetcher iteration latency) and `KEEPER_METRICS.get_tx_from_mempool` (latency of pulling a transaction), both used in the code above.

## Configuration Parameters

Mempool behavior is driven by `MempoolConfig`, part of the inherited chain configuration:

```rust
// core/lib/config/src/configs/chain.rs
#[derive(Debug, Deserialize, Clone, PartialEq)]
pub struct MempoolConfig {
    pub sync_interval_ms: u64,
    pub sync_batch_size: usize,
    pub capacity: u64,
    pub stuck_tx_timeout: u64,
    pub remove_stuck_txs: bool,
    pub delay_interval: u64,
}

impl MempoolConfig {
    pub fn sync_interval(&self) -> Duration {
        Duration::from_millis(self.sync_interval_ms)
    }

    pub fn stuck_tx_timeout(&self) -> Duration {
        Duration::from_secs(self.stuck_tx_timeout)
    }

    pub fn delay_interval(&self) -> Duration {
        Duration::from_millis(self.delay_interval)
    }
}
```

- `sync_interval_ms`: how long the fetcher sleeps after a sync round that loaded fewer than `sync_batch_size` transactions.
- `sync_batch_size`: maximum number of transactions fetched from the database per round.
- `capacity`: L2 transaction capacity of the in-memory store (triggers `gc`).
- `stuck_tx_timeout` / `remove_stuck_txs`: startup purge of transactions stuck in the database mempool (`stuck_tx_timeout` is in seconds).
- `delay_interval`: sleep between `next_transaction` polls in `MempoolIO::wait_for_next_tx` (milliseconds).

The tests in `core/node/via_state_keeper/src/mempool_actor.rs` exercise the fetcher with:

```rust
// core/node/via_state_keeper/src/mempool_actor.rs
    const TEST_MEMPOOL_CONFIG: MempoolConfig = MempoolConfig {
        sync_interval_ms: 10,
        sync_batch_size: 100,
        capacity: 100,
        stuck_tx_timeout: 0,
        remove_stuck_txs: false,
        delay_interval: 10,
    };
```

## Conclusion

The Via mempool is inherited zksync-era machinery with one substantive fork: L1 priority operations are Bitcoin deposits identified by bit-packed `ViaPriorityOpId` values, so `via_mempool::MempoolStore` keeps them in a `BTreeMap` keyed by `PriorityOpId` and tracks `last_priority_id` instead of assuming a contiguous counter. Everything else (fee-filtered L2 selection, nonce-ordered per-account queues, mutex-guarded sharing, database synchronization, capacity GC) follows the upstream design, implemented in `core/lib/via_mempool` and `core/node/via_state_keeper`.
