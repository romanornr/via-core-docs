# Proposer Role in Via L2 Bitcoin ZK-Rollup

## Overview

The Proposer in Via L2 is responsible for preparing L1 batches, obtaining their corresponding ZK proofs, and submitting both the batch data to the Data Availability (DA) layer (Celestia) and the commitments to the L1 layer (Bitcoin) via inscriptions. This document details how the Proposer functionality is implemented in the Via L2 system.

## Implementation Architecture

The Proposer role in Via is not implemented as a single, distinct process but rather as a collection of coordinated components that work together to fulfill the proposing functions. These components include:

1. **Data Availability Dispatcher** - Sends batch data to the DA layer (Celestia)
2. **BTC Inscription Aggregator** - Aggregates operations for Bitcoin inscriptions
3. **BTC Inscription Manager** - Manages the creation and submission of inscriptions to Bitcoin
4. **Commitment Generator** - Generates commitments for L1 batches
5. **Proof Data Handler** - Handles proof data from the Prover Gateway

These components are integrated within the Via node and work together to implement the Proposer functionality.

## L1 Batch Construction and Proposal Flow

### 1. L1 Batch Construction

L1 batches are initially constructed by the State Keeper component (as documented in `state_management.md`). The State Keeper is responsible for:
- Executing transactions
- Creating L1 batches
- Generating state diffs
- Storing batch data in the database

### 2. Commitment Generation

Once an L1 batch is created, the `CommitmentGenerator` component generates the necessary commitments:

```rust
// From core/node/commitment_generator/src/lib.rs
pub struct CommitmentGenerator {
    computer: Arc<dyn CommitmentComputer>,
    connection_pool: ConnectionPool<Core>,
    health_updater: HealthUpdater,
    commitment_mode: L1BatchCommitmentMode,
    parallelism: NonZeroU32,
}
```

The commitment generator:
- Calculates auxiliary commitments (events queue, bootloader initial content)
- Prepares commitment input data
- Generates L1 batch commitments
- Stores commitment artifacts in the database

### 3. Proof Acquisition

The `ProofDataHandler` component interfaces with the Prover Gateway to obtain ZK proofs for L1 batches:

```rust
// From core/node/proof_data_handler/src/lib.rs
fn create_proof_processing_router(
    blob_store: Arc<dyn ObjectStore>,
    connection_pool: ConnectionPool<Core>,
    config: ProofDataHandlerConfig,
    commitment_mode: L1BatchCommitmentMode,
) -> Router {
    // ...
}
```

The proof data handler:
- Provides proof generation data to the Prover Gateway
- Receives and validates proofs from the Prover Gateway
- Stores proof artifacts in the object store
- Updates the database with proof metadata

### 4. Data Availability Submission

The `ViaDataAvailabilityDispatcher` component is responsible for submitting batch data and proofs to the DA layer (Celestia):

```rust
// From core/node/via_da_dispatcher/src/da_dispatcher.rs
pub struct ViaDataAvailabilityDispatcher {
    client: Box<dyn DataAvailabilityClient>,
    pool: ConnectionPool<Core>,
    config: DADispatcherConfig,
    blob_store: Arc<dyn ObjectStore>,
    dispatch_real_proof: bool,
}
```

The DA dispatcher:
- Dispatches L1 batch data (pubdata) to Celestia
- Dispatches proof data to Celestia
- Tracks blob IDs and inclusion status
- Polls for confirmation of data inclusion in Celestia

### 5. Bitcoin Inscription Submission

The final step involves submitting commitments to Bitcoin via inscriptions, which is handled by two components:

#### 5.1 BTC Inscription Aggregator

```rust
// From core/node/via_btc_sender/src/btc_inscription_aggregator.rs
pub struct ViaBtcInscriptionAggregator {
    inscriber: Inscriber,
    aggregator: ViaAggregator,
    pool: ConnectionPool<Core>,
    config: ViaBtcSenderConfig,
}
```

The BTC inscription aggregator:
- Determines which L1 batches are ready for commitment to Bitcoin
- Aggregates operations for efficiency
- Constructs inscription messages
- Creates inscription requests

#### 5.2 BTC Inscription Manager

```rust
// From core/node/via_btc_sender/src/btc_inscription_manager.rs
pub struct ViaBtcInscriptionManager {
    inscriber: Inscriber,
    config: ViaBtcSenderConfig,
    pool: ConnectionPool<Core>,
}
```

The BTC inscription manager:
- Sends inscription transactions to Bitcoin
- Monitors transaction confirmation status
- Handles transaction retries if needed
- Updates the database with inscription status

## Inscription Types and Data Structure

The Via system uses two main types of inscriptions:

```rust
// From core/node/via_btc_sender/src/aggregated_operations.rs
pub enum ViaAggregatedOperation {
    CommitL1BatchOnchain(Vec<ViaBtcL1BlockDetails>),
    CommitProofOnchain(Vec<ViaBtcL1BlockDetails>),
}
```

1. **CommitL1BatchOnchain** - Contains L1 batch data references:
   ```rust
   // From core/node/via_btc_sender/src/aggregator.rs
   let input = L1BatchDAReferenceInput {
       l1_batch_hash: H256::from_slice(batch.hash.as_ref()?),
       l1_batch_index: batch.number,
       da_identifier: self.config.da_identifier().to_string(),
       blob_id: batch.blob_id.clone(),
       prev_l1_batch_hash: H256::from_slice(batch.prev_l1_batch_hash.as_ref()?),
   };
   ```

2. **CommitProofOnchain** - Contains proof data references:
   ```rust
   // From core/node/via_btc_sender/src/aggregator.rs
   let input = ProofDAReferenceInput {
       l1_batch_reveal_txid: batch.reveal_tx_id,
       da_identifier: self.config.da_identifier().to_string(),
       blob_id: batch.blob_id.clone(),
   };
   ```

## Component Interactions

The Proposer functionality involves interactions between multiple components:

1. **State Keeper → Commitment Generator**:
   - State Keeper creates L1 batches
   - Commitment Generator processes these batches to generate commitments

2. **Commitment Generator → Proof Data Handler**:
   - Commitment Generator creates commitment artifacts
   - Proof Data Handler uses these artifacts to prepare proof generation data

3. **Proof Data Handler ↔ Prover Gateway**:
   - Proof Data Handler provides proof generation data to Prover Gateway
   - Prover Gateway returns proofs to Proof Data Handler

4. **Proof Data Handler → DA Dispatcher**:
   - Proof Data Handler stores proof artifacts
   - DA Dispatcher retrieves and dispatches these proofs to Celestia

5. **State Keeper → DA Dispatcher**:
   - State Keeper creates L1 batch data
   - DA Dispatcher dispatches this data to Celestia

6. **DA Dispatcher → BTC Inscription Aggregator**:
   - DA Dispatcher records blob IDs from Celestia
   - BTC Inscription Aggregator uses these blob IDs in inscription messages

7. **BTC Inscription Aggregator → BTC Inscription Manager**:
   - BTC Inscription Aggregator creates inscription requests
   - BTC Inscription Manager sends these requests to Bitcoin

## Sequence Diagram

```
┌─────────────┐      ┌────────────────┐      ┌─────────────────┐      ┌───────────────┐      ┌───────────┐
│ State Keeper │      │ Commitment Gen │      │ Proof Data Hdlr │      │ DA Dispatcher │      │ BTC Sender│
└──────┬──────┘      └───────┬────────┘      └────────┬────────┘      └───────┬───────┘      └─────┬─────┘
       │                     │                         │                       │                     │
       │ Create L1 Batch     │                         │                       │                     │
       │─────────────────────>                         │                       │                     │
       │                     │                         │                       │                     │
       │                     │ Generate Commitments    │                       │                     │
       │                     │────────────────────────>│                       │                     │
       │                     │                         │                       │                     │
       │                     │                         │ Request Proof         │                     │
       │                     │                         │───────────────────────>                     │
       │                     │                         │                       │                     │
       │                     │                         │<───────────────────────                     │
       │                     │                         │ Receive Proof         │                     │
       │                     │                         │                       │                     │
       │                     │                         │                       │ Dispatch Batch Data │
       │                     │                         │                       │─────────────────────>
       │                     │                         │                       │                     │
       │                     │                         │                       │ Dispatch Proof Data │
       │                     │                         │                       │─────────────────────>
       │                     │                         │                       │                     │
       │                     │                         │                       │                     │
       │                     │                         │                       │ Create Inscription  │
       │                     │                         │                       │ Request             │
       │                     │                         │                       │─────────────────────>
       │                     │                         │                       │                     │
       │                     │                         │                       │                     │ Send
       │                     │                         │                       │                     │ Inscription
       │                     │                         │                       │                     │ to Bitcoin
       │                     │                         │                       │                     │
```

## Conclusion

The Proposer role in Via L2 is implemented as a collection of coordinated components that work together to:
1. Prepare L1 batches (via the State Keeper)
2. Generate commitments for these batches
3. Obtain ZK proofs from the Prover Gateway
4. Submit batch data and proofs to the DA layer (Celestia)
5. Submit commitments to the L1 layer (Bitcoin) via inscriptions

This distributed approach allows for a modular and scalable system where each component handles a specific part of the proposing process.