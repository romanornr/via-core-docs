# Via L2 Bitcoin ZK-Rollup: Prover Gateway Documentation

## 1. Introduction

The Prover Gateway is a critical component of the Via L2 Bitcoin ZK-Rollup system, serving as the interface between the core system and the prover subsystem. It manages the flow of data between these components, ensuring that batches are proven correctly and efficiently. This document provides a comprehensive overview of the Prover Gateway's architecture, implementation, and role within the overall system.

## 2. Role and Responsibilities

The Prover Gateway has two primary responsibilities:

1. **Proof Generation Data Fetching**: Polling the core system's API to retrieve data for new batches that need to be proven
2. **Proof Submission**: Collecting completed proofs from the prover subsystem and submitting them back to the core system

These responsibilities make the Prover Gateway a critical link in the proving pipeline, ensuring that batches are processed in order and that proofs are properly submitted back to the core system.

## 3. Architecture Overview

The Prover Gateway follows a modular architecture with the following components:

```
┌─────────────────────────────────────────────────────────────────┐
│                       Prover Gateway                            │
│                                                                 │
│  ┌─────────────────────┐           ┌─────────────────────────┐  │
│  │ ProofGenDataFetcher │           │    ProofSubmitter       │  │
│  │                     │           │                         │  │
│  └─────────────┬───────┘           └───────────┬─────────────┘  │
│                │                               │                │
└────────────────┼───────────────────────────────┼────────────────┘
                 │                               │
                 ▼                               ▼
         ┌───────────────┐               ┌───────────────┐
         │  Object Store │               │   Database    │
         │    (GCS)      │               │  (PostgreSQL) │
         └───────────────┘               └───────────────┘
                 │                               │
                 ▼                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                         Core System API                         │
└─────────────────────────────────────────────────────────────────┘
```

### 3.1 Key Components

The Prover Gateway implementation consists of several key components:

1. **ProofGenDataFetcher**: Responsible for fetching proof generation data from the core system's API and storing it in the object store and database for further processing by the prover subsystem.

2. **ProofSubmitter**: Responsible for retrieving completed proofs from the database and object store and submitting them back to the core system's API.

3. **ProverApiClient**: A wrapper around the HTTP client that handles communication with the core system's API.

4. **PeriodicApi**: A trait that defines the periodic polling mechanism used by both the ProofGenDataFetcher and ProofSubmitter.

## 4. Implementation Details

### 4.1 Main Entry Point

The main entry point for the Prover Gateway is defined in `prover/crates/bin/prover_fri_gateway/src/main.rs`:

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // Initialize configuration
    let opt = Cli::parse();
    let general_config = load_general_config(opt.config_path).context("general config")?;
    let database_secrets = load_database_secrets(opt.secrets_path).context("database secrets")?;
    
    // Initialize components
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
    
    // Start the services
    let tasks = vec![
        tokio::spawn(
            PrometheusExporterConfig::pull(config.prometheus_listener_port)
                .run(stop_receiver.clone()),
        ),
        tokio::spawn(proof_gen_data_fetcher.run(config.api_poll_duration(), stop_receiver.clone())),
        tokio::spawn(proof_submitter.run(config.api_poll_duration(), stop_receiver)),
    ];
    
    // Wait for tasks to complete or for stop signal
    // ...
    
    Ok(())
}
```

### 4.2 Proof Generation Data Fetching

The `ProofGenDataFetcher` is responsible for fetching proof generation data from the core system's API:

```rust
impl PeriodicApi for ProofGenDataFetcher {
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

When proof generation data is received, it is saved to the object store and database:

```rust
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

### 4.3 Proof Submission

The `ProofSubmitter` is responsible for submitting proofs back to the core system's API:

```rust
impl PeriodicApi for ProofSubmitter {
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

The `next_submit_proof_request` method retrieves the next proof to be submitted:

```rust
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
```

### 4.4 API Endpoints

The Prover Gateway interacts with the core system through two main API endpoints:

1. **Proof Generation Data Endpoint**: `/proof_generation_data`
   - Used to fetch proof generation data for new batches
   - Returns `ProofGenerationDataResponse` containing batch data and witness inputs

2. **Submit Proof Endpoint**: `/submit_proof/{l1_batch_number}`
   - Used to submit proofs back to the core system
   - Accepts `SubmitProofRequest` containing the proof data
   - Returns `SubmitProofResponse` indicating success or failure

### 4.5 Database Schema

The Prover Gateway interacts with several database tables:

1. **witness_inputs_fri**: Stores witness inputs for basic circuit generation
   - `l1_batch_number`: The batch number
   - `witness_inputs_blob_url`: URL to the witness inputs blob in the object store
   - `protocol_version`: Protocol version for the batch
   - `status`: Current status of the witness job (queued, in_progress, successful, failed)

2. **proof_compression_jobs_fri**: Stores proof compression jobs
   - `l1_batch_number`: The batch number
   - `fri_proof_blob_url`: URL to the FRI proof blob in the object store
   - `l1_proof_blob_url`: URL to the L1 proof blob in the object store
   - `status`: Current status of the compression job (queued, in_progress, successful, failed, sent_to_server)

## 5. Interactions with Other Components

### 5.1 Interaction with Core System

The Prover Gateway interacts with the Core system through HTTP API calls:

1. **Fetching Proof Generation Data**:
   - The Prover Gateway periodically polls the Core API to get new batches to prove
   - The Core system provides the necessary data for proof generation, including witness inputs

2. **Submitting Proofs**:
   - The Prover Gateway submits generated proofs back to the Core system
   - The Core system verifies the proofs and updates the batch status

### 5.2 Interaction with Prover Components

The Prover Gateway indirectly interacts with other prover components through the database and object store:

1. **Witness Generator**:
   - The Prover Gateway stores witness inputs in the database and object store
   - The Witness Generator retrieves these inputs to generate witness data

2. **Circuit Prover**:
   - The Circuit Prover generates proofs based on witness data
   - The proofs are stored in the database and object store

3. **Proof Compressor**:
   - The Proof Compressor compresses proofs for submission to L1
   - The compressed proofs are stored in the database and object store
   - The Prover Gateway retrieves these compressed proofs for submission to the Core system

## 6. Configuration Parameters

The Prover Gateway behavior is controlled by various configuration parameters:

1. **API URL**: The base URL for the Core system's API
2. **API Poll Duration**: The interval at which to poll the API for new jobs
3. **Prometheus Listener Port**: The port on which to expose Prometheus metrics
4. **Object Store Configuration**: Configuration for the object store (GCS)
5. **Database Configuration**: Configuration for the PostgreSQL database

## 7. Error Handling and Monitoring

### 7.1 Error Handling

The Prover Gateway implements robust error handling:

1. **API Errors**: Errors from API calls are logged and tracked in metrics
2. **Database Errors**: Database errors are propagated and handled appropriately
3. **Object Store Errors**: Object store errors are propagated and handled appropriately

### 7.2 Monitoring

The Prover Gateway exposes Prometheus metrics for monitoring:

1. **HTTP Errors**: Count of HTTP errors by service name
2. **Job Statistics**: Count of jobs by status (queued, in_progress, successful, failed)
3. **Processing Time**: Time taken to process jobs

## 8. Conclusion

The Prover Gateway is a critical component of the Via L2 Bitcoin ZK-Rollup system, serving as the interface between the core system and the prover subsystem. It ensures that batches are proven correctly and efficiently, and that proofs are properly submitted back to the core system. Its modular architecture and robust error handling make it a reliable component in the proving pipeline.