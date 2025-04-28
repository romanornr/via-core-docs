# Via L2 Bitcoin ZK-Rollup: Sequencer Documentation

## 1. Introduction

The Sequencer is a critical component of the Via L2 Bitcoin ZK-Rollup system responsible for processing transactions, creating L2 blocks, and aggregating them into L1 batches that are eventually committed to Bitcoin. The Sequencer ensures transaction ordering, execution, and state transitions are performed correctly and efficiently.

## 2. Architecture and Components

The Sequencer functionality in Via is distributed across several interconnected components:

### 2.1 Core Components

- **State Keeper**: The central component that processes transactions, manages state transitions, and creates L2 blocks and L1 batches.
- **Mempool**: Stores pending transactions and provides them to the State Keeper for execution.
- **Batch Executor**: Executes transactions within the VM and manages state updates.
- **Seal Criteria**: Determines when to seal L2 blocks and L1 batches based on various conditions.
- **BTC Sender**: Responsible for submitting batch data to Bitcoin through inscriptions.

### 2.2 File Structure

The Sequencer functionality is primarily implemented in the following directories:

- `core/node/state_keeper/`: Generic state keeper implementation
- `core/node/via_state_keeper/`: Via-specific state keeper implementation
- `core/lib/mempool/`: Mempool implementation for transaction storage and retrieval
- `core/node/via_btc_sender/`: Components for sending batch data to Bitcoin

## 3. Transaction Processing Flow

### 3.1 Transaction Ingestion

Transactions enter the system through the mempool, which is implemented in `core/lib/mempool/src/mempool_store.rs`. The mempool:

1. Stores pending L1 and L2 transactions
2. Maintains a priority queue for transaction ordering
3. Filters transactions based on fee requirements
4. Provides transactions to the State Keeper when requested

The `MempoolFetcher` class in `core/node/state_keeper/src/mempool_actor.rs` synchronizes transactions between the database and the in-memory mempool.

### 3.2 Transaction Execution

The State Keeper, implemented in `core/node/state_keeper/src/keeper.rs` and `core/node/via_state_keeper/src/keeper.rs`, is responsible for:

1. Requesting transactions from the mempool
2. Executing transactions using the Batch Executor
3. Managing state updates
4. Creating and sealing L2 blocks and L1 batches

The execution flow is managed by the `run_inner` method in `ZkSyncStateKeeper`, which:

1. Initializes a new batch or restores a pending batch
2. Processes transactions in a loop until a batch is sealed
3. Finalizes the batch and starts a new one

### 3.3 Transaction Ordering

Transactions are ordered based on:

1. Priority (L1 transactions have higher priority than L2 transactions)
2. Gas price and other economic factors
3. Nonce ordering for transactions from the same account

The `MempoolStore.next_transaction` method in `core/lib/mempool/src/mempool_store.rs` implements this ordering logic.

## 4. Batch Creation and Sealing

### 4.1 L2 Block Creation

L2 blocks are created and sealed based on criteria defined in `core/node/state_keeper/src/seal_criteria/mod.rs`:

1. **Time-based sealing**: L2 blocks are sealed after a configurable time interval (`l2_block_commit_deadline_ms`)
2. **Size-based sealing**: L2 blocks are sealed when they reach a maximum payload size (`l2_block_max_payload_size`)

The `TimeoutSealer` and `L2BlockMaxPayloadSizeSealer` classes implement these criteria.

### 4.2 L1 Batch Creation

L1 batches aggregate multiple L2 blocks and are sealed based on criteria defined in `core/node/state_keeper/src/seal_criteria/conditional_sealer.rs`:

1. **Slots criterion**: Maximum number of transactions in a batch
2. **Gas criterion**: Maximum gas used in a batch
3. **Pubdata bytes criterion**: Maximum pubdata size in a batch
4. **Circuits criterion**: Constraints related to proving circuit capacity
5. **Transaction encoding size criterion**: Maximum encoding size for transactions
6. **Time-based criterion**: Maximum time a batch can remain open

The `SequencerSealer` class combines these criteria to determine when to seal a batch.

## 5. Bitcoin Integration

### 5.1 Batch Submission to Bitcoin

The Via L2 system submits batch data to Bitcoin through inscriptions, implemented in `core/node/via_btc_sender/`:

1. `ViaAggregator` in `aggregator.rs` aggregates L1 batches for submission
2. `ViaBtcInscriptionAggregator` in `btc_inscription_aggregator.rs` creates inscription messages
3. `ViaBtcInscriptionManager` in `btc_inscription_manager.rs` manages the inscription process

### 5.2 Inscription Types

Two types of inscriptions are supported, defined in `ViaAggregatedOperation` in `aggregated_operations.rs`:

1. **CommitL1BatchOnchain**: Commits L1 batch data to Bitcoin
2. **CommitProofOnchain**: Commits proof data to Bitcoin

### 5.3 Publish Criteria

The system decides when to publish batches to Bitcoin based on criteria in `publish_criterion.rs`:

1. **ViaNumberCriterion**: Publishes when a certain number of batches are ready
2. **TimestampDeadlineCriterion**: Publishes when batches reach a certain age

## 6. Interactions with Other Components

### 6.1 Mempool Interaction

The State Keeper interacts with the mempool through the `MempoolIO` class in `core/node/state_keeper/src/io/mempool.rs`, which:

1. Retrieves transactions from the mempool
2. Provides batch parameters
3. Handles transaction rejections and rollbacks

### 6.2 Storage Interaction

The State Keeper interacts with the database through:

1. `StateKeeperPersistence` for persisting state updates
2. `L1BatchParamsProvider` for retrieving batch parameters
3. Various DAL (Data Access Layer) classes for specific database operations

### 6.3 VM Interaction

The State Keeper interacts with the VM through the `BatchExecutor` interface, which:

1. Initializes the VM for batch execution
2. Executes transactions
3. Provides execution results and metrics

## 7. Key Data Structures and Algorithms

### 7.1 Data Structures

- **MempoolStore**: Stores pending transactions and provides transaction ordering
- **UpdatesManager**: Manages state updates during batch execution
- **SealData**: Contains metrics used for sealing decisions
- **L1BatchParams** and **L2BlockParams**: Parameters for batch and block creation
- **ViaAggregatedOperation**: Represents operations to be submitted to Bitcoin

### 7.2 Algorithms

- **Transaction filtering**: Filters transactions based on fee requirements and other criteria
- **Batch sealing**: Determines when to seal batches based on multiple criteria
- **Transaction ordering**: Orders transactions based on priority, gas price, and nonce
- **Batch aggregation**: Aggregates batches for submission to Bitcoin

## 8. Configuration Parameters

The Sequencer behavior is controlled by various configuration parameters:

### 8.1 State Keeper Configuration

- `block_commit_deadline_ms`: Maximum time a batch can remain open
- `l2_block_commit_deadline_ms`: Maximum time an L2 block can remain open
- `max_pubdata_per_batch`: Maximum pubdata size in a batch
- `validation_computational_gas_limit`: Gas limit for batch validation
- `l2_block_max_payload_size`: Maximum payload size for L2 blocks

### 8.2 Mempool Configuration

- `capacity`: Maximum number of transactions in the mempool
- `sync_interval_ms`: Interval for syncing transactions from the database
- `sync_batch_size`: Number of transactions to sync at once

### 8.3 BTC Sender Configuration

- `max_aggregated_blocks_to_commit`: Maximum number of blocks to aggregate for commit
- `max_aggregated_proofs_to_commit`: Maximum number of proofs to aggregate for commit
- `poll_interval`: Interval for checking for new batches to submit
- `max_txs_in_flight`: Maximum number of transactions in flight
- `block_confirmations`: Number of confirmations required for a transaction

## 9. Conclusion

The Sequencer is a critical component of the Via L2 Bitcoin ZK-Rollup system, responsible for transaction processing, state management, and batch submission to Bitcoin. Its modular design allows for flexibility in transaction ordering, batch creation, and submission strategies, while ensuring the system's security and efficiency.