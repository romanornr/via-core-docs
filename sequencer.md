# Via L2 Bitcoin ZK-Rollup: Sequencer Documentation

## Overview

The sequencer orders pending transactions from the mempool, applies admission checks (signatures, nonce, gas/fee limits and protocol restraints) and executes them on the zkEVM which enforces signatures, nonce, intrinsic validity and state transition rules. 

As execution proceeds, the sequencer periodically seals executed transactions into small, frequent L2 blocks (called 'miniblocks' in DB tables). A contiguous sequence of L2 blocks is then aggregated into an L1 batch, which is the unit published on L1.

Sealing an L2 block is local to L2 and conveys only soft finality, hard finality is obtained once the batch commitment and DA blobs are confirmed on Bitcoin. State-correctness finality is reached after the batch's zk-proof is generated, committed to Bitcoin and verified by the verifier network.

Batches are sealed when configured criteria are met (e.g., transaction slot, gas usage, pubdata size, transaction-encoding size, together with I/O-level thresholds such as payload-encoding size and timeouts).

Publication to the Data Availability Layer (Celestia) and inscriptions of batch/proof commitments on Bitcoin are handled by downstream components. The sequencer does not post to DA or inscribe commitments/proofs on Bitcoin. It outputs ordered, executed L2 blocks and sealed L1 batches that downstream components use for DA publications, proof generation and commitment posting.

### Transaction processing loop

The state keeper runs a per-batch loop that:
1. Check whether the open L1 batch must be sealed due to an unconditional timeout; if so, stop processing and proceed to finalization.
2. If I/O thresholds indicate the current L2 block should be sealed (e.g., timeout or payload-encoding size), seal it and initialize the next L2 block.
3. Wait for the next transaction, execute it in the VM, and update the in-memory batch and block state.
4. Evaluate the batch-level sealing criteria (transaction slots, gas usage, pubdata size, and transaction-encoding size). If sealing is triggered, apply the required action (e.g., include or exclude the last transaction as dictated by the criteria).
5. When sealing is triggered, finish the batch, persist it to storage, and roll over to a new batch environment.

Full control flow is implemented in the state keeper; see [core/node/via_state_keeper/src/keeper.rs](https://github.com/vianetwork/via-core/blob/main/core/node/via_state_keeper/src/keeper.rs)


### Sealing criteria (batch level)

Batch sealing is evaluated after each executed transaction. Independent criteria are applied to the current batch state and the just-executed transaction. The most restrictive outcomes wins ("unexecutable" > "exclude-and-seal" > "include-and-seal" > "no-seal").

- Transaction slots
   - Purpose: cap the number of transactions in a batch. 
   - Behavior: When the configured slot limit is reached, include the last executed transaction and seal the batch ("include-and-seal").

- Gas usage tresholds
   - Inputs: cumulative batch gas (commit/prove/execute), gas attributable to the last tx and configured gas bounds.
   - Behavior: 
      - If the last tx's gas (without required overhead) exceeds the per-transaction reject bound, it is "unexecutable".
      - If including the last tx would push the batch above the hard per-batch gas limit, seal the battch without this tx ("exclude-and-seal").
      - If, after applying the just-executed (last) transaction, the cumulative batch gas crosses the configured "close" threshold (still below the hard per-batch limit), include that last executed transaction and seal the batch ("include-and-seal").
      - Otherwise, continue ("no-seal").

- Pubdata size thresholds
   - Inputs: the cumulative batch pubdata size before applying the just-executed transaction (the "last transaction"), the pubdata attributable to that last transaction, the required bootloader/batch-tip overhead, and the configured pubdata bounds (per-transaction reject bound, batch "close" threshold, and batch hard limit).
   - Behavior:
      - If the last transaction's effective pubdata (its own pubdata plus required overhead) exceeds the per-transaction reject bound ⇒ resolution = "unexecutable".
      - Otherwise, compute the prospective cumulative batch pubdata after including the last transaction. If that prospective size would exed the hard batch pubdata limit ⇒ resolution = "exclude-and-seal".
      - Otherwise, if the the prospective cumulative size is at or above the configured "close" treshold while the still not exceeding the hard limit ⇒ resolution = "include-and-seal" (keep the last transaction and seal the batch)
      - Otherwise, resolution = "no-seal" (keep processing)

- Transaction-encoding size tresholds
   - Inputs: The cumulative encoded size already used within the bootloader's encoding space before applying the last transaction, the encoded size of the last transaction and the configured geometry percentages (per-tx reject bound, "close", treshold and total capacity).
   - Behavior:
      - If the last transaction's encoded size exeeds the per transaction reject bound ⇒ resolution ⇒ "unexecutable".
      - Otherwise, compute the prospective cumulative encoded size after including the last transaction. If that prospective size would exceed the bootloader's total capacity ⇒ resolution ⇒ "exclude-and-seal" (do not keep the last transaction; seal the batch).
      - Otherwise, if the prospective cumulative size is at or above the configured "close" treshold while still not exceeding the capacity ⇒ resolution = "include-and-seal" (keep the last transaction and seal the batch).
      - Otherwise, resolution = "no-seal" (keep processing)


Decision combination
 * Each criterion yields a resolution of the current step. The final decision is the most restrictive among them, ordered: "unexecutable" > "exclude-and-seal" > "include-and-seal" > "no-seal"
 * Actions implied by the final decision (always referring to the just-executed, last transaction):
   * "include and seal": keep the last transaction in the batch and seal the batch immediately.
   * "exclude and seal": roll back the last transaction from the batch (do not include it), then seeal the batch immediately.
   * "unexecutable": reject the last transaction outright depending on other criteria in the same step. The batch may or may not seal. 
   * "no-seal": keep teh last transaction in batch and continue processing. the next transaction. 


### L2 block cadence (I/O tresholds)

L2 block sealing (mini block cadence) is determined by the I/O tresholds that operate independently from the batch-level sealing criteria. These tresholds rotate L2 blocks within the same open batch unless the seperate batch-timeout rule triggers. 

   * Timeout-based L2 block sealing:
      * Preconditions: the current L2 block contains at least one executed transaction. 
      * Condition: the elapsed time since the L2 block's open timestamp exceeds the configured L2-block deadline. 
      * Action: seal the L2 block and immediately start the enxt L2 block within the same batch L2 batch (with a fresh timestamp).
      * Effect: miniblock remains small and frequent. The batch stays open for longer.

   * Maximum L2 block payload-encoding size:
      * Inputs: the cumulative encoded payload size of all transactions currently included in the L2 block.
      * Condition: the L2 block's cumulative payload-encoding size meets or exceeds the configured limit (`l2_block_max_payload_size`).
      * Action: seal the L2 block and start the next one within the same batch.

Both live at the IO level (`IoSealCriteria` in `core/node/via_state_keeper/src/seal_criteria/mod.rs`). The batch-level unconditional timeout is the `TimeoutSealer` in the same module, driven by `block_commit_deadline_ms`; it implements `should_seal_l1_batch_unconditionally` and never seals an empty batch.

### Downstream of the sequencer

The Sequencer is implemented by `ZkSyncStateKeeper` (`core/node/via_state_keeper/src/keeper.rs`), which pulls transactions from the mempool IO (`io/mempool.rs`), executes them in the zkEVM, and assembles the results into L2 blocks and L1 batches per the sealing criteria (`seal_criteria/conditional_sealer.rs`; sealing background in `docs/specs/blocks_batches.md`). Once a batch is sealed, downstream components take over: the DA dispatcher sends pubdata and proofs to Celestia (`CelestiaClient`, wired in the via_server node builder), and the BTC sender inscribes the batch/proof commitments referencing the DA blobs (`core/node/via_btc_sender/src/btc_inscription_manager.rs`; protocol in `core/lib/via_btc_client/DEV.md`). That is the default split: DA on Celestia, settlement anchoring on Bitcoin.


**Data Availability and Settlement Architecture**: Via L2 uses a dual-layer approach for data availability and settlement:
- **Data Availability**: Celestia network stores batch data and proofs as blobs with inclusion proofs
- **Settlement Anchoring**: Bitcoin inscriptions contain batch commitments and references to Celestia blob IDs, providing final settlement anchoring on Bitcoin's immutable ledger

## Build Variants

Via Core supports two deployment modes:
- **via_server** ([`core/bin/via_server/src/node_builder.rs`](core/bin/via_server/src/node_builder.rs:32)): Bitcoin settlement + Celestia DA (Via L2's target architecture)
- **zksync_server** ([`core/bin/zksync_server/src/node_builder.rs`](core/bin/zksync_server/src/node_builder.rs:32)): Ethereum L1 contracts (legacy/zkSync-compatible mode)

This documentation focuses on the Via L2 Bitcoin + Celestia configuration.

## Core Components

### State Keeper: The Central Orchestrator

The [`ZkSyncStateKeeper`](core/node/via_state_keeper/src/keeper.rs:68) is the main sequencer component that coordinates all transaction processing:

```rust
// core/node/via_state_keeper/src/keeper.rs
#[derive(Debug)]
pub struct ZkSyncStateKeeper {
    io: Box<dyn StateKeeperIO>,
    output_handler: OutputHandler,
    batch_executor: Box<dyn BatchExecutorFactory<OwnedStorage>>,
    sealer: Arc<dyn ConditionalSealer>,
    storage_factory: Arc<dyn ReadStorageFactory>,
    health_updater: HealthUpdater,
}
```

- **Initialization**: [`run_inner()`](core/node/via_state_keeper/src/keeper.rs:107) loads pending batches or starts new ones
- **Transaction Processing**: [`process_l1_batch()`](core/node/via_state_keeper/src/keeper.rs:562) executes transactions in a loop
- **State Recovery**: [`restore_state()`](core/node/via_state_keeper/src/keeper.rs:461) replays pending transactions after restart
- **L2 Block Management**: [`seal_l2_block()`](core/node/via_state_keeper/src/keeper.rs:440) and [`start_next_l2_block()`](core/node/via_state_keeper/src/keeper.rs:418)

### IO Layer and Mempool Integration

- **State Keeper IO**: [`StateKeeperIO`](core/node/via_state_keeper/src/io/mod.rs:115) abstraction for transaction sourcing and batch parameters
- **Mempool IO**: [`MempoolIO`](core/node/via_state_keeper/src/io/mempool.rs:48) implementation that feeds transactions from mempool with sealing criteria
- **Persistence**: [`StateKeeperPersistence`](core/node/via_state_keeper/src/io/persistence.rs:29) handles L2 block sealing commands and database persistence

The mempool IO used by the main server:

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

Its `IoSealCriteria` implementation is what drives the L2 block cadence described above (timeout plus max payload size) and the unconditional batch timeout:

```rust
// core/node/via_state_keeper/src/io/mempool.rs
impl IoSealCriteria for MempoolIO {
    fn should_seal_l1_batch_unconditionally(&mut self, manager: &UpdatesManager) -> bool {
        self.timeout_sealer
            .should_seal_l1_batch_unconditionally(manager)
    }

    fn should_seal_l2_block(&mut self, manager: &UpdatesManager) -> bool {
        if self.timeout_sealer.should_seal_l2_block(manager) {
            AGGREGATION_METRICS.l2_block_reason_inc(&L2BlockSealReason::Timeout);
            return true;
        }

        if self
            .l2_block_max_payload_size_sealer
            .should_seal_l2_block(manager)
        {
            AGGREGATION_METRICS.l2_block_reason_inc(&L2BlockSealReason::PayloadSize);
            return true;
        }

        false
    }
}
```

### Batch Execution and VM Integration

- **Batch Executor**: [`BatchExecutor`](core/lib/vm_interface/src/executor.rs:32) and its factory [`BatchExecutorFactory`](core/lib/vm_interface/src/executor.rs:16) abstraction for VM interaction (there is no `batch_executor.rs` inside `via_state_keeper`; the traits live in `zksync_vm_interface`)
- **Main Batch Executor**: [`MainBatchExecutorFactory`](core/lib/vm_executor/src/batch/factory.rs:65) coordinates VM execution in a separate thread (`init_batch()` spawns the VM loop with `tokio::task::spawn_blocking`); it is re-exported by [`core/node/via_state_keeper/src/executor/mod.rs`](core/node/via_state_keeper/src/executor/mod.rs:6)
- **Updates Manager**: [`UpdatesManager`](core/node/via_state_keeper/src/updates/mod.rs:30) accumulates state changes during batch execution

The executor traits the state keeper is programmed against:

```rust
// core/lib/vm_interface/src/executor.rs
/// Factory of [`BatchExecutor`]s.
pub trait BatchExecutorFactory<S: Send + 'static>: 'static + Send + fmt::Debug {
    /// Initializes an executor for a batch with the specified params and using the provided storage.
    fn init_batch(
        &mut self,
        storage: S,
        l1_batch_params: L1BatchEnv,
        system_env: SystemEnv,
        pubdata_params: PubdataParams,
    ) -> Box<dyn BatchExecutor<S>>;
}

/// Handle for executing a single L1 batch.
///
/// The handle is parametric by the transaction execution output in order to be able to represent different
/// levels of abstraction.
#[async_trait]
pub trait BatchExecutor<S>: 'static + Send + fmt::Debug {
    /// Executes a transaction.
    async fn execute_tx(
        &mut self,
        tx: Transaction,
    ) -> anyhow::Result<BatchTransactionExecutionResult>;

    /// Rolls back the last executed transaction.
    async fn rollback_last_tx(&mut self) -> anyhow::Result<()>;

    /// Starts a next L2 block with the specified params.
    async fn start_next_l2_block(&mut self, env: L2BlockEnv) -> anyhow::Result<()>;

    /// Finished the current L1 batch.
    async fn finish_batch(self: Box<Self>) -> anyhow::Result<(FinishedL1Batch, StorageView<S>)>;
}
```

The in-memory accumulator for the open batch and block:

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

### Sealing Criteria System

- **Conditional Sealer**: [`ConditionalSealer`](core/node/via_state_keeper/src/seal_criteria/conditional_sealer.rs:15) interface for sealing decisions
- **Sequencer Sealer**: [`SequencerSealer`](core/node/via_state_keeper/src/seal_criteria/conditional_sealer.rs:45) aggregates the registered criteria
- **Registered criteria** (`default_sealers()` in `conditional_sealer.rs`):

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

- **Decision combination**: `SequencerSealer::should_seal_l1_batch()` folds every criterion's resolution with `SealResolution::stricter`, so the most restrictive outcome wins:

```rust
// core/node/via_state_keeper/src/seal_criteria/conditional_sealer.rs (excerpt of should_seal_l1_batch)
        let mut final_seal_resolution = SealResolution::NoSeal;
        for sealer in &self.sealers {
            let seal_resolution = sealer.should_seal(
                &self.config,
                block_open_timestamp_ms,
                tx_count,
                l1_tx_count,
                block_data,
                tx_data,
                protocol_version,
            );
            match &seal_resolution {
                SealResolution::IncludeAndSeal
                | SealResolution::ExcludeAndSeal
                | SealResolution::Unexecutable(_) => {
                    tracing::debug!(
                        "L1 batch #{l1_batch_number} processed by `{name}` with resolution {seal_resolution:?}",
                        name = sealer.prom_criterion_name()
                    );
                    AGGREGATION_METRICS
                        .l1_batch_reason_inc(sealer.prom_criterion_name(), &seal_resolution);
                }
                SealResolution::NoSeal => { /* Don't do anything */ }
            }

            final_seal_resolution = final_seal_resolution.stricter(seal_resolution);
        }
        final_seal_resolution
```

  Note there is no `GasCriterion` or `TimeoutCriterion` in this set: gas is bounded by `GasForBatchTipCriterion` (batch-tip gas), and time-based sealing is the IO-level `TimeoutSealer`, not a conditional criterion.

## Transaction Lifecycle

### 1. Initialization and Batch Setup

```
State Keeper → IO Layer → Batch Environment → VM Initialization
```

1. **Initialization**: [`run_inner()`](core/node/via_state_keeper/src/keeper.rs:107) calls [`io.initialize()`](core/node/via_state_keeper/src/io/mod.rs:121) to get cursor and pending batch data
2. **Batch Environment**: [`wait_for_new_batch_env()`](core/node/via_state_keeper/src/keeper.rs:349) loads [`L1BatchParams`](core/node/via_state_keeper/src/io/mod.rs:61) and system contracts
3. **VM Setup**: [`batch_executor.init_batch()`](core/node/via_state_keeper/src/keeper.rs:258) initializes the VM with batch and system environments
4. **State Recovery**: If pending L2 blocks exist, [`restore_state()`](core/node/via_state_keeper/src/keeper.rs:461) replays them

### 2. Transaction Processing Loop

```
Mempool → State Keeper → Batch Executor → VM → State Updates → Sealing Checks
```

The main processing loop in [`process_l1_batch()`](core/node/via_state_keeper/src/keeper.rs:562):

1. **Check Unconditional Sealing**: If IO indicates batch should be sealed immediately, exit the loop
2. **L2 Block Sealing**: Check if current L2 block should be sealed based on IO criteria
3. **L2 Block Management**: 
   - If sealing needed, call [`seal_l2_block()`](core/node/via_state_keeper/src/keeper.rs:440)
   - Get new L2 block parameters via [`wait_for_new_l2_block_params()`](core/node/via_state_keeper/src/keeper.rs:387)
   - Start next L2 block with [`start_next_l2_block()`](core/node/via_state_keeper/src/keeper.rs:418)
4. **Transaction Execution**: Execute transactions via batch executor handle
5. **State Updates**: [`UpdatesManager`](core/node/via_state_keeper/src/updates/mod.rs:30) accumulates all changes

### 3. L2 Block Sealing (Miniblocks)

L2 blocks provide immediate finality and are created frequently (every ~2 seconds per [`docs/specs/blocks_batches.md`](docs/specs/blocks_batches.md:27)):

- **Purpose**: Fast user confirmations, wallet updates, API responses
- **Frequency**: Time-based (via IO seal criteria in [`MempoolIO`](core/node/via_state_keeper/src/io/mempool.rs:66))
- **Persistence**: [`seal_l2_block()`](core/node/via_state_keeper/src/keeper.rs:440) delegates to output handler

### 4. L1 Batch Sealing and Rollover

L1 batches aggregate multiple L2 blocks for efficient DA/settlement submission:

- **Sealing Criteria**: Multiple criteria evaluated by [`SequencerSealer`](core/node/via_state_keeper/src/seal_criteria/conditional_sealer.rs:45)
- **Batch Finalization**: [`batch_executor.finish_batch()`](core/node/via_state_keeper/src/keeper.rs:204) completes the batch
- **Persistence**: [`output_handler.handle_l1_batch()`](core/node/via_state_keeper/src/keeper.rs:209) persists the sealed batch
- **Rollover**: Initialize new batch environment and continue processing

## Data Availability and Settlement Integration

### Celestia Data Availability

**Via L2 uses Celestia as its primary Data Availability layer**:

1. **DA Dispatcher**: [`ViaDataAvailabilityDispatcher`](core/node/via_da_dispatcher/src/da_dispatcher.rs:21) orchestrates DA operations
2. **Celestia Client**: [`CelestiaClient`](core/lib/via_da_clients/src/celestia/client.rs:21) implements blob submission and retrieval
3. **Blob Operations**:
   - [`dispatch_blob()`](core/lib/via_da_clients/src/celestia/client.rs:55): Submit batch data/proofs to Celestia
   - Returns blob_id for inclusion tracking
4. **Integration**: Wired via [`DataAvailabilityDispatcherLayer`](core/bin/via_server/src/node_builder.rs:450) in via_server

**Configuration**: [`ViaCelestiaConfig`](core/lib/config/src/configs/via_celestia.rs:10) with API node URL, auth token, and blob size limits.

### Bitcoin Settlement Anchoring

**Bitcoin inscriptions anchor batch commitments and reference Celestia blobs**:

1. **BTC Aggregator**: [`ViaAggregator`](core/node/via_btc_sender/src/aggregator.rs) determines when to create inscriptions
2. **Aggregated Operations** (`core/node/via_btc_sender/src/aggregated_operations.rs`):
   ```rust
   pub enum ViaAggregatedOperation {
       CommitL1BatchOnchain(Vec<ViaBtcL1BlockDetails>),
       CommitProofOnchain(Vec<ViaBtcL1BlockDetails>),
   }
   ```
   with matching request types `ViaBtcInscriptionRequestType::{CommitL1BatchOnchain, CommitProofOnchain}`
3. **Inscription Message Structure**: `construct_inscription_message()` (`aggregator.rs`) embeds:
   - DA identifier (defaults to "celestia" per [`ViaBtcSenderConfig`](core/lib/config/src/configs/via_btc_sender.rs:45))
   - Celestia blob_id references
4. **Inscription Management**: [`ViaBtcInscriptionManager`](core/node/via_btc_sender/src/btc_inscription_manager.rs:19) handles Bitcoin transaction lifecycle

**Taproot Protocol**: Inscriptions use Tapscript witness data per [`core/lib/via_btc_client/DEV.md`](core/lib/via_btc_client/DEV.md:73) with structured message encoding.

## L2 Blocks vs L1 Batches

### L2 Blocks (Miniblocks)
From [`docs/specs/blocks_batches.md`](docs/specs/blocks_batches.md:20):
- **Purpose**: User experience, fast confirmations, API compatibility
- **Frequency**: Every 2 seconds (configurable via `miniblock_commit_deadline_ms`)
- **Size**: Small (typically few to hundreds of transactions)
- **Finality**: Soft finality until L1 batch is anchored

### L1 Batches
From [`docs/specs/blocks_batches.md`](docs/specs/blocks_batches.md:31):
- **Purpose**: Proof generation unit, DA submission, settlement anchoring
- **Composition**: Multiple L2 blocks aggregated together
- **Sealing Criteria**: Resource limits (gas, pubdata, circuits, slots, time)
- **Finality**: Hard finality once DA is confirmed and Bitcoin is anchored

## Sealing Criteria Deep Dive

The sealing system uses multiple criteria from [`docs/specs/blocks_batches.md`](docs/specs/blocks_batches.md:43):

### L2 Block Sealing (IO-based)
- **Time Deadline**: Via [`IoSealCriteria`](core/node/via_state_keeper/src/io/mempool.rs:66) in MempoolIO
- **Payload Size**: Based on l2_block_max_payload_size configuration

### L1 Batch Sealing (Conditional)
- **Transaction Slots**: `SlotsCriterion` - capped by `transaction_slots` (250 in `etc/env/base/chain.toml` and `StateKeeperConfig::for_tests()`)
- **Pubdata Size**: `PubDataBytesCriterion` - bounded by the batch pubdata limit
- **Circuit Constraints**: `CircuitsCriterion` - zkEVM geometry limits
- **Encoding Size**: `TxEncodingSizeCriterion` - bootloader encoding-space limits
- **Batch Tip Gas**: `GasForBatchTipCriterion` - reserves gas for closing the batch
- **L1-L2 Transactions**: `L1L2TxsCriterion` - caps priority (deposit) transactions per batch
- **L2-L1 Logs**: `L2L1LogsCriterion` - caps L2→L1 logs per batch

### Unconditional (IO-level)
- **Batch Timeout**: `TimeoutSealer` - `block_commit_deadline_ms` (2500 ms in `StateKeeperConfig::for_tests()`, 5000 ms in `etc/env/base/chain.toml`); never seals an empty batch

## Configuration Parameters

### State Keeper Configuration
Key parameters from [`StateKeeperConfig`](core/lib/config/src/configs/chain.rs:51) (the struct is defined in `core/lib/config`; `core/lib/env_config/src/chain.rs` only implements loading it from environment variables). The values below are the ones used in the localhost environment (`etc/env/base/chain.toml`):

```yaml
# L2 Block timing (config field: l2_block_commit_deadline_ms;
# "miniblock_commit_deadline_ms" is accepted as a serde alias)
l2_block_commit_deadline_ms: 1000

# L1 Batch timing (unconditional TimeoutSealer)
block_commit_deadline_ms: 5000           # etc/env/base/chain.toml; StateKeeperConfig::for_tests() uses 2500

# L1 Batch constraints
transaction_slots: 250                   # max transactions per batch
max_pubdata_per_batch: 100000            # Max pubdata bytes
validation_computational_gas_limit: 300000

# VM and execution
save_call_traces: true                   # For debugging
```

The corresponding fields in the Rust struct (excerpt; full struct in [`core/lib/config/src/configs/chain.rs`](core/lib/config/src/configs/chain.rs:51)):

```rust
// core/lib/config/src/configs/chain.rs (excerpt of StateKeeperConfig)
#[derive(Debug, Deserialize, Clone, PartialEq, Default)]
pub struct StateKeeperConfig {
    /// The max number of slots for txs in a block before it should be sealed by the slots sealer.
    pub transaction_slots: usize,

    /// Number of ms after which an L1 batch is going to be unconditionally sealed.
    pub block_commit_deadline_ms: u64,
    /// Number of ms after which an L2 block should be sealed by the timeout sealer.
    #[serde(alias = "miniblock_commit_deadline_ms")]
    // legacy naming; since we don't serialize this struct, we use "alias" rather than "rename"
    pub l2_block_commit_deadline_ms: u64,
    /// Capacity of the queue for asynchronous L2 block sealing. Once this many L2 blocks are queued,
    /// sealing will block until some of the L2 blocks from the queue are processed.
    /// 0 means that sealing is synchronous; this is mostly useful for performance comparison, testing etc.
    #[serde(alias = "miniblock_seal_queue_capacity")]
    pub l2_block_seal_queue_capacity: usize,
    /// The max payload size threshold (in bytes) that triggers sealing of an L2 block.
    #[serde(alias = "miniblock_max_payload_size")]
    pub l2_block_max_payload_size: usize,

    // ...

    /// The maximum amount of pubdata that can be used by the batch.
    /// This variable should not exceed:
    /// - 128kb for calldata-based rollups
    /// - 120kb * n, where `n` is a number of blobs for blob-based rollups
    /// - the DA layer's blob size limit for the DA layer-based validiums
    /// - 100 MB for the object store-based or no-da validiums
    pub max_pubdata_per_batch: u64,

    /// The version of the fee model to use.
    pub fee_model_version: FeeModelVersion,

    /// Max number of computational gas that validation step is allowed to take.
    pub validation_computational_gas_limit: u32,
    pub save_call_traces: bool,

    // ...
}
```

### Celestia Configuration
From [`ViaCelestiaConfig`](core/lib/config/src/configs/via_celestia.rs:17):

```yaml
da_backend: Celestia                     # or Http (via-core-ext proxy)
api_node_url: "ws://localhost:26658"     # Celestia node API
blob_size_limit: 1973786                # Maximum blob size
proof_sending_mode: OnlyRealProofs       # or SkipEveryProof
# auth token comes from secrets (ViaDASecrets), not this config
```

```rust
// core/lib/config/src/configs/via_celestia.rs
#[derive(Debug, Deserialize, Clone, PartialEq)]
pub struct ViaCelestiaConfig {
    /// DA backend type.
    pub da_backend: DaBackend,

    /// Celestia url.
    pub api_node_url: String,

    /// Celestia blob limit
    pub blob_size_limit: usize,

    /// The mode in which proofs are sent.
    pub proof_sending_mode: ProofSendingMode,

    /// Optional external node URL for fallback when Celestia data is not available
    #[serde(default)]
    pub fallback_external_node_url: Option<String>,

    /// Whether to verify data consistency between Celestia and fallback
    #[serde(default)]
    pub verify_consistency: bool,
}
```

### Bitcoin Sender Configuration  
From [`ViaBtcSenderConfig`](core/lib/config/src/configs/via_btc_sender.rs:8):

```yaml
da_identifier: "celestia"                # Default DA layer identifier
# Aggregation and inscription parameters configured per deployment
```

```rust
// core/lib/config/src/configs/via_btc_sender.rs
#[derive(Debug, Deserialize, Serialize, Clone, PartialEq)]
pub struct ViaBtcSenderConfig {
    /// Service interval in milliseconds.
    pub poll_interval: u64,

    // Number of blocks to commit at time, should be 'one'.
    pub max_aggregated_blocks_to_commit: i32,

    // Number of proofs to commit at time, should be 'one'.
    pub max_aggregated_proofs_to_commit: i32,

    // The max number of inscription in flight
    pub max_txs_in_flight: i64,

    /// Number of block confirmations required to mark the inscription request as confirmed.
    pub block_confirmations: u32,

    /// The identifier of the DA layer.
    pub da_identifier: Option<String>,

    /// The btc sender wallet address.
    pub wallet_address: String,

    /// The number of blocks to wait before considering an inscription stuck.
    pub stuck_inscription_block_number: Option<u32>,

    /// The required time (seconds) to wait before create a commit inscription.
    pub block_time_to_commit: Option<u32>,

    /// The required time (seconds) to wait before create a proof inscription.
    pub block_time_to_proof: Option<u32>,
}
```

`da_identifier` is optional and falls back to `"celestia"` via `ViaBtcSenderConfig::da_identifier()` (`const DEFAULT_DA_LAYER: &str = "celestia";` in the same file).

## State Recovery and Consistency

### Pending Batch Recovery
The state keeper handles restarts gracefully via [`restore_state()`](core/node/via_state_keeper/src/keeper.rs:461):

1. **Pending Detection**: IO layer identifies partially processed batches
2. **L2 Block Replay**: Re-execute all pending L2 blocks in order
3. **State Consistency**: VM state must exactly match pre-restart state
4. **Continuation**: Resume normal processing from exact restoration point

### Protocol Upgrades
Protocol version changes are handled via [`load_protocol_upgrade_tx()`](core/node/via_state_keeper/src/keeper.rs:264):

- Upgrade transactions are loaded when protocol version changes
- Must be first transaction in the batch for the new version
- Proper sequencing ensures network-wide consistency

## Error Handling and Resilience

### Transaction Execution Failures
- **Validation Failures**: Rejected transactions via [`reject()`](core/node/via_state_keeper/src/io/mempool.rs:365)
- **VM Halts**: Handled gracefully with state rollback
- **Try-and-Rollback**: Per [`docs/specs/blocks_batches.md`](docs/specs/blocks_batches.md:62), attempt transaction inclusion with rollback capability

The try-and-rollback behavior is the `match` on the seal resolution inside `process_l1_batch()`: on `ExcludeAndSeal` the last transaction is rolled back in both the VM and the IO (returned to the mempool), on `Unexecutable` it is rolled back in the VM and rejected:

```rust
// core/node/via_state_keeper/src/keeper.rs (excerpt of process_l1_batch)
            match &seal_resolution {
                SealResolution::NoSeal | SealResolution::IncludeAndSeal => {
                    let TxExecutionResult::Success {
                        tx_result,
                        tx_metrics: tx_execution_metrics,
                        call_tracer_result,
                        compressed_bytecodes,
                        ..
                    } = exec_result
                    else {
                        unreachable!(
                            "Tx inclusion seal resolution must be a result of a successful tx execution",
                        );
                    };
                    updates_manager.extend_from_executed_transaction(
                        tx,
                        *tx_result,
                        compressed_bytecodes,
                        *tx_execution_metrics,
                        call_tracer_result,
                    );
                }
                SealResolution::ExcludeAndSeal => {
                    batch_executor.rollback_last_tx().await.with_context(|| {
                        format!("failed rolling back transaction {tx_hash:?} in batch executor")
                    })?;
                    self.io.rollback(tx).await.with_context(|| {
                        format!("failed rolling back transaction {tx_hash:?} in I/O")
                    })?;
                }
                SealResolution::Unexecutable(reason) => {
                    batch_executor.rollback_last_tx().await.with_context(|| {
                        format!("failed rolling back transaction {tx_hash:?} in batch executor")
                    })?;
                    self.io
                        .reject(&tx, reason.clone())
                        .await
                        .with_context(|| format!("cannot reject transaction {tx_hash:?}"))?;
                }
            };
```

Rejection resets the sender's mempool nonces and marks the transaction as rejected in Postgres; L1 (priority) transactions are never rejected:

```rust
// core/node/via_state_keeper/src/io/mempool.rs
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

### System Recovery
- **Graceful Shutdown**: the `stop_receiver` passed to [`run()`](core/node/via_state_keeper/src/keeper.rs:95) enables clean shutdown; cancellation surfaces as `Error::Canceled` rather than a failure:

```rust
// core/node/via_state_keeper/src/keeper.rs
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

- **Database Consistency**: All state changes are transactional
- **Restart Safety**: State keeper resumes from exact previous state

## Integration with External Systems

### Ethereum Compatibility Mode
When built as zksync_server:
- **Eth Sender**: [`EthTxAggregatorLayer`](core/bin/zksync_server/src/node_builder.rs:32) handles L1 contract interaction
- **L1 Operations**: [`CommitBlocks`](core/node/eth_sender/src/aggregated_operations.rs:3), [`ProveBatches`](core/node/eth_sender/src/aggregated_operations.rs:3), [`ExecuteBatches`](core/node/eth_sender/src/aggregated_operations.rs:3) via [`IExecutor`](core/node/eth_sender/src/aggregated_operations.rs:3)

### Data Availability Clients
The system supports pluggable DA via [`DataAvailabilityClient`](core/lib/via_da_clients/src/celestia/client.rs:55):
- **Celestia**: Production DA for Via L2
- **Object Store**: [`ObjectStoreClient`](core/lib/default_da_clients/README.md:11) for alternative storage
- **No DA**: [`NoDAClient`](core/lib/default_da_clients/README.md:10) for testing

## Performance and Monitoring

### Key Metrics
- **KEEPER_METRICS**: State keeper performance and health ([`core/node/via_state_keeper/src/metrics.rs`](core/node/via_state_keeper/src/metrics.rs:140))
- **L1_BATCH_METRICS**: Batch creation and sealing statistics ([`core/node/via_state_keeper/src/metrics.rs`](core/node/via_state_keeper/src/metrics.rs:315)); `seal_delta` is observed in the batch rollover loop ([`L1_BATCH_METRICS.seal_delta`](core/node/via_state_keeper/src/keeper.rs:214))
- **METRICS** (`ViaBtcSenderMetrics`): Bitcoin inscription tracking and confirmation metrics ([`core/node/via_btc_sender/src/metrics.rs`](core/node/via_btc_sender/src/metrics.rs:140)); note there is no `BTC_METRICS` static in via-core, the BTC sender's global is simply named `METRICS`

`KEEPER_METRICS` is a `vise::Global<StateKeeperMetrics>` exported with the `via_server_state_keeper` prefix:

```rust
// core/node/via_state_keeper/src/metrics.rs
/// General-purpose state keeper metrics.
#[derive(Debug, Metrics)]
#[metrics(prefix = "via_server_state_keeper")]
pub struct StateKeeperMetrics {
    /// Latency to synchronize the mempool with Postgres.
    #[metrics(buckets = Buckets::LATENCIES)]
    pub mempool_sync: Histogram<Duration>,
    /// Latency of the state keeper waiting for a transaction.
    #[metrics(buckets = Buckets::LATENCIES)]
    pub waiting_for_tx: Histogram<Duration>,
    /// Latency of the state keeper getting a transaction from the mempool.
    #[metrics(buckets = Buckets::LATENCIES)]
    pub get_tx_from_mempool: Histogram<Duration>,
    /// Number of transactions completed with a specific result.
    pub tx_execution_result: Family<TxExecutionResult, Counter>,
    /// Time spent waiting for the hash of a previous L1 batch.
    #[metrics(buckets = Buckets::LATENCIES)]
    pub wait_for_prev_hash_time: Histogram<Duration>,
    /// Time spent waiting for the header of a previous L2 block.
    #[metrics(buckets = Buckets::LATENCIES)]
    pub load_previous_miniblock_header: Histogram<Duration>,
    /// The time it takes for transactions to be included in a block. Representative of the time user must wait before their transaction is confirmed.
    #[metrics(buckets = INCLUSION_DELAY_BUCKETS)]
    pub transaction_inclusion_delay: Family<TxExecutionType, Histogram<Duration>>,
    /// The time it takes to match seal resolution for each tx.
    #[metrics(buckets = Buckets::LATENCIES)]
    pub match_seal_resolution: Histogram<Duration>,
    /// The time it takes to determine seal resolution for each tx.
    #[metrics(buckets = Buckets::LATENCIES)]
    pub determine_seal_resolution: Histogram<Duration>,
    /// The time it takes for state keeper to wait for tx execution result from batch executor.
    #[metrics(buckets = Buckets::LATENCIES)]
    pub execute_tx_outer_time: Histogram<Duration>,
    /// The time it takes for one iteration of the main loop in `process_l1_batch`.
    #[metrics(buckets = Buckets::LATENCIES)]
    pub process_l1_batch_loop_iteration: Histogram<Duration>,
    /// The time it takes to wait for new L2 block parameters
    #[metrics(buckets = Buckets::LATENCIES)]
    pub wait_for_l2_block_params: Histogram<Duration>,
}

#[vise::register]
pub static KEEPER_METRICS: vise::Global<StateKeeperMetrics> = vise::Global::new();
```

L1 batch sealing statistics live in `L1BatchMetrics` (prefix `via_server_state_keeper_l1_batch`):

```rust
// core/node/via_state_keeper/src/metrics.rs
/// Metrics related to L1 batch sealing.
#[derive(Debug, Metrics)]
#[metrics(prefix = "via_server_state_keeper_l1_batch")]
pub(crate) struct L1BatchMetrics {
    /// Delta between sealing consecutive L1 batches.
    #[metrics(buckets = L1_BATCH_SEAL_DELTA_BUCKETS)]
    pub seal_delta: Histogram<Duration>,
    /// Number of initial writes in a single L1 batch.
    #[metrics(buckets = COUNT_BUCKETS)]
    pub initial_writes: Histogram<usize>,
    /// Number of repeated writes in a single L1 batch.
    #[metrics(buckets = COUNT_BUCKETS)]
    pub repeated_writes: Histogram<usize>,
    /// Number of transactions in a single L1 batch.
    #[metrics(buckets = COUNT_BUCKETS)]
    pub transactions_in_l1_batch: Histogram<usize>,
    /// Total latency of sealing an L1 batch.
    #[metrics(buckets = Buckets::LATENCIES)]
    pub sealed_time: Histogram<Duration>,
    /// Latency of sealing an L1 batch split by the stage.
    #[metrics(buckets = Buckets::LATENCIES)]
    sealed_time_stage: Family<L1BatchSealStage, Histogram<Duration>>,
    /// Number of entities stored in Postgres during a specific stage of sealing an L1 batch.
    #[metrics(buckets = COUNT_BUCKETS)]
    sealed_entity_count: Family<L1BatchSealStage, Histogram<usize>>,
    /// Latency of sealing an L1 batch split by the stage and divided by the number of entries
    /// stored in the stage.
    #[metrics(buckets = Buckets::LATENCIES)]
    sealed_entity_per_unit: Family<L1BatchSealStage, Histogram<Duration>>,
}

#[vise::register]
pub(crate) static L1_BATCH_METRICS: vise::Global<L1BatchMetrics> = vise::Global::new();
```

The BTC sender exposes inscription metrics with the `via_btc_sender` prefix:

```rust
// core/node/via_btc_sender/src/metrics.rs
#[derive(Debug, Metrics)]
#[metrics(prefix = "via_btc_sender")]
pub struct ViaBtcSenderMetrics {
    /// Latency of collecting Ethereum sender metrics.
    #[metrics(buckets = Buckets::LATENCIES, unit = Unit::Seconds)]
    metrics_latency: Histogram<Duration>,

    /// Time taken to broadcast a transaction
    #[metrics(buckets = Buckets::LATENCIES, unit = Unit::Seconds)]
    pub broadcast_time: Histogram<Duration>,

    /// Time taken to prepare inscription data
    #[metrics(buckets = Buckets::LATENCIES, unit = Unit::Seconds)]
    pub inscription_preparation_time: Histogram<Duration>,

    /// Time between inscription submission and confirmation
    #[metrics(buckets = Buckets::exponential(60.0..=86400.0, 5.0), unit = vise::Unit::Seconds)]
    pub inscription_confirmation_time: Histogram<Duration>,

    /// Number of pending inscription requests (not yet submitted)
    pub pending_inscription_requests: Gauge<usize>,

    /// Number of inflight inscriptions (submitted but not confirmed)
    pub inflight_inscriptions: Gauge<usize>,

    /// The first l1_batch number blocked inscription.
    pub report_blocked_l1_batch_inscription: Gauge<usize>,

    /// Error when broadcast a transaction.
    pub l1_transient_errors: Counter,

    /// Aggregator errors.
    pub aggregator_errors: Counter,

    /// Manager errors.
    pub manager_errors: Counter,

    /// Last L1 block observed by the Ethereum sender.
    pub last_known_l1_block: Family<BlockNumberVariant, Gauge<usize>>,

    /// The BTC balance of the account used to created inscriptions.
    #[metrics(labels = ["address"])]
    pub btc_sender_account_balance: LabeledFamily<String, Gauge<usize>>,
}

#[vise::register]
pub static METRICS: vise::Global<ViaBtcSenderMetrics> = vise::Global::new();
```

### Critical Monitoring Points
- Transaction processing rate and latency
- L2 block sealing frequency and size
- L1 batch sealing triggers and resource utilization  
- Celestia blob dispatch and inclusion confirmation
- Bitcoin inscription success rate and confirmation time
- Mempool size and pending transaction age

## Conclusion

The Via L2 sequencer implements a sophisticated dual-layer architecture that provides immediate user finality through L2 blocks while ensuring ultimate security through L1 batch aggregation. The separation of data availability (Celestia) and settlement anchoring (Bitcoin inscriptions) enables high throughput with strong security guarantees.

**Key Architectural Benefits**:
- **Immediate Finality**: L2 blocks provide fast user confirmations
- **Cost Efficiency**: L1 batches aggregate many transactions for DA/settlement
- **Security**: Bitcoin anchoring with DA references ensures data availability
- **Modularity**: Pluggable DA and settlement layers for flexibility
- **Consistency**: Robust state recovery and upgrade handling

This design enables Via L2 to deliver Ethereum-like user experience while leveraging Bitcoin's security model and Celestia's data availability capabilities.

## References

**Core Implementation**:
- State Keeper: [`core/node/via_state_keeper/src/keeper.rs`](core/node/via_state_keeper/src/keeper.rs:68)
- IO Layer: [`core/node/via_state_keeper/src/io/mod.rs`](core/node/via_state_keeper/src/io/mod.rs:115), [`core/node/via_state_keeper/src/io/mempool.rs`](core/node/via_state_keeper/src/io/mempool.rs:48)
- Batch Executor: [`core/lib/vm_interface/src/executor.rs`](core/lib/vm_interface/src/executor.rs:16), [`core/lib/vm_executor/src/batch/factory.rs`](core/lib/vm_executor/src/batch/factory.rs:65) (re-exported in [`core/node/via_state_keeper/src/executor/mod.rs`](core/node/via_state_keeper/src/executor/mod.rs:6))
- Sealing: [`core/node/via_state_keeper/src/seal_criteria/conditional_sealer.rs`](core/node/via_state_keeper/src/seal_criteria/conditional_sealer.rs:14)

**Data Availability & Settlement**:
- Celestia: [`core/lib/via_da_clients/src/celestia/client.rs`](core/lib/via_da_clients/src/celestia/client.rs:21), [`core/node/via_da_dispatcher/src/da_dispatcher.rs`](core/node/via_da_dispatcher/src/da_dispatcher.rs:21)
- Bitcoin: [`core/node/via_btc_sender/src/aggregator.rs`](core/node/via_btc_sender/src/aggregator.rs:19), [`core/node/via_btc_sender/src/btc_inscription_manager.rs`](core/node/via_btc_sender/src/btc_inscription_manager.rs:19)
- Protocol: [`core/lib/via_btc_client/DEV.md`](core/lib/via_btc_client/DEV.md:73)

**System Integration**:
- Via Server: [`core/bin/via_server/src/node_builder.rs`](core/bin/via_server/src/node_builder.rs:32)
- Configuration: [`core/lib/config/src/configs/via_celestia.rs`](core/lib/config/src/configs/via_celestia.rs:10), [`core/lib/config/src/configs/via_btc_sender.rs`](core/lib/config/src/configs/via_btc_sender.rs:8)

**Documentation**:
- Blocks & Batches: [`docs/specs/blocks_batches.md`](docs/specs/blocks_batches.md:20)
- Data Flow: [`docs/via_guides/data-flow.md`](docs/via_guides/data-flow.md:58)
- Components: [`docs/guides/external-node/06_components.md`](docs/guides/external-node/06_components.md:31)