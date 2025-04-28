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
- `core/node/via_state_keeper/src/batch_executor/`: Handles batch execution
- `core/node/via_state_keeper/src/io/`: Handles I/O operations for the state keeper
- `core/node/via_state_keeper/src/seal_criteria/`: Defines criteria for sealing L1 batches and L2 blocks
- `core/node/via_state_keeper/src/updates/`: Manages state updates

#### ZkSyncStateKeeper

The `ZkSyncStateKeeper` is the main implementation of the state keeper. It's responsible for:

1. Processing transactions and updating the state
2. Deciding when to seal L2 blocks and L1 batches
3. Handling protocol upgrades
4. Managing the execution of transactions in batches

```rust
pub struct ZkSyncStateKeeper {
    stop_receiver: watch::Receiver<bool>,
    io: Box<dyn StateKeeperIO>,
    output_handler: OutputHandler,
    batch_executor: Box<dyn ErasedBatchExecutor>,
    sealer: Arc<dyn ConditionalSealer>,
}
```

The state keeper operates as a state machine that processes incoming transactions, turning them into executed L2 blocks and L1 batches. It maintains the batch execution state in the `UpdatesManager` until the batch is sealed, at which point the changes are persisted by the `StateKeeperIO` implementation.

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

The `MerkleTree` is a generic implementation of a sparse Merkle tree, while `ZkSyncTree` is a domain-specific wrapper that ties the Merkle tree to the Via L2 system.

```rust
pub struct MerkleTree<DB, H = Blake2Hasher> {
    db: DB,
    hasher: H,
}

pub struct ZkSyncTree {
    tree: MerkleTree<Patched<RocksDBWrapper>>,
    thread_pool: Option<ThreadPool>,
    mode: TreeMode,
    pruning_enabled: bool,
}
```

The Merkle tree is implemented using AR16MT (a variant of the Jellyfish Merkle tree), which is a binary Merkle tree with radix-16 optimization for I/O efficiency. Each internal node may have up to 16 children, and paths of internal nodes that do not fork are removed to optimize storage.

#### Tree Hashing Specification

The tree is hashed as if it was a full binary Merkle tree with 2^256 leaves:

- Hash of a vacant leaf is `hash([0_u8; 40])`, where `hash` is Blake2s-256.
- Hash of an occupied leaf is `hash(u64::to_be_bytes(leaf_index) ++ value_hash)`, where `leaf_index` is a 1-based index of the leaf key.
- Hash of an internal node is `hash(left_child_hash ++ right_child_hash)`.

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
3. **ShadowStorage**: A wrapper or specialized storage implementation

```rust
pub use self::{
    cache::sequential_cache::SequentialCache,
    catchup::{AsyncCatchupTask, RocksdbCell},
    postgres::{PostgresStorage, PostgresStorageCaches, PostgresStorageCachesTask},
    rocksdb::{
        RocksdbStorage, RocksdbStorageBuilder, RocksdbStorageOptions, StateKeeperColumnFamily,
    },
    shadow_storage::ShadowStorage,
    storage_factory::{
        BatchDiff, OwnedStorage, PgOrRocksdbStorage, ReadStorageFactory, RocksdbWithMemory,
    },
};
```

#### Database Schema

The storage logs are stored in a database table with the following schema:

```sql
CREATE TABLE storage_logs (
    hashed_key BYTEA NOT NULL,
    address BYTEA,
    key BYTEA,
    value BYTEA NOT NULL,
    operation_number INTEGER NOT NULL,
    tx_hash BYTEA,
    miniblock_number BIGINT NOT NULL,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL
);
```

Each storage log entry contains:
- `hashed_key`: The hashed storage key
- `address`: The contract address
- `key`: The storage key
- `value`: The storage value
- `operation_number`: The operation number within the block
- `miniblock_number`: The L2 block number
- Other metadata fields

### 2.4 State Updates

State updates are managed by the `UpdatesManager`, which tracks changes to the state during batch execution.

#### Key Files and Modules

- `core/node/via_state_keeper/src/updates/`: Contains the `UpdatesManager` implementation

#### UpdatesManager

The `UpdatesManager` is responsible for tracking changes to the state during batch execution. It maintains:

1. The current L1 batch and L2 block information
2. The list of executed transactions
3. The storage writes deduplicator
4. The pending L1 gas count and execution metrics

When a transaction is executed, the `UpdatesManager` extends its state with the changes from the transaction. When a block or batch is sealed, the changes are persisted to the database.

## 3. State Representation

### 3.1 Storage Model

The state in Via L2 is represented as a key-value store, where:

- **Keys**: Storage slots, consisting of an address (20 bytes) and a key (32 bytes)
- **Values**: 32-byte values stored in those slots

Each storage slot is uniquely identified by its hashed key, which is computed as the Keccak-256 hash of the address and key.

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

The state keeper decides when to seal L2 blocks and L1 batches based on various criteria, such as:

- Gas limit
- Transaction count
- Time since the last seal
- Explicit sealing requests

When a block or batch is sealed, the changes are persisted to the database, and a new block or batch is started.

### 4.3 State Commitment

The state is committed to the Merkle tree after each L1 batch. The root hash of the Merkle tree serves as a cryptographic commitment to the state, which can be used for verification.

## 5. Interactions with Other Components

### 5.1 Sequencer

The Sequencer provides transactions to the State Keeper for execution. It's responsible for ordering transactions and ensuring they're executed in the correct order.

### 5.2 Execution Environment/VM

The Execution Environment (VM) executes transactions and produces state diffs. These diffs are then applied to the state by the State Keeper.

### 5.3 Prover

The Prover accesses the state to generate witnesses for zero-knowledge proofs. It uses the Merkle tree to generate proofs of state transitions.

### 5.4 L1 Commitment Process

The L1 commitment process uses the state root to commit the state to the L1 chain. This allows for verification of state transitions on L1.

## 6. Code Structure

### 6.1 Key Modules and Files

- **State Keeper**:
  - `core/node/via_state_keeper/src/keeper.rs`: Main state keeper implementation
  - `core/node/via_state_keeper/src/batch_executor/`: Batch execution
  - `core/node/via_state_keeper/src/io/`: I/O operations
  - `core/node/via_state_keeper/src/seal_criteria/`: Sealing criteria
  - `core/node/via_state_keeper/src/updates/`: State updates

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