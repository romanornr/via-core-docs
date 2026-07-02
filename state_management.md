# State Management in Via L2

This document provides a comprehensive overview of the state management system in the Via L2 Bitcoin ZK-Rollup. It covers the architecture, implementation details, and interactions with other components of the system.

## 1. Overview

The state management system in Via L2 is responsible for maintaining, updating, and committing the rollup state. It consists of several key components:

1. **State Keeper**: The core component responsible for processing transactions, executing them, and updating the state.
2. **Merkle Tree**: A sparse Merkle tree implementation used to represent and cryptographically commit to the state.
3. **Storage Layer**: Database abstractions for persisting the state.
4. **State Updates**: Mechanisms for applying and tracking changes to the state.

The state in Via L2 is represented as a key-value store, where keys are storage slots (consisting of an address and a key) and values are 32-byte values stored in those slots. The state is cryptographically committed using a Merkle tree, which allows for efficient verification of state transitions.

## 2. Primary Components

### 2.1 State Keeper

The State Keeper is the central component responsible for maintaining the state of the rollup. It processes transactions, executes them, and updates the state accordingly.

#### Key Files and Modules

- `core/node/via_state_keeper/src/keeper.rs`: Contains the `ZkSyncStateKeeper` implementation
- `core/node/via_state_keeper/src/lib.rs`: Exports the main components of the state keeper
- `core/node/via_state_keeper/src/executor/`: Batch executor factory trait and glue (the module is named `executor`, not `batch_executor`)
- `core/node/via_state_keeper/src/io/`: Handles I/O operations for the state keeper (`MempoolIO`, output handler, persistence, seal logic)
- `core/node/via_state_keeper/src/seal_criteria/`: Defines criteria for sealing L1 batches and L2 blocks
- `core/node/via_state_keeper/src/updates/`: Manages state updates (`UpdatesManager`)
- `core/node/via_state_keeper/src/mempool_actor.rs`: `MempoolFetcher`, which syncs the in-memory mempool from Postgres
- `core/node/via_state_keeper/src/state_keeper_storage.rs`: `AsyncRocksdbCache`, the RocksDB-backed storage factory used by the state keeper

The crate's public API, verbatim:

```rust
// core/node/via_state_keeper/src/lib.rs
pub use self::{
    io::{
        mempool::MempoolIO, L2BlockParams, L2BlockSealerTask, OutputHandler, StateKeeperIO,
        StateKeeperOutputHandler, StateKeeperPersistence, TreeWritesPersistence,
    },
    keeper::ZkSyncStateKeeper,
    mempool_actor::MempoolFetcher,
    seal_criteria::SequencerSealer,
    state_keeper_storage::AsyncRocksdbCache,
    types::MempoolGuard,
    updates::UpdatesManager,
};
```

#### ZkSyncStateKeeper

The `ZkSyncStateKeeper` is the main implementation of the state keeper. It's responsible for:

1. Processing transactions and updating the state
2. Deciding when to seal L2 blocks and L1 batches
3. Handling protocol upgrades
4. Managing the execution of transactions in batches

```rust
// core/node/via_state_keeper/src/keeper.rs
/// State keeper represents a logic layer of L1 batch / L2 block processing flow.
/// It's responsible for taking all the data from the `StateKeeperIO`, feeding it into `BatchExecutor` objects
/// and calling `SealManager` to decide whether an L2 block or L1 batch should be sealed.
///
/// State keeper maintains the batch execution state in the `UpdatesManager` until batch is sealed and these changes
/// are persisted by the `StateKeeperIO` implementation.
///
/// You can think of it as a state machine that runs over a sequence of incoming transactions, turning them into
/// a sequence of executed L2 blocks and batches.
#[derive(Debug)]
pub struct ZkSyncStateKeeper {
    io: Box<dyn StateKeeperIO>,
    output_handler: OutputHandler,
    batch_executor: Box<dyn BatchExecutorFactory<OwnedStorage>>,
    sealer: Arc<dyn ConditionalSealer>,
    storage_factory: Arc<dyn ReadStorageFactory>,
    health_updater: HealthUpdater,
}

impl ZkSyncStateKeeper {
    pub fn new(
        sequencer: Box<dyn StateKeeperIO>,
        batch_executor: Box<dyn BatchExecutorFactory<OwnedStorage>>,
        output_handler: OutputHandler,
        sealer: Arc<dyn ConditionalSealer>,
        storage_factory: Arc<dyn ReadStorageFactory>,
    ) -> Self {
        Self {
            io: sequencer,
            batch_executor,
            output_handler,
            sealer,
            storage_factory,
            health_updater: ReactiveHealthCheck::new("state_keeper").1,
        }
    }

    pub async fn run(mut self, stop_receiver: watch::Receiver<bool>) -> anyhow::Result<()> {
        match self.run_inner(stop_receiver).await {
            Ok(_) => unreachable!(),
            Err(Error::Fatal(err)) => Err(err).context("state_keeper failed"),
            Err(Error::Canceled) => {
                tracing::info!("Stop signal received, state keeper is shutting down");
                Ok(())
            }
        }
    }
```

Note that the stop signal is not a struct field: `run()` takes the `watch::Receiver<bool>` as an argument. The batch executor is a `BatchExecutorFactory<OwnedStorage>` (there is no `ErasedBatchExecutor` type in this tree), and the keeper additionally holds a `ReadStorageFactory` and a `HealthUpdater`. This is inherited zksync-era state keeper machinery, vendored into the `via_state_keeper` crate.

#### State Keeper Workflow

1. **Initialization**: The state keeper is initialized with a connection to the database, a batch executor, and a sealer.
2. **Transaction Processing**: Transactions are fetched from the mempool and executed one by one.
3. **State Updates**: After each transaction execution, the state is updated with the changes.
4. **Sealing Decisions**: The sealer decides when to seal L2 blocks and L1 batches based on various criteria.
5. **Persistence**: When a block or batch is sealed, the changes are persisted to the database.

### 2.2 Merkle Tree

The Merkle tree is used to represent and cryptographically commit to the state. It allows for efficient verification of state transitions and generation of proofs.

#### Key Files and Modules

- `core/lib/merkle_tree/src/lib.rs`: Core Merkle tree implementation
- `core/lib/merkle_tree/src/domain.rs`: Domain-specific wrapper for the Merkle tree
- `core/lib/merkle_tree/src/hasher/`: Hashing implementations for the Merkle tree
- `core/lib/merkle_tree/src/storage/`: Storage implementations for the Merkle tree

#### MerkleTree and ZkSyncTree

The `MerkleTree` is a generic implementation of a sparse Merkle tree, while `ZkSyncTree` is a domain-specific wrapper that ties the Merkle tree to the Via L2 system. This crate is inherited zksync-era machinery, unchanged in Via.

```rust
// core/lib/merkle_tree/src/lib.rs
/// Binary Merkle tree implemented using AR16MT from Diem [Jellyfish Merkle tree] white paper.
///
/// A tree is persistent and is backed by a key-value store (the `DB` type param). It is versioned,
/// meaning that the store retains *all* versions of the tree since its inception. A version
/// corresponds to a block number in the domain model; it is a `u64` counter incremented each time
/// a block of changes is committed into the tree via [`Self::extend()`]. It is possible to reset
/// the tree to a previous version via [`Self::truncate_versions()`].
#[derive(Debug)]
pub struct MerkleTree<DB, H = Blake2Hasher> {
    db: DB,
    hasher: H,
}
```

```rust
// core/lib/merkle_tree/src/domain.rs
/// Domain-specific wrapper of the Merkle tree.
///
/// This wrapper will accumulate changes introduced by [`Self::process_l1_batch()`],
/// [`Self::process_l1_batches()`] and [`Self::revert_logs()`] in RAM without saving them
/// to RocksDB. The accumulated changes can be saved to RocksDB via [`Self::save()`]
/// or discarded via [`Self::reset()`].
#[derive(Debug)]
pub struct ZkSyncTree {
    tree: MerkleTree<Patched<RocksDBWrapper>>,
    thread_pool: Option<ThreadPool>,
    mode: TreeMode,
    pruning_enabled: bool,
}
```

The Merkle tree is implemented using AR16MT (a variant of the Jellyfish Merkle tree), which is a binary Merkle tree with radix-16 optimization for I/O efficiency. Each internal node may have up to 16 children, and paths of internal nodes that do not fork are removed to optimize storage. Per the crate docs in `core/lib/merkle_tree/src/lib.rs`: "The I/O optimizations do not influence tree hashing."

#### Tree Hashing Specification

Quoting the crate-level docs verbatim:

```rust
// core/lib/merkle_tree/src/lib.rs
//! # Tree hashing specification
//!
//! A tree is hashed as if it was a full binary Merkle tree with `2^256` leaves:
//!
//! - Hash of a vacant leaf is `hash([0_u8; 40])`, where `hash` is the hash function used
//!   (Blake2s-256).
//! - Hash of an occupied leaf is `hash(u64::to_be_bytes(leaf_index) ++ value_hash)`,
//!   where `leaf_index` is a 1-based index of the leaf key provided when the leaf is inserted / updated,
//!   `++` is byte concatenation.
//! - Hash of an internal node is `hash(left_child_hash ++ right_child_hash)`.
```

#### Tree Operations

The Merkle tree supports the following operations:

1. **Extending**: Adding new entries to the tree, creating a new version.
2. **Reading**: Reading entries from the tree at a specific version.
3. **Generating Proofs**: Generating Merkle proofs for entries in the tree.
4. **Truncating**: Removing recent versions of the tree.
5. **Pruning**: Removing old versions of the tree to save space.

### 2.3 Storage Layer

The storage layer is responsible for persisting the state to disk. It consists of several components that abstract the underlying storage.

#### Key Files and Modules

- `core/lib/state/src/lib.rs`: Core state storage abstractions
- `core/lib/state/src/postgres/`: PostgreSQL storage implementation
- `core/lib/state/src/rocksdb/`: RocksDB storage implementation
- `core/lib/dal/src/storage_logs_dal.rs`: Data access layer for storage logs

#### Storage Implementations

The state module includes several storage implementations:

1. **PostgresStorage**: A PostgreSQL-based storage implementation
2. **RocksdbStorage**: A RocksDB-based storage implementation
3. **ShadowStorage**: A storage wrapper that checks a source of truth against a shadowed storage and logs mismatches

```rust
// core/lib/state/src/lib.rs
pub use self::{
    cache::sequential_cache::SequentialCache,
    catchup::{AsyncCatchupTask, RocksdbCell},
    postgres::{PostgresStorage, PostgresStorageCaches, PostgresStorageCachesTask},
    rocksdb::{
        RocksdbStorage, RocksdbStorageBuilder, RocksdbStorageOptions, StateKeeperColumnFamily,
    },
    shadow_storage::ShadowStorage,
    storage_factory::{
        BatchDiff, CommonStorage, OwnedStorage, ReadStorageFactory, RocksdbWithMemory,
        SnapshotStorage,
    },
};
```

Note: the `storage_factory` exports are `CommonStorage` and `SnapshotStorage`; there is no `PgOrRocksdbStorage` type in this tree (that name existed in older zksync-era versions).

#### Database Schema

The `storage_logs` table is created in the initial migration (inherited from zksync-era) and then reshaped by later migrations. The original definition:

```sql
-- core/lib/dal/migrations/20211026134308_init.up.sql
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

CREATE INDEX storage_logs_block_number_idx ON storage_logs (block_number);
CREATE INDEX storage_logs_raw_key_block_number_idx ON storage_logs (raw_key, block_number DESC, operation_number DESC);
```

Later migrations rename `raw_key` to `hashed_key`, make it part of the primary key, rename `block_number` to `miniblock_number`, and relax nullability of the key preimage and `tx_hash` columns:

```sql
-- core/lib/dal/migrations/20220401114554_storage_tables_migration.up.sql
DROP INDEX storage_logs_raw_key_block_number_idx;
ALTER TABLE storage_logs DROP CONSTRAINT storage_logs_pkey;
ALTER TABLE storage_logs DROP COLUMN id;
ALTER TABLE storage_logs RENAME COLUMN raw_key TO hashed_key;
ALTER TABLE storage_logs ADD PRIMARY KEY (hashed_key, block_number, operation_number);
```

```sql
-- core/lib/dal/migrations/20220827110416_miniblocks.up.sql (storage_logs part)
ALTER TABLE storage_logs DROP CONSTRAINT storage_logs_block_number_fkey;
ALTER TABLE storage_logs RENAME COLUMN block_number TO miniblock_number;
```

```sql
-- core/lib/dal/migrations/20240617103351_make_key_preimage_nullable.up.sql
ALTER TABLE storage_logs
    ALTER COLUMN address DROP NOT NULL,
    ALTER COLUMN key DROP NOT NULL;
```

```sql
-- core/lib/dal/migrations/20240619060210_make_storage_logs_tx_hash_nullable.up.sql
ALTER TABLE storage_logs ALTER COLUMN tx_hash DROP NOT NULL;
```

Each storage log entry contains:
- `hashed_key`: The hashed storage key (Blake2s-256 of padded address and key, see section 3.1)
- `address`: The contract address (nullable preimage)
- `key`: The storage key (nullable preimage)
- `value`: The storage value
- `operation_number`: The operation number within the block
- `miniblock_number`: The L2 block number
- `tx_hash`, `created_at`, `updated_at`: Metadata fields

The data access layer for this table lives in `core/lib/dal/src/storage_logs_dal.rs`.

### 2.4 State Updates

State updates are managed by the `UpdatesManager`, which tracks changes to the state during batch execution.

#### Key Files and Modules

- `core/node/via_state_keeper/src/updates/`: Contains the `UpdatesManager` implementation

#### UpdatesManager

The `UpdatesManager` is responsible for tracking changes to the state during batch execution:

```rust
// core/node/via_state_keeper/src/updates/mod.rs
/// Most of the information needed to seal the l1 batch / L2 block is contained within the VM,
/// things that are not captured there are accumulated externally.
/// `L2BlockUpdates` keeps updates for the pending L2 block.
/// `L1BatchUpdates` keeps updates for the already sealed L2 blocks of the pending L1 batch.
/// `UpdatesManager` manages the state of both of these accumulators to be consistent
/// and provides information about the pending state of the current L1 batch.
#[derive(Debug)]
pub struct UpdatesManager {
    batch_timestamp: u64,
    pub fee_account_address: Address,
    batch_fee_input: BatchFeeInput,
    base_fee_per_gas: u64,
    base_system_contract_hashes: BaseSystemContractsHashes,
    protocol_version: ProtocolVersionId,
    storage_view_cache: Option<StorageViewCache>,
    pub l1_batch: L1BatchUpdates,
    pub l2_block: L2BlockUpdates,
    pub storage_writes_deduplicator: StorageWritesDeduplicator,
    pubdata_params: PubdataParams,
}
```

When a transaction is executed, the `UpdatesManager` extends its state with the changes from the transaction (executed transactions are accumulated in `L2BlockUpdates` and rolled up into `L1BatchUpdates` on L2 block sealing). When a block or batch is sealed, the changes are persisted to the database via the seal logic in `core/node/via_state_keeper/src/io/seal_logic/`.

## 3. State Representation

### 3.1 Storage Model

The state in Via L2 is represented as a key-value store, where:

- **Keys**: Storage slots, consisting of an address (20 bytes) and a key (32 bytes)
- **Values**: 32-byte values stored in those slots

Each storage slot is uniquely identified by its hashed key. Contrary to what one might expect, this is not Keccak-256: it is the Blake2s-256 hash of the zero-padded address concatenated with the key:

```rust
// core/lib/types/src/storage/mod.rs
/// Typed fully qualified key of the storage slot in global state tree.
#[derive(Debug, Copy, Clone, Hash, Eq, PartialEq, Ord, PartialOrd)]
#[derive(Serialize, Deserialize)]
pub struct StorageKey {
    account: AccountTreeId,
    key: H256,
}

impl StorageKey {
    pub fn raw_hashed_key(address: &H160, key: &H256) -> [u8; 32] {
        let mut bytes = [0u8; 64];
        bytes[12..32].copy_from_slice(&address.0);
        U256::from(key.to_fixed_bytes()).to_big_endian(&mut bytes[32..64]);

        Blake2s256::digest(bytes).into()
    }

    pub fn hashed_key(&self) -> H256 {
        Self::raw_hashed_key(self.address(), self.key()).into()
    }

    pub fn hashed_key_u256(&self) -> U256 {
        U256::from_little_endian(&Self::raw_hashed_key(self.address(), self.key()))
    }
}
```

(Methods `new`, `account`, `key`, and `address` are omitted here; see the source file.)

### 3.2 Merkle Tree Structure

The state is cryptographically committed using a sparse Merkle tree with the following properties:

- **Depth**: 256 levels (one bit per level)
- **Radix**: 16 (for I/O efficiency)
- **Hashing**: Blake2s-256

The tree is sparse, meaning that only the nodes that are needed to represent the current state are stored. This allows for efficient storage of the state, as most storage slots are empty.

### 3.3 State Versioning

The state is versioned by L1 batch number. Each L1 batch creates a new version of the state, which is committed to the Merkle tree. This allows for efficient verification of state transitions and generation of proofs.

## 4. State Update Process

### 4.1 Transaction Execution

Transactions are executed by the batch executor, which applies the changes to the state. The execution process is as follows:

1. The transaction is fetched from the mempool.
2. The batch executor executes the transaction, producing a result that includes the state changes.
3. The state keeper decides whether to include the transaction in the current batch based on the execution result and sealing criteria.
4. If the transaction is included, the state is updated with the changes.

### 4.2 Block and Batch Sealing

Sealing decisions are split between two mechanisms in `core/node/via_state_keeper/src/seal_criteria/`:

1. Transaction-dependent criteria, aggregated by `SequencerSealer` (a `ConditionalSealer` implementation). The default set:

```rust
// core/node/via_state_keeper/src/seal_criteria/conditional_sealer.rs
    fn default_sealers(config: &StateKeeperConfig) -> Vec<Box<dyn SealCriterion>> {
        vec![
            Box::new(criteria::SlotsCriterion),
            Box::new(criteria::PubDataBytesCriterion {
                max_pubdata_per_batch: config.max_pubdata_per_batch,
            }),
            Box::new(criteria::CircuitsCriterion),
            Box::new(criteria::TxEncodingSizeCriterion),
            Box::new(criteria::GasForBatchTipCriterion),
            Box::new(criteria::L1L2TxsCriterion),
            Box::new(criteria::L2L1LogsCriterion),
        ]
    }
```

2. I/O-dependent criteria implementing the `IoSealCriteria` trait, notably `TimeoutSealer` (deadline-based sealing) and `L2BlockMaxPayloadSizeSealer`:

```rust
// core/node/via_state_keeper/src/seal_criteria/mod.rs
/// I/O-dependent seal criteria.
pub trait IoSealCriteria {
    /// Checks whether an L1 batch should be sealed unconditionally (i.e., regardless of metrics
    /// related to transaction execution) given the provided `manager` state.
    fn should_seal_l1_batch_unconditionally(&mut self, manager: &UpdatesManager) -> bool;
    /// Checks whether an L2 block should be sealed given the provided `manager` state.
    fn should_seal_l2_block(&mut self, manager: &UpdatesManager) -> bool;
}

#[derive(Debug, Clone, Copy)]
pub(super) struct TimeoutSealer {
    block_commit_deadline_ms: u64,
    l2_block_commit_deadline_ms: u64,
}
```

When a block or batch is sealed, the changes are persisted to the database, and a new block or batch is started.

### 4.3 State Commitment

The state is committed to the Merkle tree after each L1 batch. The root hash of the Merkle tree serves as a cryptographic commitment to the state, which can be used for verification.

The component that feeds sealed batches into the tree is the Metadata Calculator (`core/node/metadata_calculator`, inherited zksync-era machinery). It runs as a background task that polls Postgres for new sealed L1 batches, applies their storage logs to the `ZkSyncTree`, and writes the resulting tree metadata (root hash, leaf count) back to Postgres:

```rust
// core/node/metadata_calculator/src/lib.rs
#[derive(Debug)]
pub struct MetadataCalculator {
    config: MetadataCalculatorConfig,
    tree_reader: watch::Sender<Option<AsyncTreeReader>>,
    pruning_handles_sender: oneshot::Sender<PruningHandles>,
    object_store: Option<Arc<dyn ObjectStore>>,
    pool: ConnectionPool<Core>,
    recovery_pool: ConnectionPool<Core>,
    delayer: Delayer,
    health_updater: HealthUpdater,
    max_l1_batches_per_iter: usize,
}
```

The Via server wires it in explicitly:

```rust
// core/bin/via_server/src/node_builder.rs
    fn add_metadata_calculator_layer(mut self, with_tree_api: bool) -> anyhow::Result<Self> {
        let merkle_tree_env_config = try_load_config!(self.configs.db_config).merkle_tree;
        let operations_manager_env_config =
            try_load_config!(self.configs.operations_manager_config);
        let state_keeper_env_config = try_load_config!(self.configs.state_keeper_config);
        let metadata_calculator_config = MetadataCalculatorConfig::for_main_node(
            &merkle_tree_env_config,
            &operations_manager_env_config,
            &state_keeper_env_config,
        );
        let mut layer = MetadataCalculatorLayer::new(metadata_calculator_config);
        if with_tree_api {
            let merkle_tree_api_config = try_load_config!(self.configs.api_config).merkle_tree;
            layer = layer.with_tree_api_config(merkle_tree_api_config);
        }
        self.node.add_layer(layer);
        Ok(self)
    }
```

## 5. Interactions with Other Components

### 5.1 Sequencer (Mempool I/O)

Transactions reach the State Keeper through the mempool: `MempoolFetcher` (`core/node/via_state_keeper/src/mempool_actor.rs`) syncs transactions from Postgres into the in-memory `MempoolGuard`, and `MempoolIO` (`core/node/via_state_keeper/src/io/mempool.rs`) implements `StateKeeperIO`, handing transactions to the keeper in order and providing batch parameters.

### 5.2 Execution Environment/VM

The Execution Environment (VM) executes transactions and produces state diffs. These diffs are then applied to the state by the State Keeper.

### 5.3 Prover

The Prover accesses the state to generate witnesses for zero-knowledge proofs. When the tree runs in full mode, `ZkSyncTree::process_l1_batch()` produces witness data as `WitnessInputMerklePaths` (the `witness` field of `TreeMetadata` in `core/lib/merkle_tree/src/domain.rs`), which is uploaded to the object store for witness generation.

### 5.4 L1 Commitment Process (Bitcoin)

Via's L1 is Bitcoin, not Ethereum. Batch commitments (including the state root produced by the metadata calculator) are published as Bitcoin inscriptions by the `via_btc_sender` component (`core/node/via_btc_sender`): `ViaAggregator` (in `aggregator.rs`) selects ready L1 batch operations, and `ViaBtcInscriptionAggregator` (in `btc_inscription_aggregator.rs`) turns them into inscription requests, with the inscription manager tracking confirmation on Bitcoin. Data availability blobs are sent to Celestia separately. The server wires the sender in as two layers:

```rust
// core/bin/via_server/src/node_builder.rs
    fn add_btc_sender_layer(mut self) -> anyhow::Result<Self> {
        let btc_sender_config = try_load_config!(self.configs.via_btc_sender_config);
        let wallet = self.wallets.btc_sender.clone().unwrap();
        self.node.add_layer(ViaBtcInscriptionAggregatorLayer::new(
            btc_sender_config.clone(),
            wallet.clone(),
        ));
        self.node.add_layer(ViaInscriptionManagerLayer::new(
            btc_sender_config,
            wallet.clone(),
        ));
        Ok(self)
    }
```

This allows for verification of state transitions against the commitments inscribed on Bitcoin.

## 6. Code Structure

### 6.1 Key Modules and Files

- **State Keeper**:
  - `core/node/via_state_keeper/src/keeper.rs`: Main state keeper implementation
  - `core/node/via_state_keeper/src/executor/`: Batch execution
  - `core/node/via_state_keeper/src/io/`: I/O operations
  - `core/node/via_state_keeper/src/seal_criteria/`: Sealing criteria
  - `core/node/via_state_keeper/src/updates/`: State updates
  - `core/node/via_state_keeper/src/mempool_actor.rs`: Mempool fetcher
  - `core/node/via_state_keeper/src/state_keeper_storage.rs`: Async RocksDB cache

- **State Commitment**:
  - `core/node/metadata_calculator/src/lib.rs`: Metadata calculator (feeds sealed batches into the tree)
  - `core/node/via_btc_sender/`: Commits batch metadata to Bitcoin via inscriptions

- **Merkle Tree**:
  - `core/lib/merkle_tree/src/lib.rs`: Core Merkle tree implementation
  - `core/lib/merkle_tree/src/domain.rs`: Domain-specific wrapper
  - `core/lib/merkle_tree/src/hasher/`: Hashing implementations
  - `core/lib/merkle_tree/src/storage/`: Storage implementations

- **Storage**:
  - `core/lib/state/src/lib.rs`: Core state storage abstractions
  - `core/lib/state/src/postgres/`: PostgreSQL storage
  - `core/lib/state/src/rocksdb/`: RocksDB storage
  - `core/lib/dal/src/storage_logs_dal.rs`: Data access layer

### 6.2 Key Data Structures

- **ZkSyncStateKeeper**: Main state keeper implementation
- **MerkleTree**: Generic Merkle tree implementation
- **ZkSyncTree**: Domain-specific Merkle tree wrapper
- **UpdatesManager**: Manages state updates during batch execution
- **StorageKey**: Represents a storage slot (address + key)
- **StorageLog**: Represents a change to a storage slot

## 7. Conclusion

The state management system in Via L2 is a complex but well-structured component that's responsible for maintaining, updating, and committing the rollup state. It uses a sparse Merkle tree to represent the state, which allows for efficient verification of state transitions and generation of proofs.

The system is designed to be efficient, scalable, and secure, with a focus on providing strong guarantees about the correctness of state transitions. It interacts with other components of the system, such as the Sequencer, VM, and Prover, to ensure that the state is maintained correctly and can be verified on L1.