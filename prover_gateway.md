# Via L2 Bitcoin ZK-Rollup: Prover Gateway Documentation

## 1. Introduction

The Prover Gateway is the interface between the core system and the prover subsystem in the Via L2 Bitcoin ZK-Rollup. It manages the flow of data between these components, ensuring that batches are fetched for proving and that finished proofs are submitted back to the core system.

The gateway binary lives at `prover/crates/bin/prover_fri_gateway` and is inherited zksync-era machinery: Via reuses it essentially unchanged. What Via changes is what happens to a proof after submission (see Section 5.3): instead of being sent to an Ethereum L1 contract, the proof is dispatched to the data availability layer (Celestia), referenced in a Bitcoin inscription, and verified off-chain by the Via Verifier Network.

## 2. Role and Responsibilities

The Prover Gateway has two primary responsibilities:

1. **Proof Generation Data Fetching**: Polling the core system's `proof_data_handler` API to retrieve data for new batches that need to be proven
2. **Proof Submission**: Collecting completed (compressed) proofs from the prover database/object store and submitting them back to the core system

## 3. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                       Prover Gateway                            │
│                                                                 │
│  ┌─────────────────────┐           ┌─────────────────────────┐  │
│  │ ProofGenDataFetcher │           │    ProofSubmitter       │  │
│  │  (ProverApiClient)  │           │   (ProverApiClient)     │  │
│  └─────────┬───────────┘           └───────────┬─────────────┘  │
│            │ POST /proof_generation_data       │ POST /submit_proof/:n
└────────────┼───────────────────────────────────┼────────────────┘
             ▼                                   ▼
┌─────────────────────────────────────────────────────────────────┐
│           Core node proof_data_handler HTTP server              │
│      (core/node/proof_data_handler, RequestProcessor)           │
└─────────────────────────────────────────────────────────────────┘
```

Both pollers also talk to the prover PostgreSQL database (`ConnectionPool<Prover>`) and the prover object store (file-backed or GCS, depending on configuration).

### 3.1 Key Components

All in `prover/crates/bin/prover_fri_gateway/src/`:

1. **ProofGenDataFetcher** (`proof_gen_data_fetcher.rs`): Fetches proof generation data from the core system's API and stores it in the object store and prover database for further processing by the prover subsystem.

2. **ProofSubmitter** (`proof_submitter.rs`): Retrieves completed proofs from the database and object store and submits them back to the core system's API.

3. **ProverApiClient** (`client.rs`): A thin wrapper around a `reqwest` HTTP client that also holds the blob store, connection pool, and API URL.

4. **PeriodicApi** (`traits.rs`): A trait that defines the periodic polling loop used by both the ProofGenDataFetcher and ProofSubmitter.

## 4. Implementation Details

### 4.1 Main Entry Point

```rust
// prover/crates/bin/prover_fri_gateway/src/main.rs
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let opt = Cli::parse();

    let general_config = load_general_config(opt.config_path).context("general config")?;
    let database_secrets = load_database_secrets(opt.secrets_path).context("database secrets")?;

    let observability_config = general_config
        .observability
        .context("observability config")?;
    let _observability_guard = observability_config.install()?;

    let config = general_config
        .prover_gateway
        .context("prover gateway config")?;

    let postgres_config = general_config.postgres_config.context("postgres config")?;
    let pool = ConnectionPool::<Prover>::builder(
        database_secrets.prover_url()?,
        postgres_config.max_connections()?,
    )
    .build()
    .await
    .context("failed to build a connection pool")?;
    let object_store_config = ProverObjectStoreConfig(
        general_config
            .prover_config
            .context("prover config")?
            .prover_object_store
            .context("object store")?,
    );
    let store_factory = ObjectStoreFactory::new(object_store_config.0);

    let proof_submitter = ProofSubmitter::new(
        store_factory.create_store().await?,
        config.api_url.clone(),
        pool.clone(),
    );
    let proof_gen_data_fetcher = ProofGenDataFetcher::new(
        store_factory.create_store().await?,
        config.api_url.clone(),
        pool,
    );

    let (stop_sender, stop_receiver) = watch::channel(false);

    let (stop_signal_sender, stop_signal_receiver) = oneshot::channel();
    let mut stop_signal_sender = Some(stop_signal_sender);
    ctrlc::set_handler(move || {
        if let Some(stop_signal_sender) = stop_signal_sender.take() {
            stop_signal_sender.send(()).ok();
        }
    })
    .context("Error setting Ctrl+C handler")?;

    tracing::info!("Starting Fri Prover Gateway");

    let tasks = vec![
        tokio::spawn(
            PrometheusExporterConfig::pull(config.prometheus_listener_port)
                .run(stop_receiver.clone()),
        ),
        tokio::spawn(proof_gen_data_fetcher.run(config.api_poll_duration(), stop_receiver.clone())),
        tokio::spawn(proof_submitter.run(config.api_poll_duration(), stop_receiver)),
    ];

    let mut tasks = ManagedTasks::new(tasks);
    tokio::select! {
        _ = tasks.wait_single() => {},
        _ = stop_signal_receiver => {
            tracing::info!("Stop signal received, shutting down");
        }
    }
    stop_sender.send(true).ok();
    tasks.complete(Duration::from_secs(5)).await;
    Ok(())
}
```

### 4.2 The Polling Loop: PeriodicApi

Both pollers implement the `PeriodicApi` trait, whose default `run` method drives the `get_next_request` -> `send_request` -> `handle_response` cycle:

```rust
// prover/crates/bin/prover_fri_gateway/src/traits.rs
/// Trait for fetching data from an API periodically.
#[async_trait::async_trait]
pub(crate) trait PeriodicApi: Sync + Send + 'static + Sized {
    type JobId: Send + Copy;
    type Request: Send;
    type Response: Send;

    const SERVICE_NAME: &'static str;

    /// Returns the next request to be sent to the API and the endpoint to send it to.
    async fn get_next_request(&self) -> Option<(Self::JobId, Self::Request)>;

    /// Submits a request to the API.
    async fn send_request(
        &self,
        job_id: Self::JobId,
        request: Self::Request,
    ) -> reqwest::Result<Self::Response>;

    /// Handles the response from the API.
    async fn handle_response(&self, job_id: Self::JobId, response: Self::Response);

    /// Runs `get_next_request` -> `send_request` -> `handle_response` in a loop.
    async fn run(
        self,
        poll_duration: Duration,
        mut stop_receiver: watch::Receiver<bool>,
    ) -> anyhow::Result<()> {
        tracing::info!(
            "Starting periodic job: {} with frequency: {:?}",
            Self::SERVICE_NAME,
            poll_duration
        );

        loop {
            if *stop_receiver.borrow() {
                tracing::warn!("Stop signal received, shutting down {}", Self::SERVICE_NAME);
                return Ok(());
            }

            if let Some((job_id, request)) = self.get_next_request().await {
                match self.send_request(job_id, request).await {
                    Ok(response) => {
                        self.handle_response(job_id, response).await;
                    }
                    Err(err) => {
                        METRICS.http_error[&Self::SERVICE_NAME].inc();
                        tracing::error!("HTTP request failed due to error: {}", err);
                    }
                }
            }
            // Exit condition will be checked on the next iteration.
            tokio::time::timeout(poll_duration, stop_receiver.changed())
                .await
                .ok();
        }
    }
}
```

The shared HTTP client:

```rust
// prover/crates/bin/prover_fri_gateway/src/client.rs
/// A tiny wrapper over the reqwest client that also stores
/// the objects commonly needed when interacting with prover API.
#[derive(Debug)]
pub(crate) struct ProverApiClient {
    pub(crate) blob_store: Arc<dyn ObjectStore>,
    pub(crate) pool: ConnectionPool<Prover>,
    pub(crate) api_url: String,
    pub(crate) client: reqwest::Client,
}

impl ProverApiClient {
    pub(crate) fn new(
        blob_store: Arc<dyn ObjectStore>,
        pool: ConnectionPool<Prover>,
        api_url: String,
    ) -> Self {
        Self {
            blob_store,
            pool,
            api_url,
            client: reqwest::Client::new(),
        }
    }

    pub(crate) async fn send_http_request<Req, Resp>(
        &self,
        request: Req,
        endpoint: &str,
    ) -> Result<Resp, reqwest::Error>
    where
        Req: Serialize,
        Resp: DeserializeOwned,
    {
        tracing::info!("Sending request to {}", endpoint);

        self.client
            .post(endpoint)
            .json(&request)
            .send()
            .await?
            .error_for_status()?
            .json::<Resp>()
            .await
    }
}
```

### 4.3 Proof Generation Data Fetching

`ProofGenDataFetcher` polls `{api_url}/proof_generation_data` with an empty request; there is always a "next request", so it polls unconditionally:

```rust
// prover/crates/bin/prover_fri_gateway/src/proof_gen_data_fetcher.rs
/// Poller structure that will periodically check the prover API for new proof generation data.
/// Fetched data is stored to the database/object store for further processing.
#[derive(Debug)]
pub struct ProofGenDataFetcher(ProverApiClient);

/// The path to the API endpoint that returns the next proof generation data.
const PROOF_GENERATION_DATA_PATH: &str = "/proof_generation_data";

impl ProofGenDataFetcher {
    pub(crate) fn new(
        blob_store: Arc<dyn ObjectStore>,
        base_url: String,
        pool: ConnectionPool<Prover>,
    ) -> Self {
        let api_url = format!("{base_url}{PROOF_GENERATION_DATA_PATH}");
        let inner = ProverApiClient::new(blob_store, pool, api_url);
        Self(inner)
    }
}
```

```rust
// prover/crates/bin/prover_fri_gateway/src/proof_gen_data_fetcher.rs
#[async_trait]
impl PeriodicApi for ProofGenDataFetcher {
    type JobId = ();
    type Request = ProofGenerationDataRequest;
    type Response = ProofGenerationDataResponse;

    const SERVICE_NAME: &'static str = "ProofGenDataFetcher";

    async fn get_next_request(&self) -> Option<(Self::JobId, ProofGenerationDataRequest)> {
        Some(((), ProofGenerationDataRequest {}))
    }

    async fn send_request(
        &self,
        _: (),
        request: ProofGenerationDataRequest,
    ) -> reqwest::Result<Self::Response> {
        self.0.send_http_request(request, &self.0.api_url).await
    }

    async fn handle_response(&self, _: (), response: Self::Response) {
        match response {
            ProofGenerationDataResponse::Success(Some(data)) => {
                tracing::info!("Received proof gen data for: {:?}", data.l1_batch_number);
                self.save_proof_gen_data(*data).await;
            }
            ProofGenerationDataResponse::Success(None) => {
                tracing::info!("There are currently no pending batches to be proven");
            }
            ProofGenerationDataResponse::Error(err) => {
                tracing::error!("Failed to get proof gen data: {:?}", err);
            }
        }
    }
}
```

When proof generation data is received, it is saved to the object store and the prover database:

```rust
// prover/crates/bin/prover_fri_gateway/src/proof_gen_data_fetcher.rs
    #[tracing::instrument(
        name = "ProofGenDataFetcher::save_proof_gen_data",
        skip_all,
        fields(l1_batch = %data.l1_batch_number)
    )]
    async fn save_proof_gen_data(&self, data: ProofGenerationData) {
        let store = &*self.0.blob_store;
        let witness_inputs = store
            .put(data.l1_batch_number, &data.witness_input_data)
            .await
            .expect("Failed to save proof generation data to GCS");
        let mut connection = self.0.pool.connection().await.unwrap();

        connection
            .fri_protocol_versions_dal()
            .save_prover_protocol_version(data.protocol_version, data.l1_verifier_config)
            .await;

        connection
            .fri_witness_generator_dal()
            .save_witness_inputs(data.l1_batch_number, &witness_inputs, data.protocol_version)
            .await;
    }
```

### 4.4 Proof Submission

`ProofSubmitter` polls the prover database for the oldest compressed proof not yet sent, and posts it to `{api_url}/submit_proof/{l1_batch_number}`:

```rust
// prover/crates/bin/prover_fri_gateway/src/proof_submitter.rs
/// The path to the API endpoint that submits the proof.
const SUBMIT_PROOF_PATH: &str = "/submit_proof";

/// Poller structure that will periodically check the database for new proofs to submit.
/// Once a new proof is detected, it will be sent to the prover API.
#[derive(Debug)]
pub struct ProofSubmitter(ProverApiClient);
```

```rust
// prover/crates/bin/prover_fri_gateway/src/proof_submitter.rs
impl ProofSubmitter {
    async fn next_submit_proof_request(&self) -> Option<(L1BatchNumber, SubmitProofRequest)> {
        let (l1_batch_number, protocol_version, status) = self
            .0
            .pool
            .connection()
            .await
            .unwrap()
            .fri_proof_compressor_dal()
            .get_least_proven_block_not_sent_to_server()
            .await?;

        let request = match status {
            ProofCompressionJobStatus::Successful => {
                let proof = self
                    .0
                    .blob_store
                    .get((l1_batch_number, protocol_version))
                    .await
                    .expect("Failed to get compressed snark proof from blob store");
                SubmitProofRequest::Proof(Box::new(proof))
            }
            ProofCompressionJobStatus::Skipped => SubmitProofRequest::SkippedProofGeneration,
            _ => panic!(
                "Trying to send proof that are not successful status: {:?}",
                status
            ),
        };

        Some((l1_batch_number, request))
    }

    async fn save_successful_sent_proof(&self, l1_batch_number: L1BatchNumber) {
        self.0
            .pool
            .connection()
            .await
            .unwrap()
            .fri_proof_compressor_dal()
            .mark_proof_sent_to_server(l1_batch_number)
            .await;
    }
}
```

```rust
// prover/crates/bin/prover_fri_gateway/src/proof_submitter.rs
#[async_trait]
impl PeriodicApi for ProofSubmitter {
    type JobId = L1BatchNumber;
    type Request = SubmitProofRequest;
    type Response = SubmitProofResponse;
    const SERVICE_NAME: &'static str = "ProofSubmitter";

    async fn get_next_request(&self) -> Option<(Self::JobId, SubmitProofRequest)> {
        let (l1_batch_number, request) = self.next_submit_proof_request().await?;
        Some((l1_batch_number, request))
    }

    async fn send_request(
        &self,
        job_id: Self::JobId,
        request: SubmitProofRequest,
    ) -> reqwest::Result<Self::Response> {
        let endpoint = format!("{}/{job_id}", self.0.api_url);
        self.0.send_http_request(request, &endpoint).await
    }

    async fn handle_response(&self, job_id: L1BatchNumber, response: Self::Response) {
        tracing::info!("Received response: {:?}", response);
        self.save_successful_sent_proof(job_id).await;
    }
}
```

After a successful submission, `mark_proof_sent_to_server` sets the compression job status to `sent_to_server` (`ProofCompressionJobStatus::SentToServer`, `core/lib/basic_types/src/prover_dal.rs`).

### 4.5 Core-Side API Endpoints (proof_data_handler)

The counterpart of the gateway runs inside the core node: `core/node/proof_data_handler`. It is also inherited zksync-era machinery. Its router exposes the two endpoints the gateway calls (plus optional TEE endpoints):

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

So the two endpoints used by the gateway are:

1. **`POST /proof_generation_data`**: returns `ProofGenerationDataResponse` (batch data plus witness inputs) for the next batch to prove, or `Success(None)` when nothing is pending. On the core side, `RequestProcessor::get_proof_generation_data` locks a batch for proving before returning it.
2. **`POST /submit_proof/:l1_batch_number`**: accepts a `SubmitProofRequest` (either `Proof(...)` or `SkippedProofGeneration`) and returns `SubmitProofResponse`.

On proof submission, `RequestProcessor::submit_proof` (`core/node/proof_data_handler/src/request_processor.rs`) does not verify the SNARK itself. It stores the proof blob in the core object store, cross-checks the prover's `aggregation_result_coords` (system logs hash, state diff hash, bootloader heap initial content, events queue state) against server-side batch metadata (panicking on mismatch), and then records the blob URL via `proof_generation_dal().save_proof_artifacts_metadata(l1_batch_number, &blob_url)`. For `SkippedProofGeneration` it calls `proof_generation_dal().mark_proof_generation_job_as_skipped(l1_batch_number)`.

### 4.6 Database Tables

The gateway touches these prover-database tables (via `prover/crates/lib/prover_dal`):

1. **`witness_inputs_fri`**: written by `save_witness_inputs` (`fri_witness_generator_dal.rs`) with `l1_batch_number`, `witness_inputs_blob_url`, `protocol_version`, `status` (inserted as `'queued'`), `created_at`, `updated_at`, `protocol_version_patch`. This is the entry point of a batch into the prover pipeline.

2. **`proof_compression_jobs_fri`**: read by `get_least_proven_block_not_sent_to_server` (selects the minimum `l1_batch_number` with status `successful` or `skipped`) and updated by `mark_proof_sent_to_server` (sets status to `sent_to_server`), both in `fri_proof_compressor_dal.rs`.

## 5. Interactions with Other Components

### 5.1 Interaction with the Core System

1. **Fetching Proof Generation Data**: the gateway periodically polls the core `proof_data_handler`; the core locks the next unproven batch and returns its witness input data.
2. **Submitting Proofs**: the gateway posts the compressed SNARK proof back; the core stores the proof blob, sanity-checks the prover's auxiliary output against its own batch metadata, and marks the proof artifacts as saved. Actual proof verification happens later, off the core node (see 5.3).

### 5.2 Interaction with Prover Components

The gateway interacts with the rest of the prover subsystem only indirectly, through the prover database and object store: it enqueues `witness_inputs_fri` rows that the witness generator picks up, and it consumes the output of the proof compressor (`proof_compression_jobs_fri` plus the compressed proof blob).

### 5.3 Via-Specific: What Happens to a Submitted Proof

This is where Via diverges from zksync-era. After `submit_proof` stores the proof on the core side:

1. **DA dispatch**: the Via DA dispatcher (`core/node/via_da_dispatcher`) picks up proofs that are ready and publishes them to the DA layer in chunks:

```rust
// core/node/via_da_dispatcher/src/da_dispatcher.rs
    async fn dispatch_real_proofs(&self) -> anyhow::Result<()> {
        let mut conn = self.pool.connection_tagged("da_dispatcher").await?;

        if self.is_rollback_required(&mut conn).await? {
            return Ok(());
        }

        let proofs = conn
            .via_data_availability_dal()
            .get_ready_for_da_dispatch_proofs(self.config.max_rows_to_dispatch() as usize)
            .await?;

        drop(conn);

        for proof in proofs {
            // fetch the proof from object store
            let final_proof = match self.load_real_proof_operation(proof.l1_batch_number).await {
                Some(proof) => proof,
                None => {
                    tracing::error!("Failed to load proof for batch {}", proof.l1_batch_number.0);
                    continue;
                }
            };

            let chunks: Vec<Vec<u8>> = final_proof
                .chunks(BLOB_CHUNK_SIZE)
                .map(|chunk| chunk.to_vec())
                .collect();

            self._dispatch_chunks(proof.l1_batch_number, chunks, true, "real_proof_dispatcher")
                .await?;

            METRICS
                .last_dispatched_proof_batch
                .set(proof.l1_batch_number.0 as usize);
            METRICS.blob_size.observe(final_proof.len());
        }
        Ok(())
    }
```

   The dispatcher has a `dispatch_real_proof: bool` flag; when it is false, `dispatch_dummy_proofs` is used instead (same file, `dispatch_proofs` selects between them).

2. **Bitcoin inscription**: the BTC sender inscribes a `ProofDAReference` on Bitcoin pointing at the DA blob (see `via_btc_client` message types `ProofDAReference` / `L1BatchDAReference`).

3. **Verifier network verification**: each verifier runs `ViaVerifier` (`via_verifier/node/via_zk_verifier/src/lib.rs`), which follows the inscription chain, fetches the proof and batch pubdata blobs from DA via `get_inclusion_data(&...input.blob_id)`, and verifies the proof off-chain (`via_verification::verify_proof`), then votes on the batch:

```rust
// via_verifier/node/via_zk_verifier/src/lib.rs
    pub async fn run(mut self, mut stop_receiver: watch::Receiver<bool>) -> anyhow::Result<()> {
        let mut timer = tokio::time::interval(self.config.polling_interval());
        let pool = self.pool.clone();

        while !*stop_receiver.borrow_and_update() {
            tokio::select! {
                _ = timer.tick() => { /* continue iterations */ }
                _ = stop_receiver.changed() => break,
            }

            let mut storage = pool.connection_tagged("via_zk_verifier").await?;
            match self.loop_iteration(&mut storage).await {
                Ok(()) => {}
                Err(err) => {
                    METRICS.errors.inc();
                    tracing::error!("Failed to process via_zk_verifier: {err}")
                }
            }
        }

        tracing::info!("Stop signal received, via_zk_verifier is shutting down");
        Ok(())
    }
```

   Inside `loop_iteration`, the verifier takes the first not-yet-verified L1 batch in the canonical inscription chain (`via_votes_dal().get_first_not_verified_l1_batch_in_canonical_inscription_chain()`), parses the `ProofDAReference` inscription, fetches the proof blob and the referenced `L1BatchDAReference` pubdata blob from the DA client, and proceeds to verification and voting.

## 6. Configuration Parameters

The gateway is configured by `FriProverGatewayConfig` (loaded from the general config as `prover_gateway`):

```rust
// core/lib/config/src/configs/fri_prover_gateway.rs
#[derive(Debug, Deserialize, Clone, PartialEq)]
pub struct FriProverGatewayConfig {
    pub api_url: String,
    pub api_poll_duration_secs: u16,

    /// Configurations for prometheus
    pub prometheus_listener_port: u16,
    pub prometheus_pushgateway_url: String,
    pub prometheus_push_interval_ms: Option<u64>,
}

impl FriProverGatewayConfig {
    pub fn api_poll_duration(&self) -> Duration {
        Duration::from_secs(self.api_poll_duration_secs as u64)
    }
}
```

In addition, `main.rs` pulls the prover object store config (`general_config.prover_config.prover_object_store`), the postgres config (`general_config.postgres_config`), the prover database URL from the secrets file (`database_secrets.prover_url()`), and the observability config.

## 7. Error Handling and Monitoring

HTTP failures in the polling loop are logged and counted; database and object-store failures in the gateway's own code paths mostly `unwrap`/`expect` (crashing the task rather than retrying in place).

The gateway exposes exactly one Prometheus metric, an HTTP error counter labeled by service name:

```rust
// prover/crates/bin/prover_fri_gateway/src/metrics.rs
use vise::{Counter, LabeledFamily, Metrics};

#[derive(Debug, Metrics)]
#[metrics(prefix = "prover_fri_prover_fri_gateway")]
pub(crate) struct ProverFriGatewayMetrics {
    #[metrics(labels = ["service_name"])]
    pub http_error: LabeledFamily<&'static str, Counter>,
}

#[vise::register]
pub(crate) static METRICS: vise::Global<ProverFriGatewayMetrics> = vise::Global::new();
```

Job statistics and processing-time metrics are reported by other prover components (e.g. the witness generators and the prover job monitor), not by the gateway.

## 8. Conclusion

The Prover Gateway is the bridge between the Via core node and the prover subsystem: it pulls witness input data over HTTP from the core's `proof_data_handler` and pushes compressed proofs back. The gateway itself is inherited zksync-era machinery; Via's differences begin after submission, when the proof is dispatched to the DA layer, referenced on Bitcoin, and verified off-chain by the Verifier Network.
