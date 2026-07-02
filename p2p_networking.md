# P2P Networking Layer in Via L2 Bitcoin ZK-Rollup

> **Note (2026-07-02):** Re-verified against via-core source. The core finding stands: Via has no
> P2P layer of its own. All inter-node communication in via-core is client-server (JSON-RPC and
> authenticated REST). The zksync-era consensus/gossip code exists in the tree but the Via main
> node does not run it, so there is no live gossip network. Stale code excerpts in earlier
> revisions of this document have been replaced with verbatim source.

## Executive Summary

The via-core codebase contains **no operational peer-to-peer networking layer** for direct
communication between distributed Via nodes. The actual communication paths are:

1. **External node to main node**: JSON-RPC (the `en` namespace plus standard Web3 namespaces), via the `MainNodeClient` abstraction.
2. **Verifier to coordinator**: an authenticated REST API (Axum) exposed by the coordinator; verifiers poll it over HTTP. Verifiers never talk to each other directly.
3. **Every node to Bitcoin**: JSON-RPC against a Bitcoin node (`via_btc_client` uses `bitcoincore_rpc`). Bitcoin's own P2P network is external to via-core.
4. **Sequencer/verifier to Celestia**: RPC against a Celestia light node. Celestia's P2P network is likewise external to via-core.

The zksync-era P2P consensus stack (`core/node/consensus/`, built on era-consensus gossip) is
inherited in the source tree, and `via_external_node` even keeps the upstream `--enable-consensus`
CLI flag, but the `via_server` main node wires no consensus component at all, so consensus-based
(gossip) syncing has no counterpart to connect to. In practice every external node syncs over
JSON-RPC.

## 1. Verifier Network Communication

The Verifier Network uses a centralized coordinator with a REST API. The coordinator serves the
API; verifier nodes are HTTP clients of it.

```rust
// via_verifier/node/via_verifier_coordinator/src/coordinator/api.rs
#[allow(clippy::too_many_arguments)]
pub async fn start_coordinator_server(
    config: ViaVerifierConfig,
    master_connection_pool: ConnectionPool<Verifier>,
    btc_client: Arc<dyn BitcoinOps>,
    withdrawal_client: WithdrawalClient,
    verifiers_pub_keys: Vec<String>,
    mut stop_receiver: watch::Receiver<bool>,
) -> anyhow::Result<()> {
    let bind_address = config.bind_addr();
    let api = RestApi::new(
        config,
        master_connection_pool,
        btc_client,
        withdrawal_client,
        verifiers_pub_keys,
    )?
    .into_router();

    let listener = tokio::net::TcpListener::bind(bind_address)
        .await
        .context("Cannot bind to the specified address")?;
    axum::serve(listener, api)
        .with_graceful_shutdown(async move {
            if stop_receiver.changed().await.is_err() {
                tracing::warn!("Stop signal sender for coordinator server was dropped without sending a signal");
            }
            tracing::info!("Stop signal received, coordinator server is shutting down");
        })
        .await
        .with_context(|| "coordinator handler server failed")?;
    tracing::info!("coordinator handler server shut down");
    Ok(())
}
```

The routes are nested under `/session` and protected by an authentication middleware:

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

So the effective endpoints are `POST /session/new`, `GET /session`, `POST /session/signature`,
`GET /session/signature`, `POST /session/nonce`, and `GET /session/nonce`. Verifier nodes call
these URLs by building them from `coordinator_http_url`
(`via_verifier/node/via_verifier_coordinator/src/verifier/mod.rs`, e.g.
`format!("{}/session/new", self.verifier_config.coordinator_http_url)`).

Requests are authenticated with secp256k1 ECDSA signatures over the payload, verified against the
configured verifier public keys:

```rust
// via_verifier/node/via_verifier_coordinator/src/auth.rs
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

There is no direct P2P communication between verifiers; all coordination (MuSig2 nonce and
partial-signature exchange for withdrawal signing) is mediated through the coordinator.

## 2. External Node Synchronization

External nodes fetch blocks and metadata from the main node over JSON-RPC through the
`MainNodeClient` trait, implemented for the standard L2 Web3 client (which speaks the `en`, `eth`,
and `zks` namespaces):

```rust
// core/node/node_sync/src/client.rs
/// Client abstracting connection to the main node.
#[async_trait]
pub trait MainNodeClient: 'static + Send + Sync + fmt::Debug {
    async fn fetch_system_contract_by_hash(
        &self,
        hash: H256,
    ) -> EnrichedClientResult<Option<Vec<u8>>>;

    async fn fetch_genesis_contract_bytecode(
        &self,
        address: Address,
    ) -> EnrichedClientResult<Option<Vec<u8>>>;

    async fn fetch_protocol_version(
        &self,
        protocol_version: ProtocolVersionId,
    ) -> EnrichedClientResult<Option<api::ProtocolVersion>>;

    async fn fetch_l2_block_number(&self) -> EnrichedClientResult<L2BlockNumber>;

    async fn fetch_l2_block(
        &self,
        number: L2BlockNumber,
        with_transactions: bool,
    ) -> EnrichedClientResult<Option<en::SyncBlock>>;

    async fn fetch_genesis_config(&self) -> EnrichedClientResult<GenesisConfig>;
}
```

`via_external_node` builds this client in `core/bin/via_external_node/src/main.rs` from
`config.required.main_node_url` (`Client::http(main_node_url.clone())`).

## 3. Consensus (the inherited zksync-era gossip stack) is present but not live

The `core/node/consensus/` crate is zksync-era's consensus component, which does implement P2P
gossip networking (via the era-consensus executor). Its wiring status in Via:

- **Main node (`via_server`)**: no consensus component is wired. `grep -i consensus` over `core/bin/via_server/src/node_builder.rs` and `main.rs` finds only `consensus: None` in the default secrets construction (`core/bin/via_server/src/main.rs`). The main node never runs a consensus executor, so there is nothing for a gossip peer to connect to.
- **External node (`via_external_node`)**: the upstream `--enable-consensus` CLI flag is kept, documented in the source as "Enables consensus-based syncing instead of JSON-RPC based one. This is an experimental and incomplete feature" (`core/bin/via_external_node/src/main.rs`). Without the flag, the consensus config is dropped:

```rust
// core/bin/via_external_node/src/main.rs
    if !opt.enable_consensus {
        config.consensus = None;
    }
```

The `Core` component always adds `ExternalNodeConsensusLayer`
(`core/bin/via_external_node/src/node_builder.rs`, `add_consensus_layer`), but with
`config.consensus == None` that layer runs only the JSON-RPC fetcher task, which logs the upstream
deprecation warning:

```rust
// core/node/consensus/src/en.rs
    /// Task fetching L2 blocks using JSON-RPC endpoint of the main node.
    pub async fn run_fetcher(
        self,
        ctx: &ctx::Ctx,
        actions: ActionQueueSender,
    ) -> anyhow::Result<()> {
        tracing::warn!("\
            WARNING: this node is using ZKsync API synchronization, which will be deprecated soon. \
            Please follow this instruction to switch to p2p synchronization: \
            https://github.com/matter-labs/zksync-era/blob/main/docs/guides/external-node/10_decentralization.md");
```

Note: this warning lives in `core/node/consensus/src/en.rs`, not in `core/node/node_sync/src/`
(there is no `en.rs` in `node_sync`), and it is upstream zksync-era text; the linked instructions
concern zksync-era's decentralization, not a Via feature. Since the Via main node runs no
consensus executor, enabling `--enable-consensus` on a Via external node has no working gossip
counterpart. All Via external nodes sync over JSON-RPC.

## 4. Data Availability (Celestia)

The DA client talks to a Celestia light node over its RPC API. Celestia itself is a P2P network,
but via-core only holds an RPC client to one node:

```rust
// core/lib/via_da_clients/src/celestia/client.rs
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
```

The `p2p_info()` call is a health check against the light node's RPC (`celestia_rpc::P2PClient`);
it does not make via-core a Celestia peer.

## 5. Prover System

The prover gateway communicates with the core over plain HTTP (reqwest), not P2P:

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
```

## 6. Bitcoin

`via_btc_client` (`core/lib/via_btc_client/src/client/rpc_client.rs`) uses the `bitcoincore_rpc`
crate to talk to a Bitcoin node over JSON-RPC. The Bitcoin P2P network exists, but it is the
Bitcoin node's concern, entirely outside via-core.

## Conclusion

Via has no P2P networking layer of its own:

1. **Client-server architecture**: external nodes sync from the main node over JSON-RPC; verifiers poll the coordinator's authenticated REST API; provers poll the prover API over HTTP.
2. **Centralized coordination**: the verifier coordinator is a central point for the MuSig2 signing sessions.
3. **External P2P networks**: Bitcoin and Celestia are P2P networks, but via-core only holds RPC clients to single nodes of each.
4. **Inherited but dormant gossip stack**: zksync-era's consensus P2P code ships in the tree and the external node keeps the experimental `--enable-consensus` flag, but the Via main node wires no consensus component, so no gossip network runs.
