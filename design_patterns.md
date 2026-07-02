# Via L2 Bitcoin ZK-Rollup: Design Patterns

This document identifies the major design patterns actually used in the via-core codebase. Every pattern below is anchored to real code: the trait or struct definition is shown verbatim together with at least one real implementation, with file paths in comment headers.

## 1. Dependency Injection: the Node Framework Wiring Layer System

The most load-bearing structural pattern in via-core is the node framework's dependency injection system. Every long-running binary (`via_server`, `verifier_server`, `indexer`, `via_external_node`) is assembled from `WiringLayer` implementations registered on a `ZkStackServiceBuilder` (defined in `core/node/node_framework/src/service/mod.rs`, which exposes `add_layer<T: WiringLayer>`). Each layer declares typed inputs (resources it consumes) and outputs (resources and tasks it produces), and the framework resolves them at startup.

The core trait:

```rust
// core/node/node_framework/src/wiring_layer.rs
/// Wiring layer provides a way to customize the `ZkStackService` by
/// adding new tasks or resources to it.
///
/// Structures that implement this trait are advised to specify in doc comments
/// which resources they use or add, and the list of tasks they add.
#[async_trait::async_trait]
pub trait WiringLayer: 'static + Send + Sync {
    type Input: FromContext;
    type Output: IntoContext;

    /// Identifier of the wiring layer.
    fn layer_name(&self) -> &'static str;

    /// Performs the wiring process, e.g. adds tasks and resources to the node.
    /// This method will be called once during the node initialization.
    async fn wire(self, input: Self::Input) -> Result<Self::Output, WiringError>;
}
```

The framework erases the layer's associated types by capturing them in a closure, so the rest of the service only deals with `WireFn`:

```rust
// core/node/node_framework/src/wiring_layer.rs
pub(crate) trait WiringLayerExt: WiringLayer {
    /// Hires the actual type of the wiring layer into the closure, so that rest of application
    /// doesn't have to know it.
    fn into_wire_fn(self) -> WireFn
    where
        Self: Sized,
    {
        WireFn(Box::new(move |rt, ctx| {
            let input = Self::Input::from_context(ctx)?;
            let output = rt.block_on(self.wire(input))?;
            output.into_context(ctx)?;
            Ok(())
        }))
    }
}
```

A real Via-specific layer, wiring the Bitcoin watcher into the sequencer node:

```rust
// core/node/node_framework/src/implementations/layers/via_btc_watch.rs
/// Wiring layer for bitcoin watcher
///
/// Responsible for initializing and running of [`BtcWatch`] component, that polls the Bitcoin node for the relevant events.
#[derive(Debug)]
pub struct BtcWatchLayer {
    pub via_bridge_config: ViaBridgeConfig,
    pub via_btc_watch_config: ViaBtcWatchConfig,
    pub is_main_node: bool,
}

#[derive(Debug, FromContext)]
#[context(crate = crate)]
pub struct Input {
    pub master_pool: PoolResource<MasterPool>,
    pub btc_client_resource: BtcClientResource,
}

#[derive(Debug, IntoContext)]
#[context(crate = crate)]
pub struct Output {
    pub system_wallets_resource: ViaSystemWalletsResource,
    pub btc_indexer_resource: BtcIndexerResource,
    #[context(task)]
    pub btc_watch: BtcWatch,
}

#[async_trait::async_trait]
impl WiringLayer for BtcWatchLayer {
    type Input = Input;
    type Output = Output;

    fn layer_name(&self) -> &'static str {
        "via_btc_watch_layer"
    }

    async fn wire(self, input: Self::Input) -> Result<Self::Output, WiringError> {
        let main_pool = input.master_pool.get().await?;
        let client = input.btc_client_resource.default;

        let system_wallets = ViaWalletsInitializer::load_system_wallets(main_pool.clone()).await?;
        let system_wallets_resource = ViaSystemWalletsResource::from(system_wallets);

        let indexer =
            BitcoinInscriptionIndexer::new(client.clone(), system_wallets_resource.0.clone());
        let btc_indexer_resource = BtcIndexerResource::from(indexer.clone());

        let btc_watch = BtcWatch::new(
            self.via_btc_watch_config,
            indexer,
            client,
            main_pool,
            self.is_main_node,
        )
        .await?;

        Ok(Output {
            system_wallets_resource,
            btc_indexer_resource,
            btc_watch,
        })
    }
}
```

## 2. Builder Pattern: Node Assembly per Component

`ViaNodeBuilder` (and its siblings in `via_verifier/bin/verifier_server/src/node_builder.rs` and `via_indexer/bin/indexer/src/node_builder.rs`) uses a fluent builder over the wiring layers, adding layers conditionally based on which components the operator enabled:

```rust
// core/bin/via_server/src/node_builder.rs
pub fn build(mut self, mut components: Vec<ViaComponent>) -> anyhow::Result<ZkStackService> {
    self = self
        .add_pools_layer()?
        .add_sigint_handler_layer()?
        .add_object_store_layer()?
        .add_healthcheck_layer()?
        .add_circuit_breaker_checker_layer()?
        .add_postgres_layer()?
        .add_query_eth_client_layer()?
        .add_prometheus_exporter_layer()?
        .add_storage_initialization_layer(LayerKind::Precondition)?;

    // Sort the components, so that the components they may depend on each other are added in the correct order.
    components.sort_unstable_by_key(|component| match component {
        // API consumes the resources provided by other layers (multiple ones), so it has to come the last.
        ViaComponent::HttpApi | ViaComponent::WsApi => 1,
        // Default priority.
        _ => 0,
    });

    // Add "component-specific" layers.
    // Note that the layers are added only once, so it's fine to add the same layer multiple times.
    for component in &components {
        match component {
            ViaComponent::StateKeeper => {
                // State keeper is the core component of the sequencer,
                // which is why we consider it to be responsible for the storage initialization.
                self = self
                    .add_gas_adjuster_layer()?
                    .add_l1_gas_layer()?
                    .add_storage_initialization_layer(LayerKind::Task)?
                    .add_state_keeper_layer()?
                    .add_logs_bloom_backfill_layer()?;
            }
            // ... HttpApi, WsApi, Tree, TreeApi, Housekeeper, ProofDataHandler,
            //     CommitmentGenerator, DADispatcher, VmRunner*, Btc components elided ...
            _ => {}
        }
    }
    // ...
}
```

A second, lower-level builder assembles Bitcoin bridge transactions:

```rust
// via_verifier/lib/via_musig2/src/transaction_builder.rs
#[derive(Debug, Clone)]
pub struct TransactionBuilder {
    pub utxo_manager: UtxoManager,
}

impl TransactionBuilder {
    #[instrument(skip(btc_client), target = "bitcoin_transaction_builder")]
    pub fn new(btc_client: Arc<dyn BitcoinOps>) -> Result<Self> {
        let utxo_manager = UtxoManager::new(btc_client, Amount::from_sat(1000), 128);
        Ok(Self { utxo_manager })
    }

    /// Builds transactions with OP_RETURN data from the provided outputs
    pub async fn build_transaction_with_op_return(
        &self,
        outputs: Vec<TransactionOutput>,
        config: TransactionBuilderConfig,
    ) -> Result<Vec<UnsignedBridgeTx>> {
        self.utxo_manager.sync_context_with_blockchain().await?;

        let available_utxos = self.get_available_utxos(&config).await?;
        let fee_rate = self.get_fee_rate(&config).await?;

        self.build_bridge_txs(available_utxos, outputs, config, fee_rate)
            .await
    }
    // ...
}
```

## 3. Strategy Pattern: Trait Objects Selected at Construction Time

### 3.1 Inscription Message Processors

The Bitcoin watcher parses inscription messages from L1 and dispatches them to a list of processors, each handling one message family. The trait exists in three variants, one per node role: `core/node/via_btc_watch/src/message_processors/mod.rs` (sequencer), `via_verifier/node/via_btc_watch/src/message_processors/mod.rs` (verifier), and `via_indexer/node/indexer/src/message_processors/mod.rs` (indexer).

```rust
// core/node/via_btc_watch/src/message_processors/mod.rs
#[async_trait::async_trait]
pub(super) trait MessageProcessor: 'static + std::fmt::Debug + Send + Sync {
    async fn process_messages(
        &mut self,
        storage: &mut Connection<'_, Core>,
        msgs: Vec<FullInscriptionMessage>,
        indexer: &mut BitcoinInscriptionIndexer,
    ) -> Result<Option<u32>, MessageProcessorError>;
}
```

Sequencer-side implementations are `L1ToL2MessageProcessor`, `VotableMessageProcessor`, `SystemWalletProcessor`, and `GovernanceUpgradesEventProcessor`. One real implementation:

```rust
// core/node/via_btc_watch/src/message_processors/l1_to_l2.rs
#[async_trait::async_trait]
impl MessageProcessor for L1ToL2MessageProcessor {
    async fn process_messages(
        &mut self,
        storage: &mut Connection<'_, Core>,
        msgs: Vec<FullInscriptionMessage>,
        _: &mut BitcoinInscriptionIndexer,
    ) -> Result<Option<u32>, MessageProcessorError> {
        let mut priority_ops = Vec::new();
        for msg in msgs {
            if let FullInscriptionMessage::L1ToL2Message(l1_to_l2_msg) = msg {
                let mut tx_id_bytes = l1_to_l2_msg.common.tx_id.as_raw_hash()[..].to_vec();
                tx_id_bytes.reverse();
                let tx_id = H256::from_slice(&tx_id_bytes);

                if storage
                    .via_transactions_dal()
                    .transaction_exists_with_txid(&tx_id)
                    .await
                    .map_err(|e| MessageProcessorError::DatabaseError(e.to_string()))?
                {
                    tracing::info!(
                        "Transaction with tx_id {} already processed, skipping",
                        tx_id
                    );
                    continue;
                }
                let Some(l1_tx) = self.create_l1_tx_from_message(&l1_to_l2_msg)? else {
                    tracing::warn!("Invalid deposit, l1 tx_id {}", &l1_to_l2_msg.common.tx_id);
                    continue;
                };
                // ...
            }
        }
        // ...
    }
}
```

The strategies are composed at construction time, and the set depends on the node's role:

```rust
// core/node/via_btc_watch/src/lib.rs
let system_wallet_processor = Box::new(SystemWalletProcessor::new(btc_client.clone()));

// Only build message processors that match the actor role:
let mut message_processors: Vec<Box<dyn MessageProcessor>> = vec![
    Box::new(L1ToL2MessageProcessor::default()),
    Box::new(VotableMessageProcessor::default()),
];

if is_main_node {
    // ... loads protocol_semantic_version, then:
    // message_processors.push(Box::new(GovernanceUpgradesEventProcessor::new(btc_client, ...)));
}
```

### 3.2 Fee Strategies

Fee estimation for bridge withdrawals is a strategy trait with a default `estimate_fee` and a per-strategy `apply_fee_to_outputs`:

```rust
// via_verifier/lib/via_musig2/src/fee.rs
pub trait FeeStrategy: Send + Sync {
    fn estimate_fee(
        &self,
        input_count: u32,
        output_count: u32,
        fee_rate: u64,
    ) -> anyhow::Result<Amount> {
        let input_size =
            (WITNESS_OVERHEAD + INPUT_BASE_SIZE + INPUT_WITNESS_SIZE) * u64::from(input_count);
        // approximate size per output +2 (+1 for potential change)
        let output_size = OUTPUT_SIZE * u64::from(output_count + 1);

        let total_size = TX_OVERHEAD + input_size + output_size + OP_RETURN_SIZE;
        let fee = fee_rate * total_size;

        // Ensure fee is divisible by output_count to avoid decimals when splitting
        let output_count_u64 = std::cmp::max(output_count, 1) as u64;
        let remainder = fee % output_count_u64;
        let adjusted_fee = if remainder == 0 {
            fee
        } else {
            fee + (output_count_u64 - remainder)
        };

        Ok(Amount::from_sat(adjusted_fee))
    }

    fn apply_fee_to_outputs(
        &self,
        outputs: Vec<TransactionOutput>,
        input_count: u32,
        fee_rate: u64,
    ) -> anyhow::Result<TransactionWithFee>;
}

pub struct WithdrawalFeeStrategy {}
```

`WithdrawalFeeStrategy` implements `apply_fee_to_outputs` by iteratively splitting the fee across outputs and dropping outputs whose value cannot cover their fee share (same file, `impl FeeStrategy for WithdrawalFeeStrategy`).

### 3.3 Coordinator Signing Sessions

The verifier coordinator abstracts each kind of MuSig2 signing session behind `ISession`:

```rust
// via_verifier/node/via_verifier_coordinator/src/traits.rs
#[async_trait]
pub trait ISession: Any + Send + Sync {
    async fn prepare_session(&self) -> anyhow::Result<()>;

    async fn session(&self) -> anyhow::Result<Option<SessionOperation>>;

    async fn is_session_in_progress(
        &self,
        session_op_pts: &SessionOperation,
    ) -> anyhow::Result<bool>;

    async fn verify_message(&self, session_op: &SessionOperation) -> anyhow::Result<bool>;

    async fn before_process_session(&self, session_op: &SessionOperation) -> anyhow::Result<bool>;

    async fn before_broadcast_final_transaction(
        &self,
        session_op: &SessionOperation,
    ) -> anyhow::Result<bool>;

    async fn after_broadcast_final_transaction(
        &self,
        txid: Txid,
        session_op: &SessionOperation,
    ) -> anyhow::Result<bool>;

    async fn is_bridge_session_already_processed(
        &self,
        session_op: &SessionOperation,
    ) -> anyhow::Result<bool>;

    fn as_any(&self) -> &dyn Any;
}
```

The concrete implementation is `WithdrawalSession`:

```rust
// via_verifier/node/via_verifier_coordinator/src/sessions/withdrawal.rs
#[async_trait]
impl ISession for WithdrawalSession {
    async fn prepare_session(&self) -> anyhow::Result<()> {
        self.prepare_withdrawal_session().await?;
        Ok(())
    }

    async fn session(&self) -> anyhow::Result<Option<SessionOperation>> {
        let mut storage = self
            .master_connection_pool
            .connection_tagged("verifier task")
            .await?;

        // Set the minimum amount to withdraw + fee = 660 sats.
        let min_value = 660;

        let no_processed_withdrawals = storage
            .via_withdrawal_dal()
            .list_no_processed_withdrawals(min_value, WITHDRAWAL_LIMIT)
            .await?;
        // ...
    }
    // ...
}
```

Sessions are held in a registry keyed by `SessionType`, so new session kinds can be added without changing dispatch code:

```rust
// via_verifier/node/via_verifier_coordinator/src/sessions/session_manager.rs
pub struct SessionManager {
    pub sessions: HashMap<SessionType, Arc<dyn ISession>>,
}

impl SessionManager {
    pub fn new(sessions: HashMap<SessionType, Arc<dyn ISession>>) -> Self {
        Self { sessions }
    }

    pub async fn prepare_session(&self) -> anyhow::Result<()> {
        for session in self.sessions.values() {
            session.prepare_session().await?;
        }
        Ok(())
    }

    pub async fn get_next_session(&self) -> anyhow::Result<Option<SessionOperation>> {
        for session in self.sessions.values() {
            if let Some(op) = session.session().await? {
                return Ok(Some(op));
            }
        }
        Ok(None)
    }
    // ...
}
```

## 4. Repository Pattern: the DAL

All Postgres access goes through data access layer (DAL) structs that borrow a `Connection`. The sequencer's DAL is gated behind a sealed extension trait on `Connection<'a, Core>`, so consumers write `storage.via_votes_dal().insert_vote(...)`:

```rust
// core/lib/dal/src/lib.rs
mod private {
    pub trait Sealed {}
}

// Here we are making the trait sealed, because it should be public to function correctly, but we don't
// want to allow any other downstream implementations of this trait.
pub trait CoreDal<'a>: private::Sealed
where
    Self: 'a,
{
    fn transactions_dal(&mut self) -> TransactionsDal<'_, 'a>;

    fn via_transactions_dal(&mut self) -> ViaTransactionsDal<'_, 'a>;

    fn via_transaction_web3_dal(&mut self) -> ViaTransactionsWeb3Dal<'_, 'a>;

    fn via_votes_dal(&mut self) -> ViaVotesDal<'_, 'a>;

    fn via_indexer_dal(&mut self) -> ViaIndexerDal<'_, 'a>;
    // ... many more accessors ...
}
```

A representative Via repository struct with an instrumented query:

```rust
// core/lib/dal/src/via_votes_dal.rs
pub struct ViaVotesDal<'c, 'a> {
    pub(crate) storage: &'c mut Connection<'a, Core>,
}

impl ViaVotesDal<'_, '_> {
    /// Inserts a new vote for an l1 batch.
    pub async fn insert_vote(
        &mut self,
        l1_batch_number: u32,
        proof_reveal_tx_id: &[u8],
        verifier_address: &str,
        vote: bool,
    ) -> DalResult<()> {
        sqlx::query!(
            r#"
            INSERT INTO
            via_votes (l1_batch_number, proof_reveal_tx_id, verifier_address, vote)
            VALUES
            ($1, $2, $3, $4)
            ON CONFLICT (l1_batch_number, proof_reveal_tx_id, verifier_address) DO NOTHING
            "#,
            l1_batch_number as i32,
            proof_reveal_tx_id,
            verifier_address,
            vote
        )
        .instrument("insert_vote")
        .report_latency()
        .fetch_optional(self.storage)
        .await?;

        Ok(())
    }
    // ...
}
```

The verifier has its own parallel DAL crate at `via_verifier/lib/verifier_dal/` (including its own `via_votes_dal.rs`), keeping the two databases decoupled.

## 5. Abstract Factory: Batch Executors

The state keeper does not construct VM executors directly; it receives a factory trait object, which lets tests substitute mock executors (`core/node/via_state_keeper/src/testonly/test_batch_executor.rs` implements the same trait with `TestBatchExecutorBuilder`):

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
```

The production implementation spawns the VM on a blocking thread and talks to it over a channel:

```rust
// core/lib/vm_executor/src/batch/factory.rs
impl<S: ReadStorage + Send + 'static, Tr: BatchTracer> BatchExecutorFactory<S>
    for MainBatchExecutorFactory<Tr>
{
    fn init_batch(
        &mut self,
        storage: S,
        l1_batch_params: L1BatchEnv,
        system_env: SystemEnv,
        pubdata_params: PubdataParams,
    ) -> Box<dyn BatchExecutor<S>> {
        // Since we process `BatchExecutor` commands one-by-one (the next command is never enqueued
        // until a previous command is processed), capacity 1 is enough for the commands channel.
        let (commands_sender, commands_receiver) = mpsc::channel(1);
        let executor = CommandReceiver {
            optional_bytecode_compression: self.optional_bytecode_compression,
            fast_vm_mode: self.fast_vm_mode,
            observe_storage_metrics: self.observe_storage_metrics,
            divergence_handler: self.divergence_handler.clone(),
            commands: commands_receiver,
            _storage: PhantomData,
            _tracer: PhantomData::<Tr>,
        };
        // ...
    }
}
```

## 6. Client Abstraction and Fallback Decorator

### 6.1 Bitcoin RPC Client

All Bitcoin node interaction goes through the `BitcoinRpc` trait, implemented by `BitcoinRpcClient` in `core/lib/via_btc_client/src/client/rpc_client.rs`:

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

The same file also defines `BitcoinOps` (higher-level operations) and `BitcoinSigner` traits.

### 6.2 Data Availability Client with Fallback

DA is abstracted behind a boxed trait object:

```rust
// core/lib/da_client/src/lib.rs
/// Trait that defines the interface for the data availability layer clients.
#[async_trait]
pub trait DataAvailabilityClient: Sync + Send + fmt::Debug {
    /// Dispatches a blob to the data availability layer.
    async fn dispatch_blob(
        &self,
        batch_number: u32,
        data: Vec<u8>,
    ) -> Result<DispatchResponse, DAError>;

    /// Fetches the inclusion data for a given blob_id.
    async fn get_inclusion_data(&self, blob_id: &str) -> Result<Option<InclusionData>, DAError>;

    /// Clones the client and wraps it in a Box.
    fn clone_boxed(&self) -> Box<dyn DataAvailabilityClient>;

    /// Returns the maximum size of the blob (in bytes) that can be dispatched. None means no limit.
    fn blob_size_limit(&self) -> Option<usize>;
}

impl Clone for Box<dyn DataAvailabilityClient> {
    fn clone(&self) -> Box<dyn DataAvailabilityClient> {
        self.clone_boxed()
    }
}
```

Via ships four implementations in `core/lib/via_da_clients/src/`: `celestia/client.rs` (`CelestiaClient`), `external_node/client.rs`, `http/client.rs`, and a fallback decorator that composes two other clients. The fallback client is a decorator: it implements the same trait it wraps, adding failover and optional cross-source consistency checking:

```rust
// core/lib/via_da_clients/src/fallback/client.rs
/// A fallback DA client that tries the primary client first (typically Celestia),
/// and falls back to a secondary client (typically External Node) if the primary fails.
/// This is useful for handling scenarios where Celestia data expires after 30 days.
#[derive(Clone)]
pub struct FallbackDaClient {
    primary: Box<dyn DataAvailabilityClient>,
    fallback: Option<Box<dyn DataAvailabilityClient>>,
    /// If true, verifies that data from fallback matches data from primary when both are available
    verify_consistency: bool,
}

#[async_trait]
impl DataAvailabilityClient for FallbackDaClient {
    /// Dispatches blob to the primary client only
    async fn dispatch_blob(
        &self,
        batch_number: u32,
        data: Vec<u8>,
    ) -> Result<DispatchResponse, DAError> {
        self.primary.dispatch_blob(batch_number, data).await
    }

    /// Fetches inclusion data, trying primary first, then fallback if primary fails
    async fn get_inclusion_data(&self, blob_id: &str) -> Result<Option<InclusionData>, DAError> {
        // Try primary client first
        match self.primary.get_inclusion_data(blob_id).await {
            Ok(Some(primary_data)) => {
                // If verification is enabled, also fetch from fallback and verify consistency
                // ... (fetches from fallback, calls verify_data_consistency, logs mismatches) ...
            }
            // ... Ok(None) and Err arms fall through to the fallback client ...
        }
    }
    // ...
}
```

## 7. Polling Watcher Loop (Observer over Bitcoin L1)

Via has no push-based event bus for L1 events. Instead, `BtcWatch` (and the analogous watchers in `via_verifier/node/via_btc_watch` and `via_indexer/node/indexer`) run a poll-tick-process loop driven by a `tokio` interval and a stop signal `watch::Receiver`:

```rust
// core/node/via_btc_watch/src/lib.rs
pub async fn run(mut self, mut stop_receiver: watch::Receiver<bool>) -> anyhow::Result<()> {
    let mut timer = tokio::time::interval(self.btc_watch_config.poll_interval());
    let pool = self.pool.clone();

    while !*stop_receiver.borrow_and_update() {
        tokio::select! {
            _ = timer.tick() => { /* continue iterations */ }
            _ = stop_receiver.changed() => break,
        }

        let mut storage = pool.connection_tagged(BtcWatch::module_name()).await?;
        match self.loop_iteration(&mut storage).await {
            Ok(()) => { /* everything went fine */ }
            Err(err) => {
                METRICS.errors.inc();
                tracing::error!("Failed to process new blocks: {err}");
            }
        }
    }

    tracing::info!("Stop signal received, via_btc_watch is shutting down");
    Ok(())
}
```

Each `loop_iteration` reads the last processed L1 block from `via_indexer_dal()`, applies a confirmation-depth cutoff (`current_l1_block_number` is the chain tip minus `block_confirmations`), fetches inscriptions in chunks of `L1_BLOCKS_CHUNK` blocks, and hands them to the `MessageProcessor` strategies from section 3.1. Progress is persisted in Postgres, so processing is resumable and idempotent (processors check for already-seen txids before inserting).

## 8. Versioned Modules for Protocol Evolution

### 8.1 Versioned Proof Verification

The verifier must be able to verify proofs produced under different protocol versions. `via_verification` isolates each version in its own module and dispatches through an enum:

```rust
// via_verifier/lib/via_verification/src/lib.rs
pub mod version_27;
pub mod version_28;

#[derive(Clone)]
pub enum ProveBatchData {
    V27(ProveBatchesV27),
    V28(ProveBatchesV28),
}

/// Decodes the proof data into the appropriate version. It's possible that the data serialization format changes even within the same version.
pub fn decode_prove_batch_data(
    protocol_version_id: ProtocolVersionId,
    proof_data: &[u8],
) -> anyhow::Result<ProveBatchData> {
    if protocol_version_id <= ProtocolVersionId::Version27 {
        if let Ok(prove_batch) = bincode::deserialize::<ProveBatchesV27>(proof_data) {
            tracing::info!("Decode proof data with V27");
            return Ok(ProveBatchData::V27(prove_batch));
        }
        tracing::warn!("Failed to decode proof data as V27");
    }

    if let Ok(prove_batch) = bincode::deserialize::<ProveBatchesV28>(proof_data) {
        tracing::info!("Decode proof data with V28");
        return Ok(ProveBatchData::V28(prove_batch));
    }

    // If both fail, return an error
    anyhow::bail!("Failed to decode proof data as either V27 or V28");
}

pub async fn verify_proof(prover_batch_data: ProveBatchData) -> anyhow::Result<bool> {
    match prover_batch_data {
        ProveBatchData::V27(data) => verify_proof_v27(data).await,
        ProveBatchData::V28(data) => verify_proof_v28(data).await,
    }
}
```

Each `version_XX` module carries its own `proof.rs`, `verification.rs`, `l1_data_fetcher.rs`, and `public_inputs.rs`, so an old version's verification path is never mutated by new-version work.

### 8.2 Versioned VMs (multivm)

The same idea at larger scale: `core/lib/multivm/src/versions/` contains a directory per historical VM (`vm_1_3_2`, `vm_1_4_1`, `vm_1_4_2`, `vm_boojum_integration`, `vm_fast`, `vm_latest`, plus `shadow` for divergence testing), allowing any historical batch to be re-executed under the exact VM semantics it was originally sealed with.

## 9. Fault Tolerance: Circuit Breakers and Retries

### 9.1 Circuit Breaker

Via wires the `CircuitBreakerCheckerLayer` into every server build (`add_circuit_breaker_checker_layer()` in `core/bin/via_server/src/node_builder.rs`, see section 2). The mechanism is a registry of named checks polled on an interval; if any check fails, the service is signaled to stop. Shipped breakers are `FailedL1TransactionChecker` (`l1_txs.rs`) and `ReplicationLagChecker` (`replication_lag.rs`).

```rust
// core/lib/circuit_breaker/src/lib.rs
#[derive(Default, Debug)]
pub struct CircuitBreakers(Mutex<Vec<Box<dyn CircuitBreaker>>>);

impl CircuitBreakers {
    pub async fn insert(&self, circuit_breaker: Box<dyn CircuitBreaker>) {
        let mut guard = self.0.lock().await;
        if !guard
            .iter()
            .any(|existing_breaker| existing_breaker.name() == circuit_breaker.name())
        {
            guard.push(circuit_breaker);
        }
    }

    pub async fn check(&self) -> Result<(), CircuitBreakerError> {
        for circuit_breaker in self.0.lock().await.iter() {
            circuit_breaker.check().await?;
        }
        Ok(())
    }
}

#[async_trait::async_trait]
pub trait CircuitBreaker: fmt::Debug + Send + Sync {
    fn name(&self) -> &'static str;

    async fn check(&self) -> Result<(), CircuitBreakerError>;
}
```

Note this is a fail-stop breaker (unhealthy check halts the node), not the classic open/half-open/closed request breaker; there is no half-open probing state in this implementation.

### 9.2 Retry with Fixed Delay

Bitcoin RPC calls are wrapped in a bounded retry helper. The delay is fixed per attempt, not exponential:

```rust
// core/lib/via_btc_client/src/utils.rs
pub(crate) async fn with_retry<F, T, E>(
    f: F,
    max_retries: u8,
    retry_delay_ms: u64,
    operation_name: &str,
) -> Result<T, E>
where
    F: Fn() -> Result<T, E> + Send + Sync,
    E: std::fmt::Debug,
{
    let mut retries = 0;
    loop {
        match f() {
            Ok(result) => return Ok(result),
            Err(e) if retries < max_retries => {
                tracing::warn!(
                    error = ?e,
                    retries,
                    "{} failed, retrying",
                    operation_name
                );
                retries += 1;
                tokio::time::sleep(Duration::from_millis(retry_delay_ms)).await;
            }
            Err(e) => {
                METRICS.rpc_max_retries_exceeded[&RpcMethodLabel {
                    method: operation_name.into(),
                }]
                    .inc();
                return Err(e);
            }
        }
    }
}
```

```rust
// core/lib/via_btc_client/src/client/rpc_client.rs
async fn retry_rpc<F, T>(f: F) -> BitcoinRpcResult<T>
where
    F: Fn() -> BitcoinRpcResult<T> + Send + Sync,
{
    with_retry(f, RPC_MAX_RETRIES, RPC_RETRY_DELAY_MS, "RPC call").await
}
```

## 10. Multi-Round Protocol State Machine: MuSig2 Signer

Bridge withdrawals are signed with MuSig2 (two rounds: nonce exchange, then partial signatures). The `Signer` object encodes the round progression in its fields and enforces ordering at runtime:

```rust
// via_verifier/lib/via_musig2/src/lib.rs
pub struct Signer {
    secret_key: SecretKey,
    public_key: PublicKey,
    signer_index: usize,
    key_agg_ctx: KeyAggContext,
    first_round: Option<FirstRound>,
    second_round: Option<SecondRound<Vec<u8>>>,
    message: Vec<u8>,
    nonce_submitted: bool,
    partial_sig_submitted: bool,
}
```

```rust
// via_verifier/lib/via_musig2/src/lib.rs
pub fn start_signing_session(&mut self, message: Vec<u8>) -> Result<PubNonce, MusigError> {
    self.message = message.clone();

    let msg_array = message.as_slice();

    let first_round = FirstRound::new(
        self.key_agg_ctx.clone(),
        OsRng.gen::<[u8; 32]>(),
        self.signer_index,
        SecNonceSpices::new()
            .with_seckey(self.secret_key)
            .with_message(&msg_array),
    )
    .map_err(|e| MusigError::Musig2Error(e.to_string()))?;

    let nonce = first_round.our_public_nonce();
    self.first_round = Some(first_round);
    Ok(nonce)
}
```

The remaining transitions are `receive_nonce(signer_index, nonce)`, `create_partial_signature()`, and `create_final_signature()` (same file). The coordinator drives this over HTTP: verifiers poll the coordinator, submit nonces, then partial signatures, and the coordinator aggregates and broadcasts the final Taproot transaction using the `SessionManager` from section 3.3.

## 11. Cross-Cutting Notes

- **State commitment**: the state tree is the zksync-era Merkle tree, a binary tree implemented with AR16MT (amortized radix-16 Merkle tree, from the Jellyfish Merkle tree paper) as documented in `core/lib/merkle_tree/src/lib.rs` and `core/lib/merkle_tree/README.md`, versioned by L1 batch number.
- **Role-specialized copies over shared abstractions**: the sequencer, verifier, and indexer each carry their own `MessageProcessor` trait, watcher crate, and (for the verifier) DAL crate, rather than sharing one parameterized implementation. This trades duplication for independent evolution of the three node roles.
- **Testability through the trait seams**: the factory (`BatchExecutorFactory`), client (`BitcoinRpc`, `DataAvailabilityClient`), and strategy (`MessageProcessor`) traits are exactly the seams the test suites use, e.g. `MockBatchExecutor` in `core/node/via_state_keeper/src/testonly/mod.rs`.

## 12. Conclusion

The patterns in via-core are pragmatic Rust service patterns rather than textbook enterprise patterns: typed dependency injection through `WiringLayer`, trait-object strategies chosen per node role, repository-style DALs sealed behind `CoreDal`, decorator-based DA failover, poll-based L1 watching with persisted cursors, and strict module-per-version isolation for anything consensus-critical (VM semantics, proof verification). The consistent theme is that every external dependency (Bitcoin node, Celestia, Postgres, the VM) sits behind a trait, and every consensus-relevant behavior is versioned rather than mutated.
