# Proposer Role in Via L2 Bitcoin ZK-Rollup

## Overview

The Proposer in Via L2 is responsible for preparing L1 batches, obtaining their corresponding ZK proofs, and submitting both the batch data to the Data Availability (DA) layer (Celestia) and the commitments to the L1 layer (Bitcoin) via inscriptions. This document details how the Proposer functionality is implemented in the Via L2 system.

Note on terminology: "Proposer" is a role named in the via-core `README.md` component list ("Sequencer, Proposer, Prover, Prover Gateway, Verifier Network Node"), which also states that `/core/bin` contains "`via_server` for Sequencer and Proposer software". There is no separate proposer binary, crate, or config section in the codebase; the via-core guide `docs/via_guides/data-flow.md` consistently refers to this side of the system as the "sequencer/proposer". In code, the role is realized by the components documented below, all running inside `via_server`.

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
// core/node/commitment_generator/src/lib.rs
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
// core/node/proof_data_handler/src/lib.rs
fn create_proof_processing_router(
    blob_store: Arc<dyn ObjectStore>,
    connection_pool: ConnectionPool<Core>,
    config: ProofDataHandlerConfig,
    commitment_mode: L1BatchCommitmentMode,
    l2_chain_id: L2ChainId,
) -> Router {
    let get_proof_gen_processor = RequestProcessor::new(
        blob_store.clone(),
        connection_pool.clone(),
        config.clone(),
        commitment_mode,
    );
    let submit_proof_processor = get_proof_gen_processor.clone();
    let mut router = Router::new()
        .route(
            "/proof_generation_data",
            post(
                // we use post method because the returned data is not idempotent,
                // i.e we return different result on each call.
                move |payload: Json<ProofGenerationDataRequest>| async move {
                    get_proof_gen_processor
                        .get_proof_generation_data(payload)
                        .await
                },
            ),
        )
        .route(
            "/submit_proof/:l1_batch_number",
            post(
                move |l1_batch_number: Path<u32>, payload: Json<SubmitProofRequest>| async move {
                    submit_proof_processor
                        .submit_proof(l1_batch_number, payload)
                        .await
                },
            ),
        );

    if config.tee_config.tee_support {
        let get_tee_proof_gen_processor =
            TeeRequestProcessor::new(blob_store, connection_pool, config.clone(), l2_chain_id);
        let submit_tee_proof_processor = get_tee_proof_gen_processor.clone();
        let register_tee_attestation_processor = get_tee_proof_gen_processor.clone();

        router = router.route(
            "/tee/proof_inputs",
            post(
                move |payload: Json<TeeProofGenerationDataRequest>| async move {
                    let result = get_tee_proof_gen_processor
                        .get_proof_generation_data(payload)
                        .await;

                    match result {
                        Ok(Some(data)) => (StatusCode::OK, data).into_response(),
                        Ok(None) => { StatusCode::NO_CONTENT.into_response()},
                        Err(e) => e.into_response(),
                    }
                },
            ),
        )
        .route(
            "/tee/submit_proofs/:l1_batch_number",
            post(
                move |l1_batch_number: Path<u32>, payload: Json<SubmitTeeProofRequest>| async move {
                    submit_tee_proof_processor
                        .submit_proof(l1_batch_number, payload)
                        .await
                },
            ),
        )
        .route(
            "/tee/register_attestation",
            post(
                move |payload: Json<RegisterTeeAttestationRequest>| async move {
                    register_tee_attestation_processor
                        .register_tee_attestation(payload)
                        .await
                },
            ),
        );
    }

    router
        .layer(tower_http::compression::CompressionLayer::new())
        .layer(tower_http::decompression::RequestDecompressionLayer::new().zstd(true))
}
```

The Prover Gateway pulls work from `/proof_generation_data` (handled by `RequestProcessor::get_proof_generation_data` in `core/node/proof_data_handler/src/request_processor.rs`) and submits finished proofs to `/submit_proof/:l1_batch_number` (handled by `RequestProcessor::submit_proof`, which stores the proof in the blob store, cross-checks the prover's `aggregation_result_coords` against the batch metadata's `events_queue_commitment` and `bootloader_initial_content_commitment`, and saves the proof artifact metadata in the database).

The proof data handler:
- Provides proof generation data to the Prover Gateway
- Receives and validates proofs from the Prover Gateway
- Stores proof artifacts in the object store
- Updates the database with proof metadata

### 4. Data Availability Submission

The `ViaDataAvailabilityDispatcher` component is responsible for submitting batch data and proofs to the DA layer (Celestia):

```rust
// core/node/via_da_dispatcher/src/da_dispatcher.rs
pub struct ViaDataAvailabilityDispatcher {
    client: Box<dyn DataAvailabilityClient>,
    pool: ConnectionPool<Core>,
    config: DADispatcherConfig,
    blob_store: Arc<dyn ObjectStore>,
    dispatch_real_proof: bool,
}
```

Its main loop runs three subtasks concurrently on each polling interval:

```rust
// core/node/via_da_dispatcher/src/da_dispatcher.rs
    pub async fn run(self, mut stop_receiver: Receiver<bool>) -> anyhow::Result<()> {
        loop {
            if *stop_receiver.borrow() {
                break;
            }

            // Run dispatch, poll_for_inclusion, and dispatch_proofs concurrently
            let subtasks = futures::future::join3(
                async {
                    if let Err(err) = self.dispatch().await {
                        METRICS.errors.inc();
                        tracing::error!("dispatch error {err:?}");
                    }
                },
                async {
                    if let Err(err) = self.poll_for_inclusion().await {
                        METRICS.errors.inc();
                        tracing::error!("poll_for_inclusion error {err:?}");
                    }
                },
                async {
                    if let Err(err) = self.dispatch_proofs().await {
                        METRICS.errors.inc();
                        tracing::error!("dispatch_proofs error {err:?}");
                    }
                },
            );

            tokio::select! {
                _ = subtasks => {},
                _ = stop_receiver.changed() => {
                    break;
                }
            }

            if tokio::time::timeout(self.config.polling_interval(), stop_receiver.changed())
                .await
                .is_ok()
            {
                break;
            }
        }

        tracing::info!("Stop signal received, da_dispatcher is shutting down");
        Ok(())
    }
```

The DA dispatcher:
- Dispatches L1 batch data (pubdata) to Celestia (`dispatch`)
- Dispatches proof data to Celestia (`dispatch_proofs`, which calls `dispatch_real_proofs` or `dispatch_dummy_proofs` depending on the `dispatch_real_proof` flag)
- Tracks blob IDs and inclusion status
- Polls for confirmation of data inclusion in Celestia (`poll_for_inclusion`, which delegates to `poll_for_inclusion_l1_batch` and `poll_for_inclusion_proof`)

### 5. Bitcoin Inscription Submission

The final step involves submitting commitments to Bitcoin via inscriptions, which is handled by two components:

#### 5.1 BTC Inscription Aggregator

```rust
// core/node/via_btc_sender/src/btc_inscription_aggregator.rs
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
// core/node/via_btc_sender/src/btc_inscription_manager.rs
pub struct ViaBtcInscriptionManager {
    inscriber: Inscriber,
    config: ViaBtcSenderConfig,
    pool: ConnectionPool<Core>,
}
```

The BTC inscription manager:
- Sends inscription transactions to Bitcoin (`send_new_inscription_txs` / `send_inscription_tx`)
- Monitors transaction confirmation status (`update_inscription_status`)
- Detects inscriptions stuck for more than `stuck_inscription_block_number` blocks (via `get_first_stuck_l1_batch_number_inscription_request`)
- Updates the database with inscription status

## Inscription Types and Data Structure

The BTC sender aggregates batches into two operation types:

```rust
// core/node/via_btc_sender/src/aggregated_operations.rs
#[allow(clippy::large_enum_variant)]
#[derive(Debug, Clone)]
pub enum ViaAggregatedOperation {
    CommitL1BatchOnchain(Vec<ViaBtcL1BlockDetails>),
    CommitProofOnchain(Vec<ViaBtcL1BlockDetails>),
}

impl ViaAggregatedOperation {
    pub fn get_l1_batches_detail(&self) -> &Vec<ViaBtcL1BlockDetails> {
        match self {
            Self::CommitL1BatchOnchain(l1_batch) => l1_batch,
            Self::CommitProofOnchain(l1_batch) => l1_batch,
        }
    }

    pub fn get_inscription_request_type(&self) -> ViaBtcInscriptionRequestType {
        match self {
            Self::CommitL1BatchOnchain(..) => ViaBtcInscriptionRequestType::CommitL1BatchOnchain,
            Self::CommitProofOnchain(..) => ViaBtcInscriptionRequestType::CommitProofOnchain,
        }
    }
}
```

`ViaAggregator::construct_inscription_message` maps each operation type to one of the `InscriptionMessage` variants defined in `via_btc_client` (`L1BatchDAReference` for batch commitments, `ProofDAReference` for proof commitments; these are two of the seven inscribable message variants in the protocol):

```rust
// core/node/via_btc_sender/src/aggregator.rs
    pub fn construct_inscription_message(
        &self,
        inscription_request_type: &ViaBtcInscriptionRequestType,
        batch: &ViaBtcL1BlockDetails,
    ) -> anyhow::Result<InscriptionMessage> {
        match inscription_request_type {
            ViaBtcInscriptionRequestType::CommitL1BatchOnchain => {
                let input = L1BatchDAReferenceInput {
                    l1_batch_hash: H256::from_slice(
                        batch
                            .hash
                            .as_ref()
                            .ok_or_else(|| anyhow!("Via l1 batch hash is None"))?,
                    ),
                    l1_batch_index: batch.number,
                    da_identifier: self.config.da_identifier().to_string(),
                    blob_id: batch.blob_id.clone(),
                    prev_l1_batch_hash: H256::from_slice(
                        batch
                            .prev_l1_batch_hash
                            .as_ref()
                            .ok_or_else(|| anyhow!("Via previous l1 batch hash is None"))?,
                    ),
                };
                Ok(InscriptionMessage::L1BatchDAReference(input))
            }
            ViaBtcInscriptionRequestType::CommitProofOnchain => {
                let input = ProofDAReferenceInput {
                    l1_batch_reveal_txid: batch.reveal_tx_id,
                    da_identifier: self.config.da_identifier().to_string(),
                    blob_id: batch.blob_id.clone(),
                };
                Ok(InscriptionMessage::ProofDAReference(input))
            }
        }
    }
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