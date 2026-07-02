# Celestia Integration in Via L2

This document describes how Via L2 integrates with Celestia as its Data Availability (DA) layer.

## 1. Overview

Via L2 uses Celestia as a Data Availability (DA) layer to ensure that L2 batch data is publicly available. The integration consists of these components:

1. **Celestia Client** (`core/lib/via_da_clients/src/celestia/`): implements the `DataAvailabilityClient` trait against a Celestia light node.
2. **HTTP DA Client** (`core/lib/via_da_clients/src/http/`): an alternative backend that talks to the `via-core-ext` proxy service instead of a light node; selected by `DaBackend::Http` vs `DaBackend::Celestia` in config.
3. **Fallback DA Client** (`core/lib/via_da_clients/src/fallback/`): wraps the primary client with an external-node fallback for data past Celestia's retention window, with optional byte-for-byte consistency checking.
4. **DA Dispatcher** (`core/node/via_da_dispatcher/`): orchestrates submission of batch pubdata and proofs, chunking large payloads, and polls for inclusion.

The integration ensures that both L2 batch data and batch proofs are published to Celestia, making them available for verification by the Verifier Network.

## 2. Role of Celestia in Via L2 Architecture

Celestia serves as the Data Availability layer for Via L2, fulfilling these key functions:

- **Ensuring Data Availability**: Celestia stores the L2 batch data (pubdata) and proofs, making them publicly available.
- **Providing Inclusion Proofs**: Celestia provides cryptographic proofs that data has been included in its blockchain.
- **Decoupling Security from Ethereum**: By using Celestia for DA, Via L2 can achieve scalability without being constrained by Ethereum's data availability limitations.

This integration is critical for the security of the Via L2 system, as it ensures that all transaction data is publicly available, allowing anyone to reconstruct the state of the L2 chain if needed.

## 3. Implementation Details

### 3.1 Core Components

#### 3.1.1 CelestiaClient

The `CelestiaClient` is the primary interface for interacting with the Celestia network. It's defined in `core/lib/via_da_clients/src/celestia/client.rs` and implements the `DataAvailabilityClient` trait.

```rust
// core/lib/via_da_clients/src/celestia/client.rs
/// If no value is provided for GasPrice, then this will be serialized to `-1.0` which means the node that
/// receives the request will calculate the GasPrice for given blob.
const GAS_PRICE: f64 = -1.0;

/// An implementation of the `DataAvailabilityClient` trait that stores the pubdata in the Celestia DA.
#[derive(Clone)]
pub struct CelestiaClient {
    light_node_url: SensitiveUrl,
    inner: Arc<Client>,
    blob_size_limit: usize,
    namespace: Namespace,
}

impl CelestiaClient {
    pub async fn new(secrets: ViaDASecrets, blob_size_limit: usize) -> anyhow::Result<Self> {
        let client = Client::new(secrets.api_node_url.expose_str(), Some(&secrets.auth_token))
            .await
            .map_err(|error| anyhow!("Failed to create a client: {}", error))?;

        // connection test
        let _info = client.p2p_info().await?;

        let namespace_bytes = [b'V', b'I', b'A', 0, 0, 0, 0, 0]; // Pad with zeros to reach 8 bytes
        let namespace_bytes: &[u8] = &namespace_bytes;
        let namespace = Namespace::new_v0(namespace_bytes).map_err(|error| types::DAError {
            error: error.into(),
            is_retriable: false,
        })?;

        Ok(Self {
            light_node_url: secrets.api_node_url,
            inner: Arc::new(client),
            blob_size_limit,
            namespace,
        })
    }
}
```

The node URL and auth token come from secrets (`ViaDASecrets`), not from the plain config file. Note that `new` performs a live connection test (`p2p_info`) against the light node before returning, so a misconfigured node fails fast at wiring time.

Key methods:
- `dispatch_blob`: Submits data to Celestia and returns a blob ID.
- `get_inclusion_data`: Retrieves inclusion data for a given blob ID; if the blob decodes as a multi-chunk `ViaDaBlob` manifest, it fetches every chunk blob and reassembles the original payload.
- `blob_size_limit`: Returns the maximum size of a blob that can be dispatched.

`dispatch_blob` in full:

```rust
// core/lib/via_da_clients/src/celestia/client.rs
    async fn dispatch_blob(
        &self,
        _batch_number: u32,
        data: Vec<u8>,
    ) -> Result<types::DispatchResponse, types::DAError> {
        let share_version = celestia_types::consts::appconsts::SHARE_VERSION_ZERO;

        let blob = Blob::new(self.namespace, data.clone()).map_err(|error| types::DAError {
            error: error.into(),
            is_retriable: false,
        })?;

        let commitment_result = match Commitment::from_blob(self.namespace, share_version, &data) {
            Ok(commit) => commit,
            Err(error) => {
                return Err(types::DAError {
                    error: error.into(),
                    is_retriable: false,
                })
            }
        };

        // NOTE: during refactoring add address to the config
        // we can specify the sender address for the transaction with using TxConfig
        let tx_config = TxConfig {
            gas_price: Some(GAS_PRICE),
            ..Default::default()
        };

        let block_height = self
            .inner
            .blob_submit(&[blob], tx_config)
            .await
            .map_err(|error| types::DAError {
                error: error.into(),
                is_retriable: true,
            })?;

        // [8]byte block height ++ [32]byte commitment
        let mut blob_id = Vec::with_capacity(8 + 32);
        blob_id.extend_from_slice(&block_height.to_be_bytes());
        blob_id.extend_from_slice(&commitment_result.0);

        // Convert blob_id to a hex string
        let blob_id_str = hex::encode(blob_id);

        return Ok(types::DispatchResponse {
            blob_id: blob_id_str,
        });
    }
```

`get_inclusion_data` in full (including the manifest-resolution branch for chunked payloads):

```rust
// core/lib/via_da_clients/src/celestia/client.rs
    async fn get_inclusion_data(
        &self,
        blob_id: &str,
    ) -> Result<Option<types::InclusionData>, types::DAError> {
        let (commitment, block_height) =
            self.parse_blob_id(&blob_id)
                .map_err(|error| types::DAError {
                    error: error.into(),
                    is_retriable: true,
                })?;

        let blob = self
            .inner
            .blob_get(block_height, self.namespace, commitment)
            .await
            .map_err(|error| types::DAError {
                error: error.into(),
                is_retriable: true,
            })?;

        let data = match ViaDaBlob::from_bytes(&blob.data) {
            Some(blob) => {
                if blob.chunks == 1 {
                    blob.data
                } else {
                    let blob_ids: Vec<String> =
                        deserialize_blob_ids(&blob.data).map_err(|_| types::DAError {
                            error: anyhow!("Failed to deserialize blob ids"),
                            is_retriable: false,
                        })?;
                    if blob_ids.len() != blob.chunks {
                        return Err(types::DAError {
                            error: anyhow!(
                                "Mismatch, blob ids len [{}] != chunk size [{}]",
                                blob_ids.len(),
                                blob.chunks
                            ),
                            is_retriable: false,
                        });
                    }

                    let mut batch_blob = vec![];

                    for blob_id in blob_ids {
                        let (commitment, block_height) =
                            self.parse_blob_id(&blob_id)
                                .map_err(|error| types::DAError {
                                    error: error.into(),
                                    is_retriable: true,
                                })?;

                        let blob = self
                            .inner
                            .blob_get(block_height, self.namespace, commitment)
                            .await
                            .map_err(|error| types::DAError {
                                error: error.into(),
                                is_retriable: true,
                            })?;

                        batch_blob.extend_from_slice(&blob.data);
                    }

                    batch_blob
                }
            }
            None => blob.data,
        };

        let inclusion_data = types::InclusionData { data };

        Ok(Some(inclusion_data))
    }
```

`blob_size_limit` (and the boxed-clone helper the dispatcher uses):

```rust
// core/lib/via_da_clients/src/celestia/client.rs
    fn clone_boxed(&self) -> Box<dyn DataAvailabilityClient> {
        Box::new(self.clone())
    }

    fn blob_size_limit(&self) -> Option<usize> {
        Some(self.blob_size_limit)
    }
```

#### 3.1.2 ViaDataAvailabilityDispatcher

The `ViaDataAvailabilityDispatcher` (in `core/node/via_da_dispatcher/src/da_dispatcher.rs`) is responsible for:

1. Dispatching L2 batch data to Celestia
2. Dispatching L2 batch proofs to Celestia
3. Polling for inclusion data
4. Storing blob IDs and inclusion data in the database

```rust
pub struct ViaDataAvailabilityDispatcher {
    client: Box<dyn DataAvailabilityClient>,
    pool: ConnectionPool<Core>,
    config: DADispatcherConfig,
    blob_store: Arc<dyn ObjectStore>,
    dispatch_real_proof: bool,
}
```

### 3.2 Data Formatting and Submission

#### 3.2.1 Namespace

The Celestia client uses a specific namespace for Via L2 data:

```rust
// core/lib/via_da_clients/src/celestia/client.rs (CelestiaClient::new)
        let namespace_bytes = [b'V', b'I', b'A', 0, 0, 0, 0, 0]; // Pad with zeros to reach 8 bytes
        let namespace_bytes: &[u8] = &namespace_bytes;
        let namespace = Namespace::new_v0(namespace_bytes).map_err(|error| types::DAError {
            error: error.into(),
            is_retriable: false,
        })?;
```

This namespace ensures that Via L2 data is properly segregated within the Celestia network.

#### 3.2.2 Blob Submission

When submitting data to Celestia, the following steps occur:

1. Data is wrapped in a Celestia `Blob` with the Via namespace
2. A commitment is generated from the blob
3. The blob is submitted to Celestia with a transaction configuration
4. A blob ID is created by concatenating the block height and commitment
5. The blob ID is stored in the database

The full `dispatch_blob` body is shown in section 3.1.1. The blob ID it returns is built like this:

```rust
// core/lib/via_da_clients/src/celestia/client.rs (dispatch_blob)
        // [8]byte block height ++ [32]byte commitment
        let mut blob_id = Vec::with_capacity(8 + 32);
        blob_id.extend_from_slice(&block_height.to_be_bytes());
        blob_id.extend_from_slice(&commitment_result.0);

        // Convert blob_id to a hex string
        let blob_id_str = hex::encode(blob_id);
```

The blob ID format is: `[8]byte block height ++ [32]byte commitment`, hex-encoded for storage.

#### 3.2.3 Inclusion Data Retrieval

When retrieving inclusion data:

1. The blob ID is decoded to extract the block height and commitment
2. The blob is retrieved from Celestia using these parameters
3. The inclusion data is stored in the database

Decoding is done by the `parse_blob_id` helper:

```rust
// core/lib/via_da_clients/src/celestia/client.rs
    fn parse_blob_id(&self, blob_id: &str) -> anyhow::Result<(Commitment, u64)> {
        // [8]byte block height ++ [32]byte commitment
        let blob_id_bytes = hex::decode(blob_id).map_err(|error| types::DAError {
            error: error.into(),
            is_retriable: false,
        })?;

        let block_height =
            u64::from_be_bytes(blob_id_bytes[..8].try_into().map_err(|_| types::DAError {
                error: anyhow!("Failed to convert block height"),
                is_retriable: false,
            })?);

        let commitment_data: [u8; 32] =
            blob_id_bytes[8..40]
                .try_into()
                .map_err(|_| types::DAError {
                    error: anyhow!("Failed to convert commitment"),
                    is_retriable: false,
                })?;
        let commitment = Commitment(commitment_data);

        Ok((commitment, block_height))
    }
```

The retrieval itself (`self.inner.blob_get(block_height, self.namespace, commitment)`) and the chunk reassembly are shown in the full `get_inclusion_data` body in section 3.1.1.

### 3.3 Configuration Parameters

The Celestia client is configured using the `ViaCelestiaConfig` struct:

```rust
// core/lib/config/src/configs/via_celestia.rs
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
    pub fallback_external_node_url: Option<String>,

    /// Whether to verify data consistency between Celestia and fallback
    pub verify_consistency: bool,
}
```

Key configuration parameters:
- `da_backend`: `Celestia` (direct light-node client) or `Http` (via-core-ext proxy)
- `api_node_url`: URL of the Celestia node API (default for tests: "ws://localhost:26658")
- `blob_size_limit`: Maximum size of a blob that can be dispatched (default for tests: 1973786 bytes)
- `proof_sending_mode`: `OnlyRealProofs` or `SkipEveryProof`; the node builder derives the dispatcher's `dispatch_real_proof` flag from this (`proof_sending_mode == ProofSendingMode::OnlyRealProofs`)
- `fallback_external_node_url` and `verify_consistency`: enable the fallback client and its byte-for-byte cross-check

The auth token is supplied through `ViaDASecrets`, not this struct.

### 3.4 Database Schema

The Celestia integration uses the `via_data_availability` table to store:

- L1 batch numbers
- Whether the data is a proof or not
- Blob IDs
- A chunk `index` (added by migration `20251030121202_via_da_chunk_blob`; one row per chunk plus one for the manifest)
- Inclusion data
- Timestamps for when data was sent

The table is created by migration `20240916094925_create_via_data_availability`:

```sql
-- core/lib/dal/migrations/20240916094925_create_via_data_availability.up.sql
CREATE TABLE IF NOT EXISTS via_data_availability
(
    l1_batch_number BIGINT NOT NULL REFERENCES l1_batches (number) ON DELETE CASCADE,
    is_proof        BOOLEAN NOT NULL,

    blob_id         TEXT      NOT NULL, -- blob here is an abstract term, unrelated to any DA implementation
    inclusion_data  BYTEA,
    sent_at         TIMESTAMP NOT NULL,

    created_at      TIMESTAMP NOT NULL,
    updated_at      TIMESTAMP NOT NULL,

    PRIMARY KEY (l1_batch_number, is_proof) -- for ensuring uniqueness of the combination of l1_batch_number and is_proof
);
```

The chunking migration adds the `index` column and widens the primary key so a batch can own multiple rows (one per chunk plus the manifest):

```sql
-- core/lib/dal/migrations/20251030121202_via_da_chunk_blob.up.sql
-- 1. Add the new index column with default 0
ALTER TABLE via_data_availability
ADD COLUMN index INTEGER NOT NULL DEFAULT 0;

-- 2. Drop the old primary key constraint
ALTER TABLE via_data_availability
DROP CONSTRAINT via_data_availability_pkey;

-- 3. Add the new primary key including the index column
ALTER TABLE via_data_availability
ADD PRIMARY KEY ("l1_batch_number", "is_proof", "index");
```

### 3.5 Chunking and the Manifest Blob

Payloads larger than 500 KiB are split before dispatch (`BLOB_CHUNK_SIZE = 500 * 1024` in `da_dispatcher.rs`). The envelope type is:

```rust
// core/lib/types/src/via_da_dispatcher.rs
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ViaDaBlob {
    pub chunks: usize,
    pub data: Vec<u8>,
}
```

- A single-chunk payload is dispatched as `ViaDaBlob::new(1, data)`
- A multi-chunk payload dispatches the raw chunks first, then a final **manifest blob** `ViaDaBlob::new(chunk_count, serialized_chunk_blob_ids)` listing where the chunks live (`_dispatch_chunks()` in `da_dispatcher.rs`)

Readers resolve the manifest first, then fetch and reassemble the chunks. Each chunk gets its own row in `via_data_availability` keyed by `index`.

### 3.6 Fallback for Expired Blobs

Celestia light nodes retain blobs for a limited window (about 30 days). For older data, the `FallbackDaClient` (`core/lib/via_da_clients/src/fallback/client.rs`) wraps the primary backend with an external-node client (`core/lib/via_da_clients/src/external_node/`):

1. Try the primary backend (Celestia or HTTP proxy) first
2. If it fails or returns nothing, query the external node configured by `fallback_external_node_url`
3. If `verify_consistency` is set and both return data, compare the raw bytes; on mismatch, the error message reports the keccak256 hashes of both sides

## 4. System Interactions

### 4.1 Interaction with Sequencer/State Keeper

1. The Sequencer/State Keeper prepares L2 batch data
2. The DA Dispatcher queries the database for batches ready for dispatch
3. The DA Dispatcher sends the data to Celestia using the Celestia client
4. Blob IDs are stored in the database

### 4.2 Interaction with Prover

1. The Prover generates proofs for L2 batches
2. The DA Dispatcher queries the database for proofs ready for dispatch
3. The DA Dispatcher sends the proofs to Celestia
4. Proof blob IDs are stored in the database

### 4.3 Interaction with Verifier Network

1. The Verifier Network needs to verify that data is available
2. It can retrieve inclusion data from the database
3. This inclusion data can be used to verify that data was properly submitted to Celestia

## 5. Integration Flow

The complete flow of the Celestia integration is as follows:

1. **Initialization**:
   - The `ViaCelestiaClientWiringLayer` creates a `CelestiaClient`
   - The `DataAvailabilityDispatcherLayer` creates a `ViaDataAvailabilityDispatcher`

2. **L2 Batch Data Submission**:
   - The DA Dispatcher queries for batches ready for dispatch
   - For each batch, it calls `dispatch_blob` on the Celestia client
   - The blob ID is stored in the database

3. **Proof Submission**:
   - The DA Dispatcher queries for proofs ready for dispatch
   - For each proof, it calls `dispatch_blob` on the Celestia client
   - The proof blob ID is stored in the database

4. **Inclusion Verification**:
   - The DA Dispatcher polls for inclusion data for each blob
   - It calls `get_inclusion_data` on the Celestia client
   - The inclusion data is stored in the database

5. **Verification**:
   - The Verifier Network can verify that data is available using the inclusion data

## 6. Metrics and Monitoring

The Celestia integration includes metrics for monitoring:

- Blob dispatch latency
- Proof dispatch latency
- Inclusion latency
- Blob size
- Number of dispatch retries
- Last dispatched L1 batch
- Last dispatched proof batch
- Last included L1 batch
- Last included proof batch
- A counter of dispatcher errors

They are defined in a single `vise` metrics struct, exported under the `via_server_da_dispatcher` prefix:

```rust
// core/node/via_da_dispatcher/src/metrics.rs
use std::time::Duration;

use vise::{Buckets, Counter, Gauge, Histogram, Metrics, Unit};

/// Buckets for `blob_dispatch_latency` (from 0.1 to 120 seconds).
const DISPATCH_LATENCIES: Buckets =
    Buckets::values(&[0.1, 0.5, 1.0, 2.0, 5.0, 10.0, 30.0, 60.0, 120.0]);

#[derive(Debug, Metrics)]
#[metrics(prefix = "via_server_da_dispatcher")]
pub(super) struct DataAvailabilityDispatcherMetrics {
    /// Latency of the dispatch of the blob.
    #[metrics(buckets = DISPATCH_LATENCIES, unit = Unit::Seconds)]
    pub blob_dispatch_latency: Histogram<Duration>,
    // Latency of the dispatch of the proof.
    #[metrics(buckets = DISPATCH_LATENCIES, unit = Unit::Seconds)]
    pub proof_dispatch_latency: Histogram<Duration>,
    /// The duration between the moment when the blob is dispatched and the moment when it is included.
    #[metrics(buckets = Buckets::LATENCIES)]
    pub inclusion_latency: Histogram<Duration>,
    /// Size of the dispatched blob.
    /// Buckets are bytes ranging from 1 KB to 16 MB, which has to satisfy all blob size values.
    #[metrics(buckets = Buckets::exponential(1_024.0..=16.0 * 1_024.0 * 1_024.0, 2.0), unit = Unit::Bytes)]
    pub blob_size: Histogram<usize>,

    /// Number of transactions resent by the DA dispatcher.
    #[metrics(buckets = Buckets::linear(0.0..=10.0, 1.0))]
    pub dispatch_call_retries: Histogram<usize>,
    /// Last L1 batch that was dispatched to the DA layer.
    pub last_dispatched_l1_batch: Gauge<usize>,
    /// Last Proof batch that was dispatched to the DA layer.
    pub last_dispatched_proof_batch: Gauge<usize>,
    /// Last L1 batch that has its inclusion finalized by DA layer.
    pub last_included_l1_batch: Gauge<usize>,
    /// Last Proof batch that has its inclusion finalized by DA layer.
    pub last_included_proof_batch: Gauge<usize>,
    /// Counter to store the layer errors.
    pub errors: Counter,
}

#[vise::register]
pub(super) static METRICS: vise::Global<DataAvailabilityDispatcherMetrics> = vise::Global::new();
```

These metrics help monitor the health and performance of the Celestia integration.

## 7. Error Handling and Retries

The DA Dispatcher includes a retry mechanism for handling transient errors when interacting with Celestia:

- Retries are performed with exponential backoff (starting at 1 second, doubling, capped at 128 seconds) with random jitter of 0.8x to 1.2x on each sleep
- The maximum number of retries is configurable (`config.max_retries()`)
- Errors are classified as retriable or fatal by the client via the `is_retriable` flag on `DAError`; only retriable errors are retried

The retry loop is a free function in the dispatcher:

```rust
// core/node/via_da_dispatcher/src/da_dispatcher.rs
async fn retry<T, Fut, F>(
    max_retries: u16,
    batch_number: L1BatchNumber,
    mut f: F,
) -> Result<T, DAError>
where
    Fut: Future<Output = Result<T, DAError>>,
    F: FnMut() -> Fut,
{
    let mut retries = 1;
    let mut backoff_secs = 1;
    loop {
        match f().await {
            Ok(result) => {
                METRICS.dispatch_call_retries.observe(retries as usize);
                return Ok(result);
            }
            Err(err) => {
                if !err.is_retriable() || retries > max_retries {
                    return Err(err);
                }

                retries += 1;
                let sleep_duration = Duration::from_secs(backoff_secs)
                    .mul_f32(rand::thread_rng().gen_range(0.8..1.2));
                tracing::warn!(%err, "Failed DA dispatch request {retries}/{max_retries} for batch {batch_number}, retrying in {} milliseconds.", sleep_duration.as_millis());
                tokio::time::sleep(sleep_duration).await;

                backoff_secs = (backoff_secs * 2).min(128); // cap the back-off at 128 seconds
            }
        }
    }
}
```

It is invoked around every `dispatch_blob` call, for example in `_dispatch_chunks`:

```rust
// core/node/via_da_dispatcher/src/da_dispatcher.rs (_dispatch_chunks)
            let dispatch_response = retry(self.config.max_retries(), l1_batch_number, || {
                self.client.dispatch_blob(l1_batch_number.0, data.clone())
            })
            .await
            .with_context(|| {
                format!(
                    "failed to dispatch a blob with batch_number: {}, pubdata_len: {}, is_proof: {}",
                    l1_batch_number,
                    data.len(),
                    is_proof,
                )
            })?;
```

Above the retry layer, the dispatcher's `run` loop never aborts on a subtask failure: each of the three concurrent subtasks logs the error and bumps the `errors` counter, then the loop continues on the next polling interval:

```rust
// core/node/via_da_dispatcher/src/da_dispatcher.rs (run)
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
```

This ensures robustness in the face of network issues or temporary Celestia node unavailability.

## 8. Conclusion

The Celestia integration in Via L2 provides a robust and scalable data availability solution. By leveraging Celestia's specialized DA capabilities, Via L2 can achieve high throughput while maintaining security through data availability guarantees.