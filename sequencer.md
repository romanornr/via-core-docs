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

Full control flow is implemented in the state keeper; see [core/node/state_keeper/src/keeper.rs](https://github.com/vianetwork/via-core/blob/main/core/node/state_keeper/src/keeper.rs)


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
      - If, after applying the just-executed (last) transaction, the cumulative batch gas crosses the configured “close” threshold (still below the hard per-batch limit), include that last executed transaction and seal the batch (“include-and-seal”).
      - Otherwise, continue ("no-seal").

- Pubdata size thresholds
   - Inputs: the cumulative batch pubdata size before applying the just-executed transaction (the "last transaction"), the pubdata attributable to that last transaction, the required bootloader/batch-tip overhead, and the configured pubdata bounds (per-transaction reject bound, batch “close” threshold, and batch hard limit).
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

   * Maximum L2 block payload-enconding size: 
      * Inputs: the cumulative encoded payload size of all transactions currently included in the L2 block.
      * Condition: the L2 block's cumulative payload-encoding sizer meets or exceeds the configured limit. 
      

[ more todo ]

The sequencer dispatches the batch's pubdata and proofs to Celestia for data availability via [CelestiaClient::new()](https://github.com/vianetwork/via-core/blob/main/core/lib/via_da_clients/src/celestia/client.rs#L29) as wired in the [via_server node builder](https://github.com/vianetwork/via-core/blob/main/core/bin/via_server/src/node_builder.rs)
2. A Bitcoin inscription then anchors the batch/proof commitment and references the DA blob through the [inscription manager](https://github.com/vianetwork/via-core/blob/main/core/node/via_btc_sender/src/btc_inscription_manager.rs) following the protocol outlined in the [inscription DEV guide](https://github.com/vianetwork/via-core/blob/main/core/lib/via_btc_client/DEV.md#L73), establishing the default split of DA on Celestia and settlement anchoring on Bitcoin.


To be more specific, the Sequencer is the core transaction‑processing component of Via L2, implemented by [ZkSyncStateKeeper](https://github.com/vianetwork/via-core/blob/main/core/node/state_keeper/src/keeper.rs#L104), which pulls transactions from the [mempool I/O](https://github.com/vianetwork/via-core/blob/main/core/node/state_keeper/src/io/mempool.rs), executes them in the zkEVM, and assembles the results into L2 blocks and L1 batches according to [sealing criteria](https://github.com/vianetwork/via-core/blob/main/core/node/state_keeper/src/seal_criteria/conditional_sealer.rs) described in the [Blocks & Batches spec](https://github.com/vianetwork/via-core/blob/main/docs/specs/blocks_batches.md#L20) and its [sealing limits](https://github.com/vianetwork/via-core/blob/main/docs/specs/blocks_batches.md#L38); once a batch is sealed, its pubdata and proofs are dispatched to Celestia for data availability via [CelestiaClient::new()](https://github.com/vianetwork/via-core/blob/main/core/lib/via_da_clients/src/celestia/client.rs#L29) as wired in the [via_server node builder](https://github.com/vianetwork/via-core/blob/main/core/bin/via_server/src/node_builder.rs), and a Bitcoin inscription then anchors the batch/proof commitment and references the DA blob through the [inscription manager](https://github.com/vianetwork/via-core/blob/main/core/node/via_btc_sender/src/btc_inscription_manager.rs) following the protocol outlined in the [inscription DEV guide](https://github.com/vianetwork/via-core/blob/main/core/lib/via_btc_client/DEV.md#L73), establishing the default split of DA on Celestia and settlement anchoring on Bitcoin.


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

The [`ZkSyncStateKeeper`](core/node/state_keeper/src/keeper.rs:104) is the main sequencer component that coordinates all transaction processing:

- **Initialization**: [`run_inner()`](core/node/state_keeper/src/keeper.rs:145) loads pending batches or starts new ones
- **Transaction Processing**: [`process_l1_batch()`](core/node/state_keeper/src/keeper.rs:565) executes transactions in a loop
- **State Recovery**: [`restore_state()`](core/node/state_keeper/src/keeper.rs:461) replays pending transactions after restart
- **L2 Block Management**: [`seal_l2_block()`](core/node/state_keeper/src/keeper.rs:440) and [`start_next_l2_block()`](core/node/state_keeper/src/keeper.rs:418)

### IO Layer and Mempool Integration

- **State Keeper IO**: [`StateKeeperIO`](core/node/state_keeper/src/io/mod.rs:108) abstraction for transaction sourcing and batch parameters
- **Mempool IO**: [`MempoolIO`](core/node/state_keeper/src/io/mempool.rs:43) implementation that feeds transactions from mempool with sealing criteria
- **Persistence**: [`StateKeeperPersistence`](core/node/state_keeper/src/io/persistence.rs:30) handles L2 block sealing commands and database persistence

### Batch Execution and VM Integration

- **Batch Executor**: [`BatchExecutor`](core/node/state_keeper/src/batch_executor.rs:59) abstraction for VM interaction
- **Main Batch Executor**: [`MainBatchExecutor`](core/node/state_keeper/src/batch_executor/main_executor.rs:27) coordinates VM execution in a separate thread
- **Updates Manager**: [`UpdatesManager`](core/node/state_keeper/src/updates.rs:31) accumulates state changes during batch execution

### Sealing Criteria System

- **Conditional Sealer**: [`ConditionalSealer`](core/node/state_keeper/src/seal_criteria/conditional_sealer.rs:14) interface for sealing decisions
- **Sequencer Sealer**: [`SequencerSealer`](core/node/state_keeper/src/seal_criteria/conditional_sealer.rs:43) aggregates multiple sealing criteria
- **Criteria Examples**:
  - [`SlotsCriterion`](core/node/state_keeper/src/seal_criteria/criteria/slots.rs): Transaction count limits
  - [`GasCriterion`](core/node/state_keeper/src/seal_criteria/criteria/gas.rs): Gas usage limits
  - [`PubdataBytesCriterion`](core/node/state_keeper/src/seal_criteria/criteria/pubdata_bytes.rs): Public data size limits
  - [`TimeoutCriterion`](core/node/state_keeper/src/seal_criteria/criteria/timeout.rs): Time-based sealing

## Transaction Lifecycle

### 1. Initialization and Batch Setup

```
State Keeper → IO Layer → Batch Environment → VM Initialization
```

1. **Initialization**: [`run_inner()`](core/node/state_keeper/src/keeper.rs:146) calls [`io.initialize()`](core/node/state_keeper/src/io/mod.rs:108) to get cursor and pending batch data
2. **Batch Environment**: [`wait_for_new_batch_env()`](core/node/state_keeper/src/keeper.rs:356) loads [`L1BatchParams`](core/node/state_keeper/src/io/mod.rs:61) and system contracts
3. **VM Setup**: [`batch_executor.init_batch()`](core/node/state_keeper/src/keeper.rs:192) initializes the VM with batch and system environments
4. **State Recovery**: If pending L2 blocks exist, [`restore_state()`](core/node/state_keeper/src/keeper.rs:461) replays them

### 2. Transaction Processing Loop

```
Mempool → State Keeper → Batch Executor → VM → State Updates → Sealing Checks
```

The main processing loop in [`process_l1_batch()`](core/node/state_keeper/src/keeper.rs:565):

1. **Check Unconditional Sealing**: If IO indicates batch should be sealed immediately, exit the loop
2. **L2 Block Sealing**: Check if current L2 block should be sealed based on IO criteria
3. **L2 Block Management**: 
   - If sealing needed, call [`seal_l2_block()`](core/node/state_keeper/src/keeper.rs:440)
   - Get new L2 block parameters via [`wait_for_new_l2_block_params()`](core/node/state_keeper/src/keeper.rs:391)
   - Start next L2 block with [`start_next_l2_block()`](core/node/state_keeper/src/keeper.rs:418)
4. **Transaction Execution**: Execute transactions via batch executor handle
5. **State Updates**: [`UpdatesManager`](core/node/state_keeper/src/updates.rs:31) accumulates all changes

### 3. L2 Block Sealing (Miniblocks)

L2 blocks provide immediate finality and are created frequently (every ~2 seconds per [`docs/specs/blocks_batches.md`](docs/specs/blocks_batches.md:27)):

- **Purpose**: Fast user confirmations, wallet updates, API responses
- **Frequency**: Time-based (via IO seal criteria in [`MempoolIO`](core/node/state_keeper/src/io/mempool.rs:66))
- **Persistence**: [`seal_l2_block()`](core/node/state_keeper/src/keeper.rs:441) delegates to output handler

### 4. L1 Batch Sealing and Rollover

L1 batches aggregate multiple L2 blocks for efficient DA/settlement submission:

- **Sealing Criteria**: Multiple criteria evaluated by [`SequencerSealer`](core/node/state_keeper/src/seal_criteria/conditional_sealer.rs:43)
- **Batch Finalization**: [`batch_executor.finish_batch()`](core/node/state_keeper/src/keeper.rs:228) completes the batch
- **Persistence**: [`output_handler.handle_l1_batch()`](core/node/state_keeper/src/keeper.rs:232) persists the sealed batch
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

1. **BTC Aggregator**: [`ViaAggregator`](core/node/via_btc_sender/src/aggregator.rs:19) determines when to create inscriptions
2. **Inscription Types**:
   - [`CommitL1BatchOnchain`](core/node/via_btc_sender/src/aggregator.rs:122): Batch commitment with DA reference
   - [`CommitProofOnchain`](core/node/via_btc_sender/src/aggregator.rs:122): Proof commitment with DA reference
3. **Inscription Message Structure**: [`construct_inscription_message()`](core/node/via_btc_sender/src/aggregator.rs:122) embeds:
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
- **Time Deadline**: Via [`IoSealCriteria`](core/node/state_keeper/src/io/mempool.rs:66) in MempoolIO
- **Payload Size**: Based on l2_block_max_payload_size configuration

### L1 Batch Sealing (Conditional)
- **Transaction Slots**: [`SlotsCriterion`](core/node/state_keeper/src/seal_criteria/criteria/slots.rs) - typically 750 transactions
- **Gas Limit**: [`GasCriterion`](core/node/state_keeper/src/seal_criteria/criteria/gas.rs) - MAX_L2_TX_GAS_LIMIT (80M)
- **Pubdata Size**: [`PubdataBytesCriterion`](core/node/state_keeper/src/seal_criteria/criteria/pubdata_bytes.rs) - MAX_PUBDATA_PER_L1_BATCH (120k)
- **Circuit Constraints**: [`CircuitsCriterion`](core/node/state_keeper/src/seal_criteria/criteria/circuits.rs) - zkEVM geometry limits
- **Encoding Size**: [`TxEncodingSizeCriterion`](core/node/state_keeper/src/seal_criteria/criteria/tx_encoding_size.rs) - encoded transaction limits
- **Timeout**: [`TimeoutCriterion`](core/node/state_keeper/src/seal_criteria/criteria/timeout.rs) - maximum batch duration

## Configuration Parameters

### State Keeper Configuration
Key parameters from [`StateKeeperConfig`](/core/lib/env_config/src/chain.rs):

```yaml
# L2 Block timing
miniblock_commit_deadline_ms: 2000

# L1 Batch constraints
transaction_slots: 750                    # Max transactions per batch
max_pubdata_per_batch: 120000            # Max pubdata bytes
validation_computational_gas_limit: 300000000

# VM and execution
save_call_traces: false                  # For debugging
```

### Celestia Configuration
From [`ViaCelestiaConfig`](/core/lib/config/src/configs/via_celestia.rs):

```yaml
api_node_url: "ws://localhost:26658"     # Celestia node API
auth_token: "..."                        # Authentication token  
blob_size_limit: 1973786                # Maximum blob size
proof_sending_mode: OnlyRealProofs       # or SkipEveryProof
```

### Bitcoin Sender Configuration  
From [`ViaBtcSenderConfig`](core/lib/config/src/configs/via_btc_sender.rs:8):

```yaml
da_identifier: "celestia"                # Default DA layer identifier
# Aggregation and inscription parameters configured per deployment
```

## State Recovery and Consistency

### Pending Batch Recovery
The state keeper handles restarts gracefully via [`restore_state()`](core/node/state_keeper/src/keeper.rs:461):

1. **Pending Detection**: IO layer identifies partially processed batches
2. **L2 Block Replay**: Re-execute all pending L2 blocks in order
3. **State Consistency**: VM state must exactly match pre-restart state
4. **Continuation**: Resume normal processing from exact restoration point

### Protocol Upgrades
Protocol version changes are handled via [`load_protocol_upgrade_tx()`](core/node/state_keeper/src/keeper.rs:268):

- Upgrade transactions are loaded when protocol version changes
- Must be first transaction in the batch for the new version
- Proper sequencing ensures network-wide consistency

## Error Handling and Resilience

### Transaction Execution Failures
- **Validation Failures**: Rejected transactions via [`reject()`](core/node/state_keeper/src/io/mempool.rs:85)
- **VM Halts**: Handled gracefully with state rollback
- **Try-and-Rollback**: Per [`docs/specs/blocks_batches.md`](docs/specs/blocks_batches.md:62), attempt transaction inclusion with rollback capability

### System Recovery
- **Graceful Shutdown**: [`stop_receiver`](core/node/state_keeper/src/keeper.rs:105) enables clean shutdown
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
- **KEEPER_METRICS**: State keeper performance and health
- **L1_BATCH_METRICS**: Batch creation and sealing statistics ([`L1_BATCH_METRICS.seal_delta`](core/node/state_keeper/src/keeper.rs:238))
- **BTC_METRICS**: Bitcoin inscription tracking and confirmation metrics

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
- State Keeper: [`core/node/state_keeper/src/keeper.rs`](core/node/state_keeper/src/keeper.rs:104)
- IO Layer: [`core/node/state_keeper/src/io/mod.rs`](core/node/state_keeper/src/io/mod.rs:108), [`core/node/state_keeper/src/io/mempool.rs`](core/node/state_keeper/src/io/mempool.rs:43)
- Batch Executor: [`core/node/state_keeper/src/batch_executor.rs`](core/node/state_keeper/src/batch_executor.rs:59), [`core/node/state_keeper/src/batch_executor/main_executor.rs`](core/node/state_keeper/src/batch_executor/main_executor.rs:27)
- Sealing: [`core/node/state_keeper/src/seal_criteria/conditional_sealer.rs`](core/node/state_keeper/src/seal_criteria/conditional_sealer.rs:14)

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