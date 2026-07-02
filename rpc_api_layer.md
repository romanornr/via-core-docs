# Via L2 Bitcoin ZK-Rollup: RPC/API Layer Documentation

> **Verification note (2026-07-02):** This document was rewritten against the `via-core` source tree.
> Earlier versions of this file were auto-generated and contained fabricated RPC methods
> (`via_getWalletUTXOs`, `via_getBitcoinMempoolInfo`, `via_getProtocolVersion`, `via_getBridgeAddress`,
> `coordinator_submitPartialSignature`, `web3_sha3`), fabricated TOML configuration blocks
> (`[bitcoin_rpc]`, `[server.components]`, `[security]`), and a fabricated `MempoolInfo` struct.
> All code blocks below are verbatim from the repository, with the source path in a leading comment.

## 1. Overview

The RPC/API Layer in Via L2 is the primary interface for users and applications to interact with the rollup. It provides a set of JSON-RPC endpoints that allow clients to query the state of the blockchain, submit transactions, and subscribe to events. The API layer is designed to be compatible with Ethereum's JSON-RPC interface, making it easy for existing Ethereum tools and libraries to work with Via L2.

Almost all of this layer is inherited zksync-era machinery: the API server lives in `core/node/api_server`, the RPC trait definitions live in `core/lib/web3_decl/src/namespaces/`, and the Via fork adds exactly one custom namespace (`via`) with two active methods. The verifier coordinator also runs an HTTP API, but it is a separate axum REST service (see section 5), not part of the web3 JSON-RPC server.

## 2. Architecture

### 2.1 Core Components

1. **API Server** (`core/node/api_server`): handles HTTP and WebSocket requests, routes them to namespace handlers, and returns responses.
2. **Namespaces**: logical groupings of related API methods (`eth`, `net`, `web3`, `debug`, `zks`, `en`, `pubsub`, `snapshots`, `unstable`, `via`).
3. **Transaction Sender** (`core/node/api_server/src/tx_sender`): handles transaction submission and validation.
4. **State Access**: provides access to the blockchain state through the DAL (`zksync_dal`) and a Postgres connection pool.
5. **Filters and Subscriptions**: manages event filters and WebSocket subscriptions (the `pubsub` namespace).

### 2.2 Role in Via L2 Architecture

The RPC/API Layer serves as the primary interface between external users/applications and the Via L2 system. It:

- Receives transaction submissions from users and forwards them to the mempool via the tx sender
- Provides query capabilities for blockchain state, transactions, blocks, and events
- Offers WebSocket subscriptions for real-time updates
- Exposes ZKSync-specific functionality through the `zks` namespace
- Exposes Via-specific data (Bitcoin network, DA blob data) through the `via` namespace

## 3. API Standards and Namespaces

The full set of namespaces known to the server, and the default set enabled when `api_namespaces` is not configured:

```rust
// core/node/api_server/src/web3/mod.rs
pub enum Namespace {
    Eth,
    Net,
    Web3,
    Debug,
    Zks,
    En,
    Pubsub,
    Snapshots,
    Unstable,
    Via,
}

impl Namespace {
    pub const DEFAULT: &'static [Self] = &[
        Self::Eth,
        Self::Net,
        Self::Web3,
        Self::Zks,
        Self::En,
        Self::Pubsub,
        Self::Via,
    ];
}
```

The `via` namespace is enabled by default. `Debug` is added when the state keeper is configured with `save_call_traces`, and `Snapshots` is always pushed by the node builder (see section 4.1).

### 3.1 Standard Ethereum JSON-RPC Namespaces

All method lists below are verified against the `#[method(name = ...)]` attributes in the trait definitions under `core/lib/web3_decl/src/namespaces/`. This code is inherited from zksync-era.

#### `eth` Namespace

Defined in `core/lib/web3_decl/src/namespaces/eth.rs`:

- **State Access**: `eth_getBalance`, `eth_getStorageAt`, `eth_getCode`, `eth_getTransactionCount`
- **Block Information**: `eth_blockNumber`, `eth_getBlockByNumber`, `eth_getBlockByHash`, `eth_getBlockTransactionCountByNumber`, `eth_getBlockTransactionCountByHash`, `eth_getBlockReceipts`
- **Transaction Operations**: `eth_sendRawTransaction`, `eth_getTransactionByHash`, `eth_getTransactionByBlockHashAndIndex`, `eth_getTransactionByBlockNumberAndIndex`, `eth_getTransactionReceipt`, `eth_estimateGas`, `eth_call`
- **Event Filtering**: `eth_getLogs`, `eth_newFilter`, `eth_newBlockFilter`, `eth_newPendingTransactionFilter`, `eth_getFilterChanges`, `eth_getFilterLogs`, `eth_uninstallFilter`
- **Chain Information**: `eth_chainId`, `eth_gasPrice`, `eth_feeHistory`, `eth_syncing`, `eth_maxPriorityFeePerGas`
- **Compatibility stubs**: `eth_accounts`, `eth_protocolVersion`, `eth_coinbase`, `eth_getCompilers`, `eth_hashrate`, `eth_getUncleCountByBlockHash`, `eth_getUncleCountByBlockNumber`, `eth_mining`
- **Subscriptions**: `eth_subscribe` (served by the `pubsub` namespace over WebSocket)

#### `net` Namespace

Defined in `core/lib/web3_decl/src/namespaces/net.rs`:

- `net_version`: Returns the current network ID
- `net_listening`: Returns if client is actively listening for network connections
- `net_peerCount`: Returns number of peers currently connected to the client

#### `web3` Namespace

Defined in `core/lib/web3_decl/src/namespaces/web3.rs`. It contains exactly one method:

- `web3_clientVersion`: Returns the current client version

Note: `web3_sha3` is NOT implemented in this codebase, contrary to earlier versions of this document.

### 3.2 Via and ZKSync-Specific Namespaces

#### `via` Namespace

The complete trait definition. There are only two active methods; `getBridgeAddress` exists in the source but is commented out and therefore not served:

```rust
// core/lib/web3_decl/src/namespaces/via.rs
use bitcoin::Network;
#[cfg_attr(not(feature = "server"), allow(unused_imports))]
use jsonrpsee::core::RpcResult;
use jsonrpsee::proc_macros::rpc;

use crate::client::{ForWeb3Network, L2};

#[derive(Default, Debug, Clone, serde::Serialize, serde::Deserialize)]
pub struct DaBlobData {
    pub is_proof: bool,
    pub pub_data: String,   // hex-encoded blob data
    pub proof_data: String, // hex-encoded blob data
}

#[cfg_attr(
    feature = "server",
    rpc(server, client, namespace = "via", client_bounds(Self: ForWeb3Network<Net = L2>))
)]
#[cfg_attr(
    not(feature = "server"),
    rpc(client, namespace = "via", client_bounds(Self: ForWeb3Network<Net = L2>))
)]
pub trait ViaNamespace {
    // #[method(name = "getBridgeAddress")]
    // async fn get_bridge_address(&self) -> RpcResult<String>;

    #[method(name = "getBitcoinNetwork")]
    async fn get_bitcoin_network(&self) -> RpcResult<Network>;

    /// Get DA blob data for a specific blob_id
    #[method(name = "getDaBlobData")]
    async fn get_da_blob_data(&self, blob_id: String) -> RpcResult<Option<DaBlobData>>;
}
```

The server-side implementation:

```rust
// core/node/api_server/src/web3/namespaces/via.rs
impl ViaNamespace {
    pub fn new(state: RpcState) -> Self {
        Self { state }
    }

    pub(crate) fn current_method(&self) -> &MethodTracer {
        &self.state.current_method
    }

    pub fn get_bitcoin_network_impl(&self) -> Network {
        self.state.api_config.via_network
    }

    pub async fn get_da_blob_data_impl(
        &self,
        blob_id: String,
    ) -> Result<Option<DaBlobData>, Web3Error> {
        let mut conn = self
            .state
            .connection_pool
            .connection()
            .await
            .map_err(DalError::generalize)?;

        let Some(is_proof) = conn
            .via_data_availability_dal()
            .get_blob_type(&blob_id)
            .await
            .map_err(DalError::generalize)?
        else {
            return Ok(None);
        };

        let mut blob_data = DaBlobData {
            is_proof,
            ..Default::default()
        };

        if is_proof {
            let Some((mut prove_batch, _)) = conn
                .via_data_availability_dal()
                .get_proof_data_by_blob_id(&blob_id)
                .await
                .map_err(DalError::generalize)?
            else {
                return Ok(None);
            };

            prove_batch.should_verify = self.state.api_config.via_dispatch_real_proof;

            let proof_data = match bincode::serialize(&prove_batch) {
                Ok(data) => data,
                Err(e) => {
                    return Err(Web3Error::InternalError(anyhow::anyhow!(
                        "Error serializing prove_batch: {e}"
                    )));
                }
            };

            blob_data.proof_data = hex::encode(proof_data);
        } else {
            let Some((_, pub_data)) = conn
                .via_data_availability_dal()
                .get_da_blob_pub_data_by_blob_id(&blob_id)
                .await
                .map_err(DalError::generalize)?
            else {
                return Ok(None);
            };
            blob_data.pub_data = hex::encode(pub_data);
        }

        Ok(Some(blob_data))
    }
}
```

- **`via_getBitcoinNetwork`**: returns the `bitcoin::Network` the node is configured for. The value comes from `InternalApiConfig::via_network`, which the node builder derives from `ViaBtcClientConfig::network()` (defaulting to `Regtest` if the configured string does not parse).
- **`via_getDaBlobData`**: looks up a DA blob by `blob_id` in the `via_data_availability` DAL and returns either hex-encoded pubdata or a hex-encoded bincode-serialized proof batch, depending on the blob type. The `via_getDaBlobData` method is consumed by the external node DA client (`core/lib/via_da_clients/src/external_node/client.rs`).

The following methods listed in earlier versions of this document **do not exist** anywhere in the codebase: `via_getBitcoinBlockNumber`, `via_getBitcoinBlock`, `via_getBitcoinTransaction`, `via_getBitcoinMempoolInfo`, `via_getVerifierAddresses`, `via_getProtocolVersion`, `via_getCurrentProtocolVersion`, `via_getBlockDetails`, `via_getTransactionDetails`, `via_getRawBlockTransactions`, `via_estimateDepositFee`, `via_getWithdrawalStatus`, `via_getWalletUTXOs`, `via_getBridgeAddress` (commented out).

#### `zks` Namespace

Defined in `core/lib/web3_decl/src/namespaces/zks.rs` (inherited zksync-era code):

- **Fee Estimation**: `zks_estimateFee`, `zks_estimateGasL1ToL2`, `zks_getFeeParams`, `zks_getBatchFeeInput`, `zks_getL1GasPrice`
- **Contract Information**: `zks_getMainContract`, `zks_getBridgehubContract`, `zks_getTestnetPaymaster`, `zks_getTimestampAsserter`, `zks_getBridgeContracts`, `zks_getBaseTokenL1Address`
- **Chain Information**: `zks_L1ChainId`, `zks_L1BatchNumber`, `zks_getL1BatchBlockRange`, `zks_getProtocolVersion`
- **Token Information**: `zks_getConfirmedTokens`, `zks_getAllAccountBalances`
- **Block and Transaction Details**: `zks_getBlockDetails`, `zks_getTransactionDetails`, `zks_getRawBlockTransactions`, `zks_getL1BatchDetails`, `zks_getBytecodeByHash`
- **Proof Generation**: `zks_getL2ToL1MsgProof`, `zks_getL2ToL1LogProof`, `zks_getProof`
- **Transaction Submission**: `zks_sendRawTransactionWithDetailedOutput`

#### `en` Namespace

External Node synchronization methods, defined in `core/lib/web3_decl/src/namespaces/en.rs`: `en_syncL2Block`, `en_consensusGenesis`, `en_consensusGlobalConfig`, `en_blockMetadata`, `en_syncTokens`, `en_genesisConfig`, `en_attestationStatus`, `en_whitelistedTokensForAA`.

#### `debug` Namespace

Tracing methods, defined in `core/lib/web3_decl/src/namespaces/debug.rs`: `debug_traceBlockByNumber`, `debug_traceBlockByHash`, `debug_traceCall`, `debug_traceTransaction`.

#### `snapshots` Namespace

Defined in `core/lib/web3_decl/src/namespaces/snapshots.rs`: `snapshots_getAllSnapshots`, `snapshots_getSnapshot`.

#### `unstable` Namespace

Defined in `core/lib/web3_decl/src/namespaces/unstable.rs` (not in the default set): `unstable_getTransactionExecutionInfo`, `unstable_getTeeProofs`, `unstable_getChainLogProof`.

### 3.3 WebSocket Subscriptions

Via L2 supports WebSocket subscriptions through the `eth_subscribe` method (the `Pubsub` namespace, enabled by default on the WebSocket server), allowing clients to receive real-time updates for new blocks, new pending transactions, and logs matching specific criteria.

## 4. Implementation Details

### 4.1 Server Wiring in the Node Builder

The HTTP and WebSocket web3 servers are wired in the via_server node builder. This is where the namespace list, limits, and the Via-specific `InternalApiConfig` parameters (Bitcoin network, real-proof mode) are injected:

```rust
// core/bin/via_server/src/node_builder.rs
fn add_http_web3_api_layer(mut self) -> anyhow::Result<Self> {
    let rpc_config = try_load_config!(self.configs.api_config).web3_json_rpc;
    let state_keeper_config = try_load_config!(self.configs.state_keeper_config);
    let with_debug_namespace = state_keeper_config.save_call_traces;

    let mut namespaces = if let Some(namespaces) = &rpc_config.api_namespaces {
        namespaces
            .iter()
            .map(|a| a.parse())
            .collect::<Result<_, _>>()?
    } else {
        Namespace::DEFAULT.to_vec()
    };
    if with_debug_namespace {
        namespaces.push(Namespace::Debug)
    }
    namespaces.push(Namespace::Snapshots);

    let optional_config = Web3ServerOptionalConfig {
        namespaces: Some(namespaces),
        filters_limit: Some(rpc_config.filters_limit()),
        subscriptions_limit: Some(rpc_config.subscriptions_limit()),
        batch_request_size_limit: Some(rpc_config.max_batch_request_size()),
        response_body_size_limit: Some(rpc_config.max_response_body_size()),
        ..Default::default()
    };
    let via_btc_client_config = try_load_config!(self.configs.via_btc_client_config);
    let via_celestia_config = try_load_config!(self.configs.via_celestia_config);

    self.node.add_layer(Web3ServerLayer::http(
        rpc_config.http_port,
        InternalApiConfig::new(
            &rpc_config,
            &self.contracts_config,
            &self.genesis_config,
            Some(via_btc_client_config.network()),
            via_celestia_config.proof_sending_mode == ProofSendingMode::OnlyRealProofs,
        ),
        optional_config,
    ));

    Ok(self)
}
```

An analogous `add_ws_web3_api_layer` (same file, using `rpc_config.ws_port` and additionally `websocket_requests_per_minute_limit` and `extended_api_tracing`) wires the WebSocket server.

### 4.2 The `--components` CLI Flag

The `via_server` binary accepts a comma-separated `--components` flag. The real default and component set:

```rust
// core/bin/via_server/src/main.rs
#[derive(Debug, Parser)]
#[command(author = "Via Protocol", version, about = "Via sequencer node", long_about = None)]
struct Cli {
    /// Generate genesis block for the first contract deployment using temporary DB.
    #[arg(long)]
    genesis: bool,

    /// Comma-separated list of components to launch.
    #[arg(
        long,
        default_value = "api,btc,tree,tree_api,state_keeper,housekeeper,proof_data_handler,commitment_generator,celestia,da_dispatcher,vm_runner_protective_reads,vm_runner_bwip"
    )]
    components: ComponentsToRun,
    // ...
}
```

The component enum and the string-to-component mapping:

```rust
// core/lib/zksync_core_leftovers/src/lib.rs
#[derive(Debug, Clone, Copy, PartialEq)]
pub enum ViaComponent {
    /// Public Web3 API running on HTTP server.
    HttpApi,
    /// Public Web3 API (including PubSub) running on WebSocket server.
    WsApi,
    /// Metadata calculator.
    Tree,
    /// Merkle tree API.
    TreeApi,
    /// State keeper.
    StateKeeper,
    /// Component for housekeeping task such as cleaning blobs from GCS, reporting metrics etc.
    Housekeeper,
    /// Component for exposing APIs to prover for providing proof generation data and accepting proofs.
    ProofDataHandler,
    /// Component generating commitment for L1 batches.
    CommitmentGenerator,
    /// Component sending a pubdata to the DA layers.
    DADispatcher,
    /// VM runner-based component that saves protective reads to Postgres.
    VmRunnerProtectiveReads,
    /// VM runner-based component that saves VM execution data for basic witness generation.
    VmRunnerBwip,
    /// Component that interacts with Bitcoin network
    Btc,
    /// Component that writes data to Celestia network
    Celestia,
}
```

The `"api"` string expands to both API servers, and individual servers can be selected with `"http_api"` / `"ws_api"`:

```rust
// core/lib/zksync_core_leftovers/src/lib.rs
impl FromStr for ViaComponents {
    type Err = String;

    fn from_str(s: &str) -> Result<ViaComponents, String> {
        match s {
            "api" => Ok(ViaComponents(vec![
                ViaComponent::HttpApi,
                ViaComponent::WsApi,
            ])),
            "http_api" => Ok(ViaComponents(vec![ViaComponent::HttpApi])),
            "ws_api" => Ok(ViaComponents(vec![ViaComponent::WsApi])),
            "tree" => Ok(ViaComponents(vec![ViaComponent::Tree])),
            "tree_api" => Ok(ViaComponents(vec![ViaComponent::TreeApi])),
            "state_keeper" => Ok(ViaComponents(vec![ViaComponent::StateKeeper])),
            "housekeeper" => Ok(ViaComponents(vec![ViaComponent::Housekeeper])),
            "proof_data_handler" => Ok(ViaComponents(vec![ViaComponent::ProofDataHandler])),
            "commitment_generator" => Ok(ViaComponents(vec![ViaComponent::CommitmentGenerator])),
            "da_dispatcher" => Ok(ViaComponents(vec![ViaComponent::DADispatcher])),
            "vm_runner_protective_reads" => {
                Ok(ViaComponents(vec![ViaComponent::VmRunnerProtectiveReads]))
            }
            "vm_runner_bwip" => Ok(ViaComponents(vec![ViaComponent::VmRunnerBwip])),
            "btc" => Ok(ViaComponents(vec![ViaComponent::Btc])),
            "celestia" => Ok(ViaComponents(vec![ViaComponent::Celestia])),
            other => Err(format!("{} is not a valid component name", other)),
        }
    }
}
```

Note: there is no `mempool`, `eth_sender`, or `btc_sender` component name, and the Via server does not support `--components all`. Earlier versions of this document listed a fabricated component set.

### 4.3 Bitcoin RPC Client (Internal, Not Exposed Over Web3 RPC)

The Bitcoin RPC client in `core/lib/via_btc_client` is used internally by node components (btc watch, btc sender, verifier). None of its methods are exposed as JSON-RPC endpoints on the web3 API server. The complete `BitcoinRpc` trait:

```rust
// core/lib/via_btc_client/src/traits.rs
#[async_trait]
pub trait BitcoinRpc: Send + Sync + Debug {
    async fn get_balance(&self, address: &Address) -> BitcoinRpcResult<u64>;
    async fn get_balance_scan(&self, address: &Address) -> BitcoinRpcResult<u64>;
    async fn send_raw_transaction(&self, tx_hex: &str) -> BitcoinRpcResult<Txid>;
    async fn list_unspent_based_on_node_wallet(
        &self,
        address: &Address,
    ) -> BitcoinRpcResult<Vec<OutPoint>>;
    async fn list_unspent(&self, address: &Address) -> BitcoinRpcResult<Vec<OutPoint>>;
    async fn get_transaction(&self, tx_id: &Txid) -> BitcoinRpcResult<Transaction>;
    async fn get_block_count(&self) -> BitcoinRpcResult<u64>;
    async fn get_block_by_height(&self, block_height: u128) -> BitcoinRpcResult<Block>;

    async fn get_block_by_hash(&self, block_hash: &BlockHash) -> BitcoinRpcResult<Block>;
    async fn get_best_block_hash(&self) -> BitcoinRpcResult<bitcoin::BlockHash>;
    async fn get_block_stats(&self, height: u64) -> BitcoinRpcResult<GetBlockStatsResult>;
    async fn get_raw_transaction_info(
        &self,
        txid: &Txid,
    ) -> BitcoinRpcResult<bitcoincore_rpc::json::GetRawTransactionResult>;
    async fn estimate_smart_fee(
        &self,
        conf_target: u16,
        estimate_mode: Option<bitcoincore_rpc::json::EstimateMode>,
    ) -> BitcoinRpcResult<bitcoincore_rpc::json::EstimateSmartFeeResult>;
    async fn get_blockchain_info(&self) -> BitcoinRpcResult<GetBlockchainInfoResult>;
    async fn get_mempool_info(&self) -> BitcoinRpcResult<GetMempoolInfoResult>;
}
```

`get_mempool_info` returns `GetMempoolInfoResult` from the `bitcoincore_rpc` crate. There is no Via-defined `MempoolInfo` struct and no `via_getBitcoinMempoolInfo` RPC method; earlier versions of this document fabricated both.

Wallet-specific Bitcoin RPC URLs do exist, but they are built by config code (not by a TOML `[bitcoin_rpc.wallets]` table, which was fabricated):

```rust
// core/lib/config/src/configs/via_btc_client.rs
impl ViaBtcClientConfig {
    /// Returns the Bitcoin network
    pub fn network(&self) -> Network {
        Network::from_str(&self.network).unwrap_or(Network::Regtest)
    }

    pub fn rpc_url(&self, base_rpc_url: String, wallet: String) -> String {
        if self.network() == Network::Regtest {
            return base_rpc_url;
        }
        // Include the wallet endpoint to fetch the utxos.
        format!("{}wallet/{}", base_rpc_url, wallet)
    }

    pub fn use_rpc_for_fee_rate(&self) -> bool {
        if let Some(use_external_api) = self.use_rpc_for_fee_rate {
            return use_external_api;
        }
        true
    }
}
```

### 4.4 Request Routing

1. Incoming requests are received by the HTTP or WebSocket server (jsonrpsee)
2. Requests are parsed and validated
3. The appropriate namespace and method handler is identified
4. The method handler processes the request through `RpcState` (connection pool, tx sender, caches)
5. The response is formatted and returned to the client

### 4.5 State Access

The API server accesses blockchain state through:

- **Connection Pool**: manages Postgres connections (`zksync_dal::ConnectionPool<Core>`)
- **Storage DAL (Data Access Layer)**: provides methods for querying blockchain data, including the Via-specific `via_data_availability_dal` used by `via_getDaBlobData`
- **Caching**: Postgres storage caches for the VM (factory deps, initial writes, latest values) and a mempool cache, sized by `Web3JsonRpcConfig`

## 5. Coordinator REST API (Separate Service)

The verifier coordinator API is **not** part of the web3 JSON-RPC server. It is a standalone axum HTTP REST service started by the verifier node (`via_verifier/node/via_verifier_coordinator`), listening on `ViaVerifierConfig::coordinator_port`. Earlier versions of this document incorrectly presented it as a JSON-RPC namespace with methods like `coordinator_submitPartialSignature` and bearer-token authentication; none of that exists.

### 5.1 Routes

```rust
// via_verifier/node/via_verifier_coordinator/src/coordinator/api_decl.rs
pub fn into_router(self) -> axum::Router<()> {
    // Wrap the API state in an Arc.
    let shared_state = Arc::new(self);

    // Create middleware layers using from_fn_with_state.
    let auth_mw =
        middleware::from_fn_with_state(shared_state.clone(), auth_middleware::auth_middleware);
    let body_mw =
        middleware::from_fn_with_state(shared_state.clone(), auth_middleware::extract_body);

    let router = axum::Router::new()
        .route("/new", axum::routing::post(Self::new_session))
        .route("/", axum::routing::get(Self::get_session))
        .route(
            "/signature",
            axum::routing::post(Self::submit_partial_signature),
        )
        .route(
            "/signature",
            axum::routing::get(Self::get_submitted_signatures),
        )
        .route("/nonce", axum::routing::post(Self::submit_nonce))
        .route("/nonce", axum::routing::get(Self::get_nonces))
        .route_layer(body_mw)
        .route_layer(auth_mw)
        .with_state(shared_state.clone())
        .layer(
            ServiceBuilder::new()
                .layer(TimeoutLayer::new(API_TIMEOUT))
                .layer(CorsLayer::permissive())
                .into_inner(),
        );

    axum::Router::new().nest("/session", router)
}
```

The resulting endpoints (all nested under `/session`, handlers in `coordinator/api_impl.rs`):

| Method | Path                 | Handler                    |
|--------|----------------------|----------------------------|
| POST   | `/session/new`       | `new_session`              |
| GET    | `/session/`          | `get_session`              |
| POST   | `/session/signature` | `submit_partial_signature` |
| GET    | `/session/signature` | `get_submitted_signatures` |
| POST   | `/session/nonce`     | `submit_nonce`             |
| GET    | `/session/nonce`     | `get_nonces`               |

`API_TIMEOUT` is 30 seconds (`const API_TIMEOUT: Duration = Duration::from_secs(30);` in the same file). The server is bound and served in `coordinator/api.rs` (`start_coordinator_server`), using `config.bind_addr()` from `ViaVerifierConfig`.

### 5.2 Authentication

Every request passes through an auth middleware that requires four headers: `X-Timestamp`, `X-Verifier-Index`, `X-Signature`, and `X-Sequencer-Version`. The middleware:

1. Validates the timestamp format and rejects timestamps older than `verifier_request_timeout` or more than 30 seconds in the future (`MAX_CLOCK_SKEW_SECONDS`)
2. Looks up the verifier's public key by index in the configured `verifiers_pub_keys`
3. Verifies a secp256k1 ECDSA signature over the JSON payload `{"timestamp", "verifier_index", "sequencer_version"}`
4. Checks the caller's protocol version against the latest protocol semantic version in the database

The signature scheme (sign and verify) is SHA-256 over the serialized JSON payload, secp256k1 ECDSA, base64-encoded compact signature:

```rust
// via_verifier/node/via_verifier_coordinator/src/auth.rs
/// Signs a request payload using the verifier's private key.
pub fn sign_request<T: Serialize>(payload: &T, secret_key: &SecretKey) -> anyhow::Result<String> {
    let secp = Secp256k1::new();

    // Serialize and hash the payload.
    let payload_bytes =
        serde_json::to_vec(payload).with_context(|| "Failed to serialize payload")?;
    let hash = Sha256::digest(&payload_bytes);
    let message =
        Message::from_digest_slice(hash.as_ref()).with_context(|| "Hash is not 32 bytes")?;

    // Sign the message.
    let sig = secp.sign_ecdsa(&message, secret_key);
    // Encode the compact 64-byte signature in base64.
    let sig_bytes = sig.serialize_compact();
    Ok(base64::engine::general_purpose::STANDARD.encode(sig_bytes))
}

/// Verifies a request signature using the verifier's public key.
pub fn verify_signature<T: Serialize>(
    payload: &T,
    signature_b64: &str,
    public_key: &PublicKey,
) -> anyhow::Result<bool> {
    let secp = Secp256k1::new();

    // Decode the base64 signature.
    let sig_bytes = base64::engine::general_purpose::STANDARD
        .decode(signature_b64)
        .with_context(|| "Failed to decode base64 signature")?;
    let sig = bitcoin::secp256k1::ecdsa::Signature::from_compact(&sig_bytes)
        .with_context(|| "Failed to parse signature from compact form")?;

    // Serialize and hash the payload.
    let payload_bytes =
        serde_json::to_vec(payload).with_context(|| "Failed to serialize payload")?;
    let hash = Sha256::digest(&payload_bytes);
    let message =
        Message::from_digest_slice(hash.as_ref()).with_context(|| "Hash is not 32 bytes")?;

    // Verify the signature.
    Ok(secp.verify_ecdsa(&message, &sig, public_key).is_ok())
}
```

The middleware's signature check over the header payload:

```rust
// via_verifier/node/via_verifier_coordinator/src/coordinator/auth_middleware.rs
// Get the public key for this verifier
let public_key = &state.state.verifiers_pub_keys[verifier_index];

//  verify timestamp + verifier_index
let payload = serde_json::json!({
    "timestamp": timestamp,
    "verifier_index": verifier_index.to_string(),
    "sequencer_version": sequencer_version
});

// Verify the signature
if !crate::auth::verify_signature(&payload, signature, public_key)
    .map_err(|_| ApiError::InternalServerError("Signature verification failed".into()))?
{
    return Err(ApiError::Unauthorized(
        "Invalid authentication signature".into(),
    ));
}
```

There are no API keys, no bearer tokens, no nonce headers, and no request-body signing of the form `sign(method + url + timestamp + body_hash)`; those were fabricated.

## 6. Interactions with Other Components

### 6.1 Mempool

- The API server submits transactions through the `tx_sender` component (`TxSenderLayer` in the node builder)
- A mempool cache (`MempoolCacheLayer`, sized by `mempool_cache_size` / `mempool_cache_update_interval`) serves pending-transaction queries

### 6.2 State Management

- The API server queries blockchain state from Postgres via the DAL
- State queries include account balances, storage, code, and transaction data
- The API server can execute view calls against the current state through the VM (bounded by `vm_concurrency_limit`)

### 6.3 Block Information

- The API server retrieves block, receipt, and log information from the database
- The API server tracks the latest sealed block number for consistent responses

### 6.4 Bitcoin Integration

- The web3 API's only Bitcoin-facing surface is the `via` namespace (network identifier and DA blob data)
- All other Bitcoin interaction (UTXOs, fees, mempool, inscriptions) happens inside node components through `via_btc_client` and is not exposed over RPC

## 7. API Method Examples

### 7.1 Via-Specific Methods

```json
// Request
{
  "jsonrpc": "2.0",
  "method": "via_getBitcoinNetwork",
  "params": [],
  "id": 1
}
```

Returns the serialized `bitcoin::Network` value the node is configured for.

```json
// Request
{
  "jsonrpc": "2.0",
  "method": "via_getDaBlobData",
  "params": ["<blob_id>"],
  "id": 1
}
```

Returns `null` if the blob is unknown, otherwise a `DaBlobData` object with fields `is_proof` (bool), `pub_data` (hex string), and `proof_data` (hex string), as defined in `core/lib/web3_decl/src/namespaces/via.rs`.

### 7.2 L1 Batch Details

```json
// Request
{
  "jsonrpc": "2.0",
  "method": "zks_getL1BatchDetails",
  "params": [123],
  "id": 1
}
```

The response is the `L1BatchDetails` type. Its real shape (camelCase over the wire):

```rust
// core/lib/types/src/api/mod.rs
pub struct L1BatchDetails {
    pub number: L1BatchNumber,
    #[serde(flatten)]
    pub base: BlockDetailsBase,
}

pub struct BlockDetailsBase {
    pub timestamp: u64,
    pub l1_tx_count: usize,
    pub l2_tx_count: usize,
    /// Hash for an L2 block, or the root hash (aka state hash) for an L1 batch.
    pub root_hash: Option<H256>,
    pub status: BlockStatus,
    pub commit_tx_hash: Option<H256>,
    pub committed_at: Option<DateTime<Utc>>,
    pub commit_chain_id: Option<SLChainId>,
    pub prove_tx_hash: Option<H256>,
    pub proven_at: Option<DateTime<Utc>>,
    pub prove_chain_id: Option<SLChainId>,
    pub execute_tx_hash: Option<H256>,
    pub executed_at: Option<DateTime<Utc>>,
    pub execute_chain_id: Option<SLChainId>,
    pub l1_gas_price: u64,
    pub l2_fair_gas_price: u64,
    // Cost of publishing one byte (in wei).
    pub fair_pubdata_price: Option<u64>,
    pub base_system_contracts_hashes: BaseSystemContractsHashes,
}
```

There is no `data_availability` field in the response; earlier versions of this document fabricated one.

### 7.3 Transaction Submission

```json
// Request
{
  "jsonrpc": "2.0",
  "method": "eth_sendRawTransaction",
  "params": ["0x..."],
  "id": 1
}

// Response
{
  "jsonrpc": "2.0",
  "result": "0x...",
  "id": 1
}
```

### 7.4 State Query

```json
// Request
{
  "jsonrpc": "2.0",
  "method": "eth_getBalance",
  "params": ["0x...", "latest"],
  "id": 1
}

// Response
{
  "jsonrpc": "2.0",
  "result": "0x...",
  "id": 1
}
```

### 7.5 ZKSync-Specific Method

```json
// Request
{
  "jsonrpc": "2.0",
  "method": "zks_estimateFee",
  "params": [{
    "from": "0x...",
    "to": "0x...",
    "value": "0x...",
    "data": "0x..."
  }],
  "id": 1
}

// Response
{
  "jsonrpc": "2.0",
  "result": {
    "gas_limit": "0x...",
    "gas_per_pubdata_limit": "0x...",
    "max_fee_per_gas": "0x...",
    "max_priority_fee_per_gas": "0x..."
  },
  "id": 1
}
```

## 8. Configuration Parameters

### 8.1 Web3 JSON-RPC Configuration

The API server is configured through `Web3JsonRpcConfig` (env-based; the Via server does not support YAML config files at this point, per `core/bin/via_server/src/main.rs`). The key fields, from the source:

```rust
// core/lib/config/src/configs/api.rs (field excerpt of Web3JsonRpcConfig)
pub struct Web3JsonRpcConfig {
    pub http_port: u16,
    pub http_url: String,
    pub ws_port: u16,
    pub ws_url: String,
    pub req_entities_limit: Option<u32>,
    pub filters_disabled: bool,
    pub filters_limit: Option<u32>,
    pub subscriptions_limit: Option<u32>,
    pub pubsub_polling_interval: Option<u64>,
    pub max_nonce_ahead: u32,
    pub gas_price_scale_factor: f64,
    pub estimate_gas_scale_factor: f64,
    pub estimate_gas_acceptable_overestimation: u32,
    // ...
    pub max_batch_request_size: Option<usize>,
    pub max_response_body_size_mb: Option<usize>,
    pub max_response_body_size_overrides_mb: MaxResponseSizeOverrides,
    pub websocket_requests_per_minute_limit: Option<NonZeroU32>,
    pub tree_api_url: Option<String>,
    pub mempool_cache_update_interval: Option<u64>,
    pub mempool_cache_size: Option<usize>,
    pub whitelisted_tokens_for_aa: Vec<Address>,
    pub api_namespaces: Option<Vec<String>>,
    pub extended_api_tracing: bool,
    // ...
}
```

The struct also contains VM concurrency and cache-size fields exposed via accessors (`vm_concurrency_limit()`, `factory_deps_cache_size()`, `initial_writes_cache_size()`, `latest_values_cache_size()` and related); see the full definition in `core/lib/config/src/configs/api.rs`.

### 8.2 Default Ports

The base environment configuration ships these defaults:

```toml
# etc/env/base/api.toml
[api.web3_json_rpc]
# Port for the HTTP RPC API.
http_port = 3050
http_url = "http://127.0.0.1:3050"
# Port for the WebSocket RPC API.
ws_port = 3051
ws_url = "ws://127.0.0.1:3051"
req_entities_limit = 10000
filters_disabled = false
filters_limit = 10000
subscriptions_limit = 10000
# Interval between polling db for pubsub (in ms).
pubsub_polling_interval = 200
threads_per_server = 128
max_nonce_ahead = 50
gas_price_scale_factor = 1.2
estimate_gas_scale_factor = 1.2
estimate_gas_acceptable_overestimation = 1000
max_tx_size = 1000000

# Configuration for the prometheus exporter server.
[api.prometheus]
listener_port = 3312
pushgateway_url = "http://127.0.0.1:9091"
push_interval_ms = 100

# Configuration for the healtcheck server.
[api.healthcheck]
port = 3071

# Configuration for the Merkle tree API server
[api.merkle_tree]
port = 3072
```

### 8.3 Coordinator API Configuration

The coordinator REST service is configured by `ViaVerifierConfig` (`core/lib/config/src/configs/via_verifier.rs`), whose relevant fields are `role` (`ViaNodeRole`), `coordinator_port` (the bind port used by `bind_addr()`), `coordinator_http_url` (the URL verifiers use to reach the coordinator), `verifier_request_timeout` (max age in seconds for the signed `X-Timestamp` header), and `session_timeout`.

The TOML blocks in earlier versions of this document (`[bitcoin_rpc]`, `[bitcoin_rpc.wallets]`, `[server.components]`, `[security]`, `enable_wallet_operations`, `require_api_key`, and similar keys) do not exist anywhere in the repository.

## 9. Security Considerations

### 9.1 Web3 API Server

Security measures actually present in the web3 server configuration:

- **CORS**: Cross-Origin Resource Sharing configuration for HTTP requests
- **Size Limits**: `max_batch_request_size`, `max_response_body_size_mb` (with per-method overrides)
- **Rate Limiting**: `websocket_requests_per_minute_limit` for WebSocket connections
- **Concurrency Limits**: `vm_concurrency_limit` for concurrent VM executions
- **Filters and Subscriptions Limits**: `filters_limit`, `subscriptions_limit` (and `filters_disabled`)

### 9.2 Coordinator API

- **Per-request ECDSA authentication**: secp256k1 signature over `{timestamp, verifier_index, sequencer_version}` verified against the configured verifier public keys (section 5.2)
- **Timestamp freshness**: requests older than `verifier_request_timeout` seconds or more than 30 seconds in the future are rejected
- **Protocol version gating**: callers running an outdated protocol semantic version are rejected
- **Request timeout**: a 30-second `TimeoutLayer` wraps all routes

Claims in earlier versions of this document about API keys, role-based access control, TLS enforcement, IP whitelisting, and DDoS protection are not implemented in this repository.

## 10. Error Handling

- The web3 server returns standard JSON-RPC errors produced by the jsonrpsee backend; handler errors are represented by `Web3Error` (`core/lib/web3_decl/src/error.rs`), e.g. `via_getDaBlobData` maps serialization failures to `Web3Error::InternalError`.
- The coordinator REST API returns plain HTTP status codes via `ApiError` (`via_verifier/node/via_verifier_coordinator/src/coordinator/error.rs`), e.g. `401 Unauthorized` for failed authentication and `500 Internal Server Error` for signature verification failures. It does not speak JSON-RPC.

## 11. Conclusion

The RPC/API Layer in Via L2 is largely inherited zksync-era machinery: a jsonrpsee-based HTTP/WebSocket server exposing the standard `eth`, `net`, `web3`, `zks`, `en`, `debug`, `pubsub`, and `snapshots` namespaces, configured through `Web3JsonRpcConfig` and assembled by the via_server node builder. Via's additions to this layer are deliberately small:

- A `via` namespace with two methods (`via_getBitcoinNetwork`, `via_getDaBlobData`) serving the Bitcoin network identifier and DA blob data to clients such as the external node
- Via-specific parameters threaded into `InternalApiConfig` (Bitcoin network, real-proof dispatch mode)
- A component selection scheme (`--components`) with Bitcoin- and Celestia-related components (`btc`, `celestia`, `da_dispatcher`)

The verifier coordinator's HTTP API is a separate axum REST service under `/session`, authenticated per request with secp256k1 ECDSA signatures, and should not be confused with the web3 JSON-RPC interface.
