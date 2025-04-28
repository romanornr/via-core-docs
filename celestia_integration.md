# Celestia Integration in Via L2

This document describes how Via L2 integrates with Celestia as its Data Availability (DA) layer.

## 1. Overview

Via L2 uses Celestia as a Data Availability (DA) layer to ensure that L2 batch data is publicly available. The integration consists of two main components:

1. **Celestia Client**: A client that implements the `DataAvailabilityClient` trait and interacts directly with the Celestia network.
2. **DA Dispatcher**: A service that orchestrates the submission of data to Celestia and polls for inclusion proofs.

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
pub struct CelestiaClient {
    light_node_url: String,
    inner: Arc<Client>,
    blob_size_limit: usize,
    namespace: Namespace,
}
```

Key methods:
- `dispatch_blob`: Submits data to Celestia and returns a blob ID.
- `get_inclusion_data`: Retrieves inclusion data for a given blob ID.
- `blob_size_limit`: Returns the maximum size of a blob that can be dispatched.

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
let namespace_bytes = [b'V', b'I', b'A', 0, 0, 0, 0, 0]; // Pad with zeros to reach 8 bytes
let namespace = Namespace::new_v0(namespace_bytes);
```

This namespace ensures that Via L2 data is properly segregated within the Celestia network.

#### 3.2.2 Blob Submission

When submitting data to Celestia, the following steps occur:

1. Data is wrapped in a Celestia `Blob` with the Via namespace
2. A commitment is generated from the blob
3. The blob is submitted to Celestia with a transaction configuration
4. A blob ID is created by concatenating the block height and commitment
5. The blob ID is stored in the database

```rust
let blob = Blob::new(self.namespace, data.clone());
let commitment_result = Commitment::from_blob(self.namespace, share_version, &data);
let tx_config = TxConfig {
    gas_price: Some(GAS_PRICE),
    ..Default::default()
};
let block_hight = self.inner.blob_submit(&[blob], tx_config).await;
```

The blob ID format is: `[8]byte block height ++ [32]byte commitment`

#### 3.2.3 Inclusion Data Retrieval

When retrieving inclusion data:

1. The blob ID is decoded to extract the block height and commitment
2. The blob is retrieved from Celestia using these parameters
3. The inclusion data is stored in the database

```rust
let block_height = u64::from_be_bytes(blob_id_bytes[..8].try_into());
let commitment_data: [u8; 32] = blob_id_bytes[8..40].try_into();
let commitment = Commitment(commitment_data);
let blob = self.inner.blob_get(block_height, self.namespace, commitment).await;
```

### 3.3 Configuration Parameters

The Celestia client is configured using the `ViaCelestiaConfig` struct:

```rust
pub struct ViaCelestiaConfig {
    pub api_node_url: String,
    pub auth_token: String,
    pub blob_size_limit: usize,
}
```

Key configuration parameters:
- `api_node_url`: URL of the Celestia node API (default for tests: "ws://localhost:26658")
- `auth_token`: Authentication token for the Celestia node
- `blob_size_limit`: Maximum size of a blob that can be dispatched (default for tests: 1973786 bytes)

### 3.4 Database Schema

The Celestia integration uses the `via_data_availability` table to store:

- L1 batch numbers
- Whether the data is a proof or not
- Blob IDs
- Inclusion data
- Timestamps for when data was sent

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

These metrics help monitor the health and performance of the Celestia integration.

## 7. Error Handling and Retries

The DA Dispatcher includes a retry mechanism for handling transient errors when interacting with Celestia:

- Retries are performed with exponential backoff
- The maximum number of retries is configurable
- Errors are classified as retriable or fatal

This ensures robustness in the face of network issues or temporary Celestia node unavailability.

## 8. Conclusion

The Celestia integration in Via L2 provides a robust and scalable data availability solution. By leveraging Celestia's specialized DA capabilities, Via L2 can achieve high throughput while maintaining security through data availability guarantees.