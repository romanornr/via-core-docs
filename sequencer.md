# Via L2 Bitcoin ZK-Rollup: Sequencer Documentation

## Overview

The **Sequencer** is the component that processes transactions in the Via L2 Bitcoin ZK-Rollup system. It implements the L2 rollup protocol by receiving transactions from users and Bitcoin, executing them using specific ordering algorithms, and managing state transitions within the zkEVM. The sequencer coordinates with Bitcoin for data availability by grouping processed transactions into batches and submitting them as inscriptions to the Bitcoin blockchain.

The sequencer runs transaction ordering and execution algorithms that process L2 transactions for immediate finality while aggregating them into larger batches for efficient Bitcoin submission. It manages the complete transaction lifecycle from mempool ingestion through VM execution to Bitcoin inscription, implementing the core rollup protocol that enables scalable transaction processing while maintaining Bitcoin's security guarantees.

## Core Responsibilities

### 1. Transaction Processing
- **Ingestion**: Receives transactions from users (L2 transactions) and from Bitcoin (L1 transactions)
- **Validation**: Ensures transactions meet basic validity requirements (sufficient fees, correct nonces, etc.)
- **Ordering**: Prioritizes and sequences transactions for execution
- **Execution**: Runs transactions in the zkEVM to update the L2 state

### 2. Block and Batch Management
- **L2 Block Creation**: Groups executed transactions into L2 blocks for immediate finality
- **L1 Batch Creation**: Aggregates multiple L2 blocks into larger L1 batches for efficient Bitcoin submission
- **Sealing Logic**: Determines when blocks and batches are "sealed" (finalized) based on various criteria

### 3. Bitcoin Integration
- **Data Preparation**: Formats batch data for Bitcoin inscription
- **Inscription Management**: Coordinates the process of writing batch data to Bitcoin
- **Finality Tracking**: Monitors Bitcoin confirmations to ensure data availability

## Transaction Lifecycle: From Submission to Bitcoin

### Phase 1: Transaction Ingestion and Mempool Management

```
User/L1 → Mempool → Priority Queue → State Keeper
```

1. **Transaction Arrival**: Transactions enter the system through the RPC layer
2. **Mempool Storage**: Transactions are stored in the [`MempoolStore`](core/lib/mempool/src/mempool_store.rs:23) which maintains:
   - L1 transactions in a simple HashMap by priority ID
   - L2 transactions grouped by account address for nonce ordering
   - A global priority queue ([`BTreeSet<MempoolScore>`](core/lib/mempool/src/mempool_store.rs:29)) for efficient retrieval

3. **Transaction Prioritization**: The mempool orders transactions based on:
   - **Priority**: L1 transactions always have higher priority than L2 transactions
   - **Gas Price**: Higher gas prices get priority within the same transaction type
   - **Nonce Ordering**: Transactions from the same account must be processed in nonce order
   - **Age**: Older transactions get slight priority to prevent starvation

### Phase 2: State Keeping and Execution

```
Mempool → State Keeper → Batch Executor → VM → State Updates
```

The [`ZkSyncStateKeeper`](core/node/state_keeper/src/keeper.rs:94) is the central orchestrator that:

1. **Batch Initialization**: Creates a new L1 batch or restores a pending one
2. **Transaction Fetching**: Requests transactions from the mempool via [`MempoolIO`](core/node/state_keeper/src/io/mempool.rs)
3. **Execution Loop**: For each transaction:
   - Validates the transaction can be executed
   - Passes it to the [`BatchExecutor`](core/node/state_keeper/src/batch_executor.rs) 
   - Receives execution results and updates the state
   - Checks sealing criteria to determine if blocks/batches should be finalized

4. **State Management**: The [`UpdatesManager`](core/node/state_keeper/src/updates.rs) tracks all state changes until they're persisted

### Phase 3: L2 Block Sealing

```
Executed Transactions → Seal Criteria Check → L2 Block Creation
```

L2 blocks provide immediate finality for users and are sealed based on:

- **Time-based Sealing** ([`TimeoutSealer`](core/node/state_keeper/src/seal_criteria/timeout.rs)): Blocks are sealed after `l2_block_commit_deadline_ms` milliseconds
- **Size-based Sealing** ([`L2BlockMaxPayloadSizeSealer`](core/node/state_keeper/src/seal_criteria/l2_block_max_payload_size.rs)): Blocks are sealed when they reach `l2_block_max_payload_size` bytes

This dual approach ensures both responsiveness (users get quick confirmation) and efficiency (blocks aren't wastefully small).

### Phase 4: L1 Batch Sealing

```
Multiple L2 Blocks → Seal Criteria Check → L1 Batch Creation
```

L1 batches aggregate multiple L2 blocks for efficient Bitcoin submission. The [`SequencerSealer`](core/node/state_keeper/src/seal_criteria/conditional_sealer.rs) evaluates multiple criteria:

- **Transaction Count** ([`SlotsCriterion`](core/node/state_keeper/src/seal_criteria/criteria/slots.rs)): Maximum number of transactions per batch
- **Gas Usage** ([`GasCriterion`](core/node/state_keeper/src/seal_criteria/criteria/gas.rs)): Maximum gas consumed per batch  
- **Pubdata Size** ([`PubdataBytesCriterion`](core/node/state_keeper/src/seal_criteria/criteria/pubdata_bytes.rs)): Maximum public data size
- **Circuit Constraints** ([`CircuitsCriterion`](core/node/state_keeper/src/seal_criteria/criteria/circuits.rs)): Proving system limitations
- **Encoding Size** ([`TxEncodingSizeCriterion`](core/node/state_keeper/src/seal_criteria/criteria/tx_encoding_size.rs)): Maximum encoded transaction size
- **Time Deadline** ([`TimeoutCriterion`](core/node/state_keeper/src/seal_criteria/criteria/timeout.rs)): Maximum batch open time

The batch is sealed when **any** criterion is met, ensuring optimal resource utilization while respecting system constraints.

### Phase 5: Bitcoin Inscription Process

```
Sealed L1 Batch → Aggregator → Inscription Manager → Bitcoin
```

The Bitcoin integration happens through several specialized components:

1. **Batch Aggregation** ([`ViaAggregator`](core/node/via_btc_sender/src/aggregator.rs:20)): 
   - Collects sealed L1 batches that are ready for Bitcoin submission
   - Applies publish criteria to determine when to create inscriptions
   - Supports two operation types: batch commits and proof commits

2. **Inscription Creation** ([`ViaBtcInscriptionAggregator`](core/node/via_btc_sender/src/btc_inscription_aggregator.rs)):
   - Formats batch data into Bitcoin inscription messages
   - Handles data compression and encoding for efficient Bitcoin storage

3. **Inscription Management** ([`ViaBtcInscriptionManager`](core/node/via_btc_sender/src/btc_inscription_manager.rs)):
   - Coordinates the actual Bitcoin transaction creation and submission
   - Monitors inscription confirmations on Bitcoin
   - Handles retry logic for failed inscriptions

## Key Components Deep Dive

### State Keeper: The Central Orchestrator

The [`ZkSyncStateKeeper`](core/node/state_keeper/src/keeper.rs:94) is the main sequencer component that coordinates all other parts:

```rust
// Simplified state keeper loop
loop {
    // Get next transaction from mempool
    let tx = mempool_io.wait_for_next_tx().await?;
    
    // Execute transaction in VM
    let result = batch_executor.execute_tx(tx).await?;
    
    // Update state and check sealing criteria
    updates_manager.extend_from_executed_tx(result);
    
    // Seal L2 block if criteria met
    if l2_block_sealer.should_seal_l2_block() {
        seal_l2_block();
    }
    
    // Seal L1 batch if criteria met  
    if l1_batch_sealer.should_seal_l1_batch() {
        seal_l1_batch();
        break; // Start new batch
    }
}
```

### Mempool: Transaction Storage and Prioritization

The [`MempoolStore`](core/lib/mempool/src/mempool_store.rs:23) efficiently manages pending transactions:

- **L1 Transactions**: Stored by priority ID for simple FIFO processing
- **L2 Transactions**: Grouped by account address to maintain nonce ordering
- **Priority Queue**: Global ordering for efficient transaction selection
- **Capacity Management**: Prevents memory exhaustion with configurable limits

### Batch Executor: VM Interface

The [`BatchExecutor`](core/node/state_keeper/src/batch_executor.rs) abstracts the zkEVM interaction:

- **VM Initialization**: Sets up the virtual machine for batch execution
- **Transaction Execution**: Runs individual transactions and collects results
- **State Tracking**: Manages storage reads/writes and gas consumption
- **Error Handling**: Deals with transaction failures and VM halts

### Seal Criteria: Decision Making Logic

The sealing system uses a modular approach where different criteria can be combined:

- **Conditional Sealing**: Multiple criteria evaluated together
- **Priority-based**: Some criteria (like timeouts) can override others
- **Configurable**: Operators can tune sealing behavior for their needs

## L2 Blocks vs L1 Batches: Understanding the Distinction

### L2 Blocks
- **Purpose**: Provide immediate finality and user feedback
- **Size**: Typically small (few transactions to few hundred)
- **Frequency**: Created every few seconds
- **Finality**: Soft finality - can be reorganized if L1 batch fails
- **Use Case**: User-facing confirmations, wallet updates, dApp interactions

### L1 Batches  
- **Purpose**: Efficient Bitcoin submission and ultimate finality
- **Size**: Large (thousands of transactions across multiple L2 blocks)
- **Frequency**: Created every few minutes to hours
- **Finality**: Hard finality once confirmed on Bitcoin
- **Use Case**: Security, data availability, dispute resolution

This two-tier approach optimizes for both user experience (fast L2 blocks) and cost efficiency (large L1 batches).

## Sealing Logic and Criteria

### L2 Block Sealing Strategy

L2 blocks use a simple dual-criteria approach:

1. **Time-based**: Ensures users get confirmations within a reasonable time
2. **Size-based**: Prevents blocks from becoming too large for efficient processing

### L1 Batch Sealing Strategy

L1 batches use a sophisticated multi-criteria system:

```rust
// Simplified sealing logic
if slots_criterion.should_seal(seal_data) ||
   gas_criterion.should_seal(seal_data) ||
   pubdata_criterion.should_seal(seal_data) ||
   circuits_criterion.should_seal(seal_data) ||
   encoding_criterion.should_seal(seal_data) ||
   timeout_criterion.should_seal(seal_data) {
    seal_batch();
}
```

This ensures batches are sealed when any resource limit is approached, optimizing for:
- **Proving efficiency**: Staying within circuit constraints
- **Bitcoin costs**: Maximizing data per inscription
- **User experience**: Not keeping transactions pending too long

## Bitcoin Inscription Process

### Aggregation Strategy

The [`ViaAggregator`](core/node/via_btc_sender/src/aggregator.rs:20) implements a sophisticated batching strategy:

1. **Commit Operations**: Submit L1 batch data to Bitcoin
2. **Proof Operations**: Submit validity proofs to Bitcoin
3. **Publish Criteria**: Determine optimal timing for inscriptions

### Inscription Types

Two types of operations are inscribed to Bitcoin:

1. **CommitL1BatchOnchain**: Contains the actual transaction data and state roots
2. **CommitProofOnchain**: Contains zero-knowledge proofs validating the batches

### Publish Criteria

Inscriptions are triggered by:

- **Number Criterion** ([`ViaNumberCriterion`](core/node/via_btc_sender/src/publish_criterion.rs)): When enough batches/proofs accumulate
- **Timestamp Deadline** ([`TimestampDeadlineCriterion`](core/node/via_btc_sender/src/publish_criterion.rs)): When batches become too old

This dual approach balances Bitcoin transaction costs with data availability requirements.

## Configuration and Tuning

### State Keeper Configuration

Key parameters that affect sequencer behavior:

```yaml
# L2 Block timing
l2_block_commit_deadline_ms: 2500  # Max time for L2 block

# L1 Batch constraints  
block_commit_deadline_ms: 900000   # Max time for L1 batch
max_pubdata_per_batch: 120000      # Max public data size
validation_computational_gas_limit: 300000000  # Gas limit

# Resource limits
l2_block_max_payload_size: 750000  # Max L2 block size
```

### Mempool Configuration

```yaml
# Capacity management
capacity: 10000000                 # Max transactions in mempool
sync_interval_ms: 10               # Database sync frequency
sync_batch_size: 1000             # Transactions per sync
```

### BTC Sender Configuration

```yaml
# Aggregation limits
max_aggregated_blocks_to_commit: 50    # Batches per inscription
max_aggregated_proofs_to_commit: 20    # Proofs per inscription

# Timing
poll_interval: 1000                    # Check frequency (ms)
block_confirmations: 1                 # Bitcoin confirmations needed
```

## Performance Considerations

### Throughput Optimization

- **Parallel Processing**: Multiple components work concurrently
- **Batch Sizing**: Larger batches improve Bitcoin efficiency
- **Mempool Management**: Efficient transaction selection and ordering

### Latency Optimization  

- **L2 Block Frequency**: Fast sealing for user responsiveness
- **Predictive Sealing**: Anticipating resource limits
- **Async Processing**: Non-blocking operations where possible

### Resource Management

- **Memory Usage**: Bounded mempool and state tracking
- **CPU Usage**: Efficient VM execution and proof generation
- **Bitcoin Costs**: Optimized inscription timing and sizing

## Monitoring and Observability

The sequencer exposes comprehensive metrics through:

- **KEEPER_METRICS**: State keeper performance and health
- **L1_BATCH_METRICS**: Batch creation and sealing statistics  
- **AGGREGATION_METRICS**: Bitcoin submission metrics

Key metrics to monitor:

- Transaction processing rate
- Block and batch sealing frequency
- Mempool size and growth
- Bitcoin inscription success rate
- VM execution performance

## Error Handling and Recovery

### Transaction Failures

- **Invalid Transactions**: Rejected and removed from mempool
- **VM Halts**: Transaction reverted, state unchanged
- **Nonce Gaps**: Transactions stashed until gap filled

### System Failures

- **State Recovery**: Restore from last committed state
- **Mempool Rebuild**: Reload pending transactions from database
- **Bitcoin Retry**: Automatic retry of failed inscriptions

### Monitoring and Alerts

- **Health Checks**: Regular system health validation
- **Performance Alerts**: Warnings for degraded performance
- **Error Tracking**: Comprehensive error logging and metrics

## Integration with Other Components

### RPC Layer Integration

- **Transaction Submission**: Receives transactions from users
- **Status Queries**: Provides transaction and block status
- **State Queries**: Serves current L2 state information

### Prover Integration

- **Proof Requests**: Triggers proof generation for sealed batches
- **Proof Verification**: Validates proofs before Bitcoin submission
- **Circuit Constraints**: Respects proving system limitations

### Bridge Integration

- **L1→L2 Transactions**: Processes deposits from Bitcoin
- **L2→L1 Transactions**: Handles withdrawal requests
- **State Synchronization**: Maintains consistency across layers

## Conclusion

The Via L2 sequencer represents a sophisticated orchestration system that balances multiple competing requirements: user experience, cost efficiency, security, and scalability. Its modular design allows for fine-tuning of each component while maintaining the overall system's integrity and performance.

Understanding the sequencer is crucial for:
- **Operators**: Tuning performance and managing costs
- **Developers**: Building applications that work optimally with the system
- **Integrators**: Understanding transaction lifecycle and finality guarantees
- **Researchers**: Analyzing and improving L2 rollup architectures

The sequencer's success lies in its ability to provide Ethereum-like user experience while leveraging Bitcoin's security and decentralization, making it a critical component in the broader blockchain scaling landscape.