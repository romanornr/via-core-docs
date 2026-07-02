# Via External Node

The Via External Node (binary `via_external_node`) is a read replica of the Via main node (`via_server`). It re-executes L2 blocks fetched from the main node over JSON-RPC, serves its own HTTP/WS JSON-RPC, proxies transaction submission to the main node, and additionally runs the Via Bitcoin stack: a BTC client, the BTC watch in non-main-node (follower) mode, a Bitcoin-based reorg detector, and the Via consistency checker.

Source of truth:
- Binary crate: `core/bin/via_external_node/` (`main.rs`, `node_builder.rs`, `config/mod.rs`, `config/observability.rs`, `metrics/framework.rs`, `metadata.rs`, plus a `README.md` inherited from the upstream zksync-era external node)
- The inherited zksync-era external node also still exists at `core/bin/external_node/`; the Via node is a separate crate
- Docker image: `docker/via-external-node/Dockerfile` and `docker/via-external-node/entrypoint.sh`
- CLI wrapper: `infrastructure/via/src/server.ts` (`via external-node`) and `infrastructure/via/src/setup_en.ts` (`via setup-external-node`)
- Config template: `etc/env/configs/via_ext_node.toml`

## 1. CLI surface

The full CLI, component list, and startup sequence, verbatim from source:

```rust
// core/bin/via_external_node/src/main.rs
#[derive(Debug, Clone, clap::Subcommand)]
enum Command {
    /// Generates consensus secret keys to use in the secrets file.
    /// Prints the keys to the stdout, you need to copy the relevant keys into your secrets file.
    GenerateSecrets,
}

/// External node for Via protocol.
#[derive(Debug, Parser)]
#[command(author = "Via Protocol", version, about = "Via external node", long_about = None)]
struct Cli {
    #[command(subcommand)]
    command: Option<Command>,

    /// Enables consensus-based syncing instead of JSON-RPC based one. This is an experimental and incomplete feature;
    /// do not use unless you know what you're doing.
    #[arg(long)]
    enable_consensus: bool,

    /// Comma-separated list of components to launch.
    #[arg(long, default_value = "all")]
    components: ComponentsToRun,
    /// Path to the yaml config. If set, it will be used instead of env vars.
    #[arg(
        long,
        requires = "secrets_path",
        requires = "external_node_config_path"
    )]
    config_path: Option<std::path::PathBuf>,
    /// Path to the yaml with secrets. If set, it will be used instead of env vars.
    #[arg(long, requires = "config_path", requires = "external_node_config_path")]
    secrets_path: Option<std::path::PathBuf>,
    /// Path to the yaml with external node specific configuration. If set, it will be used instead of env vars.
    #[arg(long, requires = "config_path", requires = "secrets_path")]
    external_node_config_path: Option<std::path::PathBuf>,
    /// Path to the yaml with consensus config. If set, it will be used instead of env vars.
    #[arg(
        long,
        requires = "config_path",
        requires = "secrets_path",
        requires = "external_node_config_path",
        requires = "enable_consensus"
    )]
    consensus_path: Option<std::path::PathBuf>,
}

#[derive(Debug, Clone, Copy, PartialEq, Hash, Eq)]
pub enum Component {
    HttpApi,
    WsApi,
    Tree,
    TreeApi,
    TreeFetcher,
    Core,
}

impl Component {
    fn components_from_str(s: &str) -> anyhow::Result<&[Component]> {
        match s {
            "api" => Ok(&[Component::HttpApi, Component::WsApi]),
            "http_api" => Ok(&[Component::HttpApi]),
            "ws_api" => Ok(&[Component::WsApi]),
            "tree" => Ok(&[Component::Tree]),
            "tree_api" => Ok(&[Component::TreeApi]),
            "tree_fetcher" => Ok(&[Component::TreeFetcher]),
            "core" => Ok(&[Component::Core]),
            "all" => Ok(&[
                Component::HttpApi,
                Component::WsApi,
                Component::Tree,
                Component::Core,
            ]),
            other => Err(anyhow::anyhow!("{other} is not a valid component name")),
        }
    }
}
```

Notes grounded in the code above:
- `--components` accepts `api`, `http_api`, `ws_api`, `tree`, `tree_api`, `tree_fetcher`, `core`, `all`. The default `all` expands to HttpApi, WsApi, Tree, Core (not TreeApi or TreeFetcher).
- `via_external_node generate-secrets` prints consensus secret keys (`Command::GenerateSecrets` calls `generate_consensus_secrets()`).
- Consensus syncing is opt-in via `--enable-consensus` and is marked experimental in the source itself.

The `main()` function shows the startup order. Note that it always builds a rate-limited main node client and always fetches the remote part of the config from the main node; there is no offline or "database dump" boot path:

```rust
// core/bin/via_external_node/src/main.rs
fn main() -> anyhow::Result<()> {
    let runtime = tokio_runtime()?;

    // Initial setup.
    let opt = Cli::parse();

    if let Some(cmd) = &opt.command {
        match cmd {
            Command::GenerateSecrets => generate_consensus_secrets(),
        }
        return Ok(());
    }

    let mut config = if let Some(config_path) = opt.config_path.clone() {
        let secrets_path = opt.secrets_path.clone().unwrap();
        let external_node_config_path = opt.external_node_config_path.clone().unwrap();
        if opt.enable_consensus {
            anyhow::ensure!(
                opt.consensus_path.is_some(),
                "if --config-path and --enable-consensus are specified, then --consensus-path should be used to specify the location of the consensus config"
            );
        }
        ExternalNodeConfig::from_files(
            config_path,
            external_node_config_path,
            secrets_path,
            opt.consensus_path.clone(),
        )?
    } else {
        ExternalNodeConfig::new().context("Failed to load node configuration")?
    };

    if !opt.enable_consensus {
        config.consensus = None;
    }
    let guard = {
        // Observability stack implicitly spawns several tokio tasks, so we need to call this method
        // from within tokio context.
        let _rt_guard = runtime.enter();
        config.observability.build_observability()?
    };

    // Build L1 and L2 clients.
    let main_node_url = &config.required.main_node_url;
    tracing::info!("Main node URL is: {main_node_url:?}");
    let main_node_client = Client::http(main_node_url.clone())
        .context("failed creating JSON-RPC client for main node")?
        .for_network(config.required.l2_chain_id.into())
        .with_allowed_requests_per_second(config.optional.main_node_rate_limit_rps)
        .build();
    let main_node_client = Box::new(main_node_client) as Box<DynClient<L2>>;

    let config = runtime
        .block_on(config.fetch_remote(main_node_client.as_ref()))
        .context("failed fetching remote part of node config from main node")?;

    let node = ExternalNodeBuilder::on_runtime(runtime, config)
        .build(opt.components.0.into_iter().collect())?;
    node.run(guard)?;
    anyhow::Ok(())
}
```

## 2. Node builder wiring

The builder in `core/bin/via_external_node/src/node_builder.rs` composes the node from framework layers. The `build` method is the authoritative statement of what runs where:

```rust
// core/bin/via_external_node/src/node_builder.rs
    pub fn build(mut self, mut components: Vec<Component>) -> anyhow::Result<ZkStackService> {
        // Add "base" layers
        self = self
            .add_sigint_handler_layer()?
            .add_healthcheck_layer()?
            .add_prometheus_exporter_layer()?
            .add_pools_layer()?
            .add_main_node_client_layer()?;

        // Add layers that must run only on a single component.
        if components.contains(&Component::Core) {
            // Core is a singleton & mandatory component,
            // so until we have a dedicated component for "auxiliary" tasks,
            // it's responsible for things like metrics.
            self = self
                .add_postgres_layer()?
                .add_external_node_metrics_layer()?;
            // We assign the storage initialization to the core, as it's considered to be
            // the "main" component.
            self = self
                .add_block_reverter_layer()?
                .add_storage_initialization_layer(LayerKind::Task)?;
        }

        // Add preconditions for all the components.
        self = self
            .add_btc_client_layer()?
            .add_init_node_reorg_detector_layer()?
            .add_validate_chain_ids_layer()?
            .add_storage_initialization_layer(LayerKind::Precondition)?;

        // Sort the components, so that the components they may depend on each other are added in the correct order.
        components.sort_unstable_by_key(|component| match component {
            // API consumes the resources provided by other layers (multiple ones), so it has to come the last.
            Component::HttpApi | Component::WsApi => 1,
            // Default priority.
            _ => 0,
        });

        for component in &components {
            match component {
                Component::HttpApi => {
                    self = self
                        .add_sync_state_updater_layer()?
                        .add_mempool_cache_layer()?
                        .add_tree_api_client_layer()?
                        .add_main_node_fee_params_fetcher_layer()?
                        .add_tx_sender_layer()?
                        .add_http_web3_api_layer()?;
                }
                Component::WsApi => {
                    self = self
                        .add_sync_state_updater_layer()?
                        .add_mempool_cache_layer()?
                        .add_tree_api_client_layer()?
                        .add_main_node_fee_params_fetcher_layer()?
                        .add_tx_sender_layer()?
                        .add_ws_web3_api_layer()?;
                }
                Component::Tree => {
                    // Right now, distributed mode for EN is not fully supported, e.g. there are some
                    // issues with reorg detection and snapshot recovery.
                    // So we require the core component to be present, e.g. forcing the EN to run in a monolithic mode.
                    anyhow::ensure!(
                        components.contains(&Component::Core),
                        "Tree must run on the same machine as Core"
                    );
                    let with_tree_api = components.contains(&Component::TreeApi);
                    self = self.add_metadata_calculator_layer(with_tree_api)?;
                }
                Component::TreeApi => {
                    if components.contains(&Component::Tree) {
                        // Do nothing, will be handled by the `Tree` component.
                    } else {
                        self = self.add_isolated_tree_api_layer()?;
                    }
                }
                Component::TreeFetcher => {
                    self = self.add_tree_data_fetcher_layer()?;
                }
                Component::Core => {
                    // Main tasks
                    self = self
                        .add_state_keeper_layer()?
                        .add_consensus_layer()?
                        .add_pruning_layer()?
                        .add_init_node_storage_layer()?
                        .add_btc_watcher_layer()?
                        .add_consistency_checker_layer()?
                        .add_commitment_generator_layer()?
                        .add_batch_status_updater_layer()?
                        .add_logs_bloom_backfill_layer()?;
                }
            }
        }

        Ok(self.node.build())
    }
```

Key consequences:
- The BTC client layer and the Via reorg detector layer are unconditional preconditions for every component. The reorg detector is not optional and its config is required; the doc previously claiming "no reorg detector by default" was wrong.
- The Core component runs the state keeper (external IO), consensus layer, pruning (if enabled), Via node storage initializer, BTC watch, Via consistency checker, commitment generator, batch status updater, and logs bloom backfill.
- The Merkle tree is `MerkleTreeMode::Lightweight` (see `add_metadata_calculator_layer`) and must run in the same process as Core.

The Via-specific layer methods, verbatim:

```rust
// core/bin/via_external_node/src/node_builder.rs
    fn add_btc_client_layer(mut self) -> anyhow::Result<Self> {
        let via_btc_client_config = self
            .config
            .via_btc_client_config
            .clone()
            .ok_or_else(|| anyhow!("via_btc_client_config is required"))?;
        let secrets = self
            .config
            .via_secrets
            .clone()
            .ok_or_else(|| anyhow!("VIA secrets is required"))?;

        self.node.add_layer(BtcClientLayer::new(
            via_btc_client_config,
            secrets,
            ViaWallets::default(),
            None,
        ));
        Ok(self)
    }

    fn add_init_node_storage_layer(mut self) -> anyhow::Result<Self> {
        let via_genesis_config = self
            .config
            .via_genesis_config
            .clone()
            .ok_or_else(|| anyhow!("via_genesis_config is required"))?;

        let via_btc_watch_config = self
            .config
            .via_btc_watch_config
            .clone()
            .ok_or_else(|| anyhow!("via_btc_watch_config is required"))?;

        self.node.add_layer(ViaNodeStorageInitializerLayer {
            via_genesis_config,
            via_btc_watch_config,
        });

        Ok(self)
    }

    fn add_btc_watcher_layer(mut self) -> anyhow::Result<Self> {
        let via_bridge_config = self
            .config
            .via_bridge_config
            .clone()
            .ok_or_else(|| anyhow!("via_bridge_config is required"))?;

        let via_btc_watch_config = self
            .config
            .via_btc_watch_config
            .clone()
            .ok_or_else(|| anyhow!("via_btc_watch_config is required"))?;

        self.node.add_layer(BtcWatchLayer {
            via_bridge_config,
            via_btc_watch_config,
            is_main_node: false,
        });

        Ok(self)
    }

    fn add_init_node_reorg_detector_layer(mut self) -> anyhow::Result<Self> {
        let config = self
            .config
            .via_reorg_detector_config
            .clone()
            .ok_or_else(|| anyhow!("via_reorg_detector_config is required"))?;

        self.node
            .add_layer(ViaNodeReorgDetectorLayer::new(config, false));
        Ok(self)
    }
```

Other Via layers wired by the builder:

```rust
// core/bin/via_external_node/src/node_builder.rs
    fn add_validate_chain_ids_layer(mut self) -> anyhow::Result<Self> {
        let layer = ViaValidateChainIdsLayer::new(
            self.config.remote.via_network,
            self.config.required.l2_chain_id,
        );
        self.node.add_layer(layer);
        Ok(self)
    }

    fn add_consistency_checker_layer(mut self) -> anyhow::Result<Self> {
        let max_batches_to_recheck = 10; // TODO (BFT-97): Make it a part of a proper EN config
        let layer = ViaConsistencyCheckerLayer::new(max_batches_to_recheck);
        self.node.add_layer(layer);
        Ok(self)
    }

    fn add_main_node_fee_params_fetcher_layer(mut self) -> anyhow::Result<Self> {
        self.node.add_layer(ViaMainNodeFeeParamsFetcherLayer);
        Ok(self)
    }
```

## 3. How the node syncs

Two sync planes exist side by side:

1. From the main node (inherited zksync-era mechanics): the state keeper with `ExternalIOLayer` re-executes blocks fetched from the main node, `ProxySinkLayer` forwards submitted transactions to the main node, `BatchStatusUpdaterLayer` tracks batch statuses, `SyncStateUpdaterLayer` tracks sync state, and `TreeDataFetcherLayer` (only with the `tree_fetcher` component) fetches tree data.
2. From Bitcoin (Via-specific): `BtcClientLayer` provides a Bitcoin RPC client, `BtcWatchLayer` indexes bridge inscriptions in follower mode, and `ViaNodeReorgDetectorLayer` performs Bitcoin-based reorg detection.

### 3.1 Bitcoin reorg detection

The layer wires `ViaMainNodeReorgDetector` with `is_main_node = false` on the external node:

```rust
// core/node/node_framework/src/implementations/layers/via_main_node_reorg_detector.rs
#[derive(Debug)]
pub struct ViaNodeReorgDetectorLayer {
    config: ViaReorgDetectorConfig,
    is_main_node: bool,
}

#[async_trait::async_trait]
impl WiringLayer for ViaNodeReorgDetectorLayer {
    type Input = Input;
    type Output = Output;

    fn layer_name(&self) -> &'static str {
        "Via_main_node_reorg_detector"
    }

    async fn wire(self, input: Self::Input) -> Result<Self::Output, WiringError> {
        let client = input.btc_client_resource.default;
        let pool = input.master_pool.get().await?;

        let reorg_detector =
            ViaMainNodeReorgDetector::new(self.config, pool, client.clone(), self.is_main_node);

        Ok(Output { reorg_detector })
    }
}
```

The detector keeps a local mirror of Bitcoin block hashes (`via_l1_blocks` table), compares a sliding window against the canonical chain, and classifies divergences. On a node where `is_main_node` is false (the external node), every detected reorg takes the "soft" path: it records reorg metadata, waits, rewinds the `via_btc_watch` cursor, and deletes the reorged local block rows. The hard-reorg branch (marking an affected L1 batch) only runs on the main node. Verbatim:

```rust
// core/node/via_main_node_reorg_detector/src/lib.rs
    async fn loop_iteration(&mut self, storage: &mut Connection<'_, Core>) -> anyhow::Result<()> {
        if self.check_if_reorg_in_progress(storage).await? {
            return Ok(());
        }

        self.detect_reorg(storage).await?;
        self.sync_l1_blocks(storage).await?;

        Ok(())
    }
```

```rust
// core/node/via_main_node_reorg_detector/src/lib.rs
    async fn detect_reorg(&self, storage: &mut Connection<'_, Core>) -> anyhow::Result<()> {
        let Some((block_height, _)) = storage.via_l1_block_dal().get_last_l1_block().await? else {
            // Nothing to compare until `via_l1_blocks` is bootstrapped.
            tracing::debug!("Skipping reorg check: via_l1_blocks not yet bootstrapped");
            return Ok(());
        };

        let window = self.config.reorg_window().max(1);
        let start_height = reorg_window_start(block_height, window);

        // `list_l1_blocks` only returns what exists locally - the window can be sparse.
        // We must compare by height, not by position in the result.
        let db_blocks = storage.via_l1_block_dal().list_l1_blocks(start_height, window).await?;

        // A failed canonical fetch is inconclusive; it must not trigger reorg
        // handling.
        let chain_blocks = match self.fetch_blocks(start_height, block_height).await {
            Ok(blocks) => blocks,
            Err(err) => {
                tracing::warn!(
                    "Skipping reorg check ({start_height}..={block_height}): \
                     failed to fetch canonical L1 window: {err:#}",
                );
                return Ok(());
            }
        };

        if chain_blocks.is_empty() {
            return Ok(());
        }

        let canonical_by_height: HashMap<i64, String> =
            chain_blocks.into_iter().map(|(height, block)| (height, block.block_hash().to_string())).collect();

        let reorg_start_block_height = match scan_for_reorg(&db_blocks, &canonical_by_height) {
            ReorgScan::NoReorg => return Ok(()),
            ReorgScan::ReorgAt(height) => {
                tracing::warn!("Reorg detected at block {}", height);
                height
            }
            ReorgScan::SparseAt(height) => {
                // Missing canonical data for a DB-known height is inconclusive.
                tracing::warn!(
                    "Skipping reorg check: canonical chain fetch is missing block {height} \
                     for height-keyed comparison ({start_height}..={block_height} window)"
                );
                return Ok(());
            }
        };

        // `reorg_start_block_height` is the canonical divergence point and drives
        // metrics, reorg metadata, and the affected-batches query. The
        // `via_btc_watch` cursor is only used to bound the rewind target so we
        // never move the cursor forward.
        let last_processed_l1_block = storage.via_indexer_dal().get_last_processed_l1_block("via_btc_watch").await? as i64;

        let l1_batch_number_opt = storage.via_l1_block_dal().list_l1_batches_with_priority_txs(reorg_start_block_height as i32).await?;

        if l1_batch_number_opt.is_none() || !self.is_main_node {
            tracing::info!("Soft reorg detected: no l1 batch affected or not main node");

            storage.via_l1_block_dal().insert_reorg_metadata(reorg_start_block_height, 0).await?;

            sleep(Duration::from_secs(30)).await;

            let mut transaction = storage.start_transaction().await?;

            // Cursor 0 means `via_btc_watch` is uninitialized; fall back to the
            // pre-divergence height. Otherwise cap the keep-target at the cursor
            // so a reorg above the cursor never demotes it forward.
            let l1_block_number_to_keep = if last_processed_l1_block > 0 {
                (reorg_start_block_height - 1).min(last_processed_l1_block)
            } else {
                reorg_start_block_height - 1
            };

            transaction.via_indexer_dal().update_last_processed_l1_block("via_btc_watch", l1_block_number_to_keep as u32).await?;

            transaction.via_l1_block_dal().delete_l1_reorg(l1_block_number_to_keep).await?;

            transaction.via_l1_block_dal().delete_l1_blocks(l1_block_number_to_keep).await?;

            transaction.commit().await?;

            METRICS.reorg_type[&ReorgType::Soft].set(reorg_start_block_height as usize);

            tracing::info!("Soft reorg handled successfully");

            return Ok(());
        };

        if self.is_main_node {
            let l1_batch_number = l1_batch_number_opt.unwrap();
            tracing::warn!("Hard reorg detected: affects L1 batch {} from block {}", l1_batch_number, reorg_start_block_height);

            storage.via_l1_block_dal().insert_reorg_metadata(reorg_start_block_height, l1_batch_number).await?;

            METRICS.reorg_type[&ReorgType::Hard].set(reorg_start_block_height as usize);
        }

        Ok(())
    }
```

The detector's config is small and mostly hardcoded:

```rust
// core/lib/config/src/configs/via_reorg_detector.rs
#[derive(Debug, Deserialize, Default, Serialize, Clone, PartialEq)]
pub struct ViaReorgDetectorConfig {
    /// Service interval in milliseconds.
    pub poll_interval_ms: u64,
}

impl ViaReorgDetectorConfig {
    /// Converts `self.poll_interval` into `Duration`.
    pub fn poll_interval(&self) -> Duration {
        Duration::from_millis(self.poll_interval_ms)
    }

    /// The number of blocks to process per iteration.
    pub fn block_limit(&self) -> i64 {
        50
    }

    /// The reorg window the reorg detector will use to check for reorgs.
    pub fn reorg_window(&self) -> i64 {
        100
    }

    /// Maximum number of concurrent block fetches.
    pub fn max_concurrent_fetches(&self) -> usize {
        10
    }
}
```

It is loaded from the environment with the `VIA_REORG_DETECTOR_` prefix:

```rust
// core/lib/env_config/src/via_reorg_detector.rs
impl FromEnv for ViaReorgDetectorConfig {
    fn from_env() -> anyhow::Result<Self> {
        envy_load("via_reorg_detector", "VIA_REORG_DETECTOR_")
    }
}
```

### 3.2 BTC watch in follower mode

`BtcWatchLayer` is constructed with `is_main_node: false` on the external node (see section 2). Inside the BTC watch, that flag controls which message processors are built. On a follower, deposits (`L1ToL2MessageProcessor`) and votable messages (`VotableMessageProcessor`) are still parsed, and the system wallet processor always runs, but governance upgrade processing is main-node-only:

```rust
// core/node/via_btc_watch/src/lib.rs
impl BtcWatch {
    #[allow(clippy::too_many_arguments)]
    pub async fn new(
        btc_watch_config: ViaBtcWatchConfig,
        indexer: BitcoinInscriptionIndexer,
        btc_client: Arc<BitcoinClient>,
        pool: ConnectionPool<Core>,
        is_main_node: bool,
    ) -> anyhow::Result<Self> {
        let system_wallet_processor = Box::new(SystemWalletProcessor::new(btc_client.clone()));

        // Only build message processors that match the actor role:
        let mut message_processors: Vec<Box<dyn MessageProcessor>> = vec![
            Box::new(L1ToL2MessageProcessor::default()),
            Box::new(VotableMessageProcessor::default()),
        ];

        if is_main_node {
            let mut storage = pool.connection_tagged(BtcWatch::module_name()).await?;

            let protocol_semantic_version = storage
                .protocol_versions_dal()
                .latest_semantic_version()
                .await
                .expect("Failed to load the latest protocol semantic version")
                .ok_or_else(|| anyhow::anyhow!("Protocol version is missing"))?;

            message_processors.push(Box::new(GovernanceUpgradesEventProcessor::new(
                btc_client,
                protocol_semantic_version,
            )));
        }

        Ok(Self {
            btc_watch_config,
            indexer,
            pool,
            message_processors,
            system_wallet_processor,
        })
    }
```

### 3.3 Fee parameters

Fee parameters come from the main node over the `zks` namespace (`get_fee_params`), starting from a sensible default until the first successful fetch:

```rust
// core/node/via_fee_model/src/l1_gas_price/main_node_fetcher.rs
/// This structure maintains the known L1 gas price by periodically querying
/// the main node.
/// It is required since the main node doesn't only observe the current L1 gas price,
/// but also applies adjustments to it in order to smooth out the spikes.
/// The same algorithm cannot be consistently replicated on the external node side,
/// since it relies on the configuration, which may change.
#[derive(Debug)]
pub struct ViaMainNodeFeeParamsFetcher {
    client: Box<DynClient<L2>>,
    main_node_fee_params: RwLock<FeeParams>,
}

impl ViaMainNodeFeeParamsFetcher {
    pub fn new(client: Box<DynClient<L2>>) -> Self {
        Self {
            client: client.for_component("fee_params_fetcher"),
            main_node_fee_params: RwLock::new(FeeParams::sensible_v2_default()),
        }
    }
```

The fetch loop calls `self.client.get_fee_params()` (namespace import: `zksync_web3_decl::namespaces::ZksNamespaceClient`) every 5 seconds (`SLEEP_INTERVAL: Duration = Duration::from_secs(5)`) and logs a warning on failure, keeping the last known value. The wiring layer is `ViaMainNodeFeeParamsFetcherLayer` in `core/node/node_framework/src/implementations/layers/via_main_node_fee_params_fetcher.rs`, added by the builder for the HttpApi and WsApi components.

## 4. Configuration

### 4.1 Config structure

```rust
// core/bin/via_external_node/src/config/mod.rs
/// External Node Config contains all the configuration required for the EN operation.
/// It is split into three parts: required, optional and remote for easier navigation.
#[derive(Debug)]
pub(crate) struct ExternalNodeConfig<R = RemoteENConfig> {
    pub required: RequiredENConfig,
    pub postgres: PostgresConfig,
    pub optional: OptionalENConfig,
    pub observability: ObservabilityENConfig,
    pub experimental: ExperimentalENConfig,
    pub consensus: Option<ConsensusConfig>,
    pub api_component: ApiComponentConfig,
    pub tree_component: TreeComponentConfig,
    pub via_secrets: Option<ViaL1Secrets>,
    pub via_genesis_config: Option<ViaGenesisConfig>,
    pub via_bridge_config: Option<ViaBridgeConfig>,
    pub via_btc_client_config: Option<ViaBtcClientConfig>,
    pub via_btc_watch_config: Option<ViaBtcWatchConfig>,
    pub via_reorg_detector_config: Option<ViaReorgDetectorConfig>,
    pub remote: R,
}
```

Env-based loading populates all the Via configs. Inherited EN settings use the `EN_` prefix (`envy::prefixed("EN_")` for required/optional config, plus `EN_EXPERIMENTAL_`, `EN_API_`, `EN_TREE_`, `EN_SNAPSHOTS_OBJECT_STORE_`); Via configs use their own prefixes via `envy_load` (for example `VIA_REORG_DETECTOR_`):

```rust
// core/bin/via_external_node/src/config/mod.rs
impl ExternalNodeConfig<()> {
    /// Parses the local part of node configuration from the environment.
    pub fn new() -> anyhow::Result<Self> {
        Ok(Self {
            required: RequiredENConfig::from_env()?,
            postgres: PostgresConfig::from_env()?,
            optional: OptionalENConfig::from_env()?,
            observability: ObservabilityENConfig::from_env()?,
            experimental: envy::prefixed("EN_EXPERIMENTAL_")
                .from_env::<ExperimentalENConfig>()
                .context("could not load external node config (experimental params)")?,
            consensus: read_consensus_config().context("read_consensus_config()")?,
            api_component: envy::prefixed("EN_API_")
                .from_env::<ApiComponentConfig>()
                .context("could not load external node config (API component params)")?,
            tree_component: envy::prefixed("EN_TREE_")
                .from_env::<TreeComponentConfig>()
                .context("could not load external node config (tree component params)")?,
            via_secrets: Some(
                ViaL1Secrets::from_env().context("Failed to load VIA BTC client secrets config")?,
            ),
            via_genesis_config: Some(
                ViaGenesisConfig::from_env().context("Failed to load VIA genesis config")?,
            ),
            via_bridge_config: Some(
                ViaBridgeConfig::from_env().context("Failed to load VIA bridge config")?,
            ),
            via_btc_client_config: Some(
                ViaBtcClientConfig::from_env().context("Failed to load VIA BTC client config")?,
            ),
            via_btc_watch_config: Some(
                ViaBtcWatchConfig::from_env().context("Failed to load VIA BTC watch config")?,
            ),
            via_reorg_detector_config: Some(
                ViaReorgDetectorConfig::from_env()
                    .context("Failed to load VIA Reorg detector config")?,
            ),

            remote: (),
        })
    }
```

Caveat visible in the same file: the YAML file path (`from_files`, used with `--config-path` and friends) sets every `via_*` config to `None`. Since the builder errors out when `via_btc_client_config`, `via_reorg_detector_config`, etc. are missing, the env-based configuration is the working path for the Via external node.

### 4.2 Remote config fetch (main node required at startup)

`fetch_remote` is always called in `main()` and fails startup if the main node is unreachable. It fetches only a handful of values (testnet paymaster, genesis config, Bitcoin network via `via_getBitcoinNetwork`, timestamp asserter); most L1 contract addresses are hardcoded placeholders because Via settles on Bitcoin, not Ethereum:

```rust
// core/bin/via_external_node/src/config/mod.rs
impl RemoteENConfig {
    pub async fn fetch(client: &DynClient<L2>) -> anyhow::Result<Self> {
        let l2_testnet_paymaster_addr = client
            .get_testnet_paymaster()
            .rpc_context("get_testnet_paymaster")
            .await?;
        let genesis = client.genesis_config().rpc_context("genesis").await.ok();
        let via_network = client
            .get_bitcoin_network()
            .rpc_context("get_bitcoin_network")
            .await?;
        let timestamp_asserter_address = handle_rpc_response_with_fallback(
            client.get_timestamp_asserter(),
            None,
            "Failed to fetch timestamp asserter address".to_string(),
        )
        .await?;

        Ok(Self {
            l1_bridgehub_proxy_addr: None,
            l1_state_transition_proxy_addr: None,
            l1_transparent_proxy_admin_addr: None,
            l1_diamond_proxy_addr: Address::repeat_byte(1),
            l2_testnet_paymaster_addr,
            l1_erc20_bridge_proxy_addr: None,
            l2_erc20_bridge_addr: None,
            l1_shared_bridge_proxy_addr: None,
            l2_shared_bridge_addr: Some(Address::repeat_byte(1)), // required in state keeper constructor
            l2_legacy_shared_bridge_addr: None,
            l1_weth_bridge_addr: None,
            l2_weth_bridge_addr: None,
            base_token_addr: ETHEREUM_ADDRESS,
            l1_batch_commit_data_generator_mode: genesis
                .as_ref()
                .map(|a| a.l1_batch_commit_data_generator_mode)
                .unwrap_or_default(),
            dummy_verifier: genesis
                .as_ref()
                .map(|a| a.dummy_verifier)
                .unwrap_or_default(),
            l2_timestamp_asserter_addr: timestamp_asserter_address,
            via_network,
        })
    }
```

The fetched `via_network` is then cross-checked against the local L2 chain id by `ViaValidateChainIdsLayer` (section 2).

### 4.3 Config template

The operator-facing template used by `via setup-external-node` and `via external-node`, verbatim:

```toml
# etc/env/configs/via_ext_node.toml
# Note: this file doesn't depend on `base` env and will not contain variables from there.
# All the variables must be provided explicitly.
# This is on purpose: if EN will accidentally depend on the main node env, it may cause problems.

database_url = "postgres://postgres:notsecurepassword@localhost/via_local_ext_node"
test_database_url = "postgres://postgres:notsecurepassword@localhost:5433/via_local_test_ext_node"
database_pool_size = 50
zksync_action = "dont_ask"

[en]
main_node_url = "https://sepolia.era.zksync.dev"
pruning_data_retention_hours = "forever"
snapshots_recovery_enabled = false
http_port = 3060
ws_port = 3061
prometheus_port = 3322
healthcheck_port = 3081
threads_per_server = 128
l2_chain_id = 25223
l1_chain_id = 11155111

req_entities_limit = 10000

state_cache_path = "./db/via_ext_node/state_keeper"
merkle_tree_path = "./db/via_ext_node/lightweight"
max_l1_batches_per_tree_iter = 20

eth_client_url = "https://ethereum-sepolia-rpc.publicnode.com"

api_namespaces = ["via", "eth", "web3", "net", "pubsub", "zks", "en", "debug"]

# Note:
# `bootloader_hash` and `default_aa_hash` are overridden from the `.init.env` values by `zk` tool.
bootloader_hash = "0x0100038581be3d0e201b3cc45d151ef5cc59eb3a0f146ad44f0f72abf00b594c"
default_aa_hash = "0x0100038dc66b69be75ec31653c64cb931678299b9b659472772b2550b703f41c"

# Should be the same as chain.state_keeper.fee_account_addr.
operator_addr = "0xde03a0B5963f75f1C8485B355fF6D30f3093BDE7"

[en.consensus]
config_path = "etc/env/en_consensus_config.yaml"
secrets_path = "etc/env/en_consensus_secrets.yaml"

[en.database]
long_connection_threshold_ms = 2000
slow_query_threshold_ms = 100

[en.snapshots.object_store]
bucket_base_url = "zksync-era-boojnet-external-node-snapshots"
mode = "FileBacked"
file_backed_base_path = "artifacts"
# ^ Intentionally set to coincide with main node's in order to read locally produced snapshots

[en.main_node]
url = "http://127.0.0.1:3050"

[en.gateway]
url = "http://127.0.0.1:3052"


[via_bridge]
# The bridge address
bridge_address = "bcrt1p3s7m76wp5seprjy4gdxuxrr8pjgd47q5s8lu9vefxmp0my2p4t9qh6s8kq"
# The verifiers public keys
verifiers_pub_keys = [
    "03d8e2443ef58aa80fb6256bf3b94d2ecf9117f19cb17661ec60ad35fd84ff4a8b",
    "02043f839b8ecd9ffd79f26ec7d05750555cd0d1e0777cfc84a29b7e38e6324662",
]

[via_genesis]
# The bootstraping inscriptions
bootstrap_txids = []

[via_btc_client]
# The btc client user.
rpc_user = "rpcuser"
# The btc client password.
rpc_password = "rpcpassword"
# Name of the used Bitcoin network
network = "regtest"
# The Bitcoin RPC URL.
rpc_url = "http://0.0.0.0:18443"
# External fee APIs
external_apis = ["https://mempool.space/testnet/api/v1/fees/recommended"]
# Fee strategies
fee_strategies = ["fastestFee"]
# Use RPC to get the fee rate
use_rpc_for_fee_rate = true

[via_btc_watch]
# The interval in milliseconds between btc sender cycles.
poll_interval = 5000
# The number of L1 block to mark the inscription as finalized. 
block_confirmations = 0
# Number of blocks that we should check when restarting the service.
btc_blocks_lag = 1000
# The starting L1 block number from which indexing begins.
start_l1_block_number = 1
# When set to true, the btc_watch starts indexing L1 blocks from the "start_l1_block_number".
restart_indexing = false

[via_reorg_detector]
poll_interval_ms = 5000

[rust]
# `RUST_LOG` environment variable for `env_logger`
# Here we use TOML multiline strings: newlines will be trimmed.
log = """\
warn,\
zksync_node_framework=info,\
zksync_node_consensus=info,\
zksync_consensus_bft=info,\
zksync_consensus_network=info,\
zksync_consensus_storage=info,\
zksync_commitment_generator=info,\
zksync_core=debug,\
zksync_dal=info,\
zksync_db_connection=info,\
zksync_health_check=debug,\
zksync_eth_client=info,\
zksync_state_keeper=info,\
zksync_node_sync=info,\
zksync_storage=info,\
zksync_metadata_calculator=info,\
zksync_merkle_tree=info,\
zksync_node_api_server=info,\
zksync_node_db_pruner=info,\
zksync_reorg_detector=info,\
via_consistency_checker=info,\
zksync_state=debug,\
zksync_utils=debug,\
zksync_types=info,\
zksync_web3_decl=debug,\
loadnext=info,\
vm=info,\
zksync_external_node=info,\
zksync_snapshots_applier=debug,\
via_main_node_reorg_detector=info,\
"""

# `RUST_BACKTRACE` variable
backtrace = "full"
lib_backtrace = "1"
```

Facts to read off the template rather than remember:
- Ports: HTTP 3060, WS 3061, Prometheus 3322, healthcheck 3081.
- Chain ids in the template: `l2_chain_id = 25223`, `l1_chain_id = 11155111`.
- `snapshots_recovery_enabled = false` and `pruning_data_retention_hours = "forever"` in this template (the inherited zksync template `etc/env/configs/ext-node.toml` has `snapshots_recovery_enabled = true` and `pruning_data_retention_hours = 1`, but that template is not the Via one).
- API namespaces include `via` in addition to the standard set.
- A sibling inherited template `etc/env/configs/ext-node.toml` also exists with `l2_chain_id = 25223`, `l1_chain_id = 9` and no `via_*` sections; `via setup-external-node` and `via external-node` operate on `via_ext_node`.

## 5. The `via` RPC namespace

The `via` namespace is enabled in the template's `api_namespaces`. Its actual method surface, verbatim (note that `via_getBridgeAddress` is commented out in source and therefore not served; earlier versions of this document described it as active, which is wrong):

```rust
// core/lib/web3_decl/src/namespaces/via.rs
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

The external node itself is a consumer of `via_getBitcoinNetwork`: `RemoteENConfig::fetch` calls it on the main node at startup (section 4.2).

## 6. Docker image

The image builds the `via_external_node` binary, ships `sqlx` and the core DAL migrations, and runs migrations before starting the node. Verbatim:

```dockerfile
# docker/via-external-node/Dockerfile
# Will work locally only after prior contracts build

FROM lukemathwalker/cargo-chef:0.1.77-rust-bookworm AS cargo-chef

FROM matterlabs/zksync-build-base:latest AS chef

# set of args for use of sccache
ARG SCCACHE_GCS_BUCKET=""
ARG SCCACHE_GCS_SERVICE_ACCOUNT=""
ARG SCCACHE_GCS_RW_MODE=""
ARG RUSTC_WRAPPER=""

ENV SCCACHE_GCS_BUCKET=${SCCACHE_GCS_BUCKET}
ENV SCCACHE_GCS_SERVICE_ACCOUNT=${SCCACHE_GCS_SERVICE_ACCOUNT}
ENV SCCACHE_GCS_RW_MODE=${SCCACHE_GCS_RW_MODE}
ENV RUSTC_WRAPPER=${RUSTC_WRAPPER}

WORKDIR /usr/src/via
RUN apt-get update && apt-get install -y --no-install-recommends protobuf-compiler libprotobuf-dev git && rm -rf /var/lib/apt/lists/*

COPY --from=cargo-chef /usr/local/cargo/bin/cargo-chef /usr/local/cargo/bin/cargo-chef

FROM chef AS planner
COPY . .
RUN cargo chef prepare --recipe-path recipe.json

FROM chef AS cacher
COPY --from=planner /usr/src/via/recipe.json recipe.json
RUN cargo chef cook --release --recipe-path recipe.json --bin via_external_node

FROM chef AS builder
RUN apt-get update && apt-get install -y --no-install-recommends mold && rm -rf /var/lib/apt/lists/*
COPY . .
COPY --from=cacher /usr/local/cargo /usr/local/cargo
COPY --from=cacher /usr/src/via/target target
RUN mold -run cargo build --release --bin via_external_node

FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y curl libpq5 ca-certificates  && rm -rf /var/lib/apt/lists/*

COPY --from=builder /usr/src/via/target/release/via_external_node /usr/bin
COPY --from=builder /usr/local/cargo/bin/sqlx /usr/bin
COPY --from=builder --chmod=775 /usr/src/via/docker/via-external-node/entrypoint.sh /usr/bin
COPY contracts/system-contracts/bootloader/build/artifacts/ /contracts/system-contracts/bootloader/build/artifacts/
COPY contracts/system-contracts/contracts-preprocessed/artifacts/ /contracts/system-contracts/contracts-preprocessed/artifacts/
COPY contracts/system-contracts/contracts-preprocessed/precompiles/artifacts/ /contracts/system-contracts/contracts-preprocessed/precompiles/artifacts/
COPY contracts/system-contracts/artifacts-zk /contracts/system-contracts/artifacts-zk
COPY contracts/l2-contracts/artifacts-zk/ /contracts/l2-contracts/artifacts-zk/
COPY etc/tokens/ /etc/tokens/
COPY etc/ERC20/ /etc/ERC20/
COPY etc/multivm_bootloaders/ /etc/multivm_bootloaders/
COPY core/lib/dal/migrations/ /migrations

ENTRYPOINT [ "sh", "/usr/bin/entrypoint.sh"]
```

```sh
# docker/via-external-node/entrypoint.sh
#!/bin/bash

set -e

# Prepare the database if it's not ready. No-op if the DB is prepared.
sqlx database setup
# Run the external node.
exec via_external_node "$@"
```

## 7. Running it

### 7.1 `via external-node`

The `via` CLI wraps `cargo run --bin via_external_node --release` and sets the bootloader/default AA hashes from the compiled env:

```typescript
// infrastructure/via/src/server.ts
export async function externalNode(reinit: boolean = false, args: string[]) {
    if (process.env.VIA_ENV != 'via_ext_node') {
        console.warn(`WARNING: using ${process.env.VIA_ENV} environment for external node`);
        console.warn('If this is a mistake, set $VIA_ENV to "via_ext_node" or other environment');
    }

    // Set proper environment variables for external node.
    process.env.EN_BOOTLOADER_HASH = process.env.CHAIN_STATE_KEEPER_BOOTLOADER_HASH;
    process.env.EN_DEFAULT_AA_HASH = process.env.CHAIN_STATE_KEEPER_DEFAULT_AA_HASH;

    // On --reinit we want to reset RocksDB and Postgres before we start.
    if (reinit) {
        await utils.confirmAction();
        await db.reset({ core: true, prover: false, verifier: false, indexer: false });
        clean.clean(path.dirname(process.env.EN_MERKLE_TREE_PATH!));
    }

    await utils.spawn(`cargo run  --bin via_external_node --release -- ${args.join(' ')}`);
}
```

```typescript
// infrastructure/via/src/server.ts
export const enCommand = new Command('external-node')
    .description('start via external node')
    .option('--reinit', 'reset postgres and rocksdb before starting')
    .action(async (cmd: Command) => {
        await externalNode(cmd.reinit, cmd.args);
    });
```

### 7.2 `via setup-external-node`

`infrastructure/via/src/setup_en.ts` provides a guided setup registered as:

```typescript
// infrastructure/via/src/setup_en.ts
export const command = new Command('setup-external-node')
    .description('prepare local setup for running external-node on mainnet/testnet')
    .action(async (_: Command) => {
        await configExternalNode();
    });
```

It switches the active env to `via_ext_node`, optionally clears existing state, selects the environment (Testnet, Mainnet, or Local) and a pruning data retention duration, compiles the config, reloads env from file, sets up Postgres, and optionally starts the node. The cleanup and run prompts, verbatim:

```typescript
// infrastructure/via/src/setup_en.ts
async function clearIfNeeded() {
    const filePath = path.join(path.join(process.env.VIA_HOME as string, `etc/env/target/via_ext_node.env`));
    if (!fs.existsSync(filePath)) {
        return true;
    }

    const question = {
        type: 'confirm',
        name: 'cleanup',
        message: 'Do you want to clear the external node database?'
    };

    const answer: { cleanup: boolean } = await prompt(question);
    if (!answer.cleanup) {
        return false;
    }
    const cmd = chalk.yellow;
    console.log(`cleaning up database (${cmd('via clean --config via_ext_node --database')})`);
    await utils.exec('via clean --config via_ext_node --database');
    console.log(`cleaning up db (${cmd('via db drop --core')})`);
    await utils.exec('via db drop --core');
    return true;
}

async function runEnIfAskedTo() {
    const question = {
        type: 'confirm',
        name: 'runRequested',
        message: 'Do you want to run external-node now?'
    };
    const answer: { runRequested: boolean } = await prompt(question);
    if (!answer.runRequested) {
        return false;
    }
    await utils.spawn('via external-node');
}
```

The tail of the setup flow shows the compile-then-reload-then-db-setup order (`load_from_file` is imported from `./env`):

```typescript
// infrastructure/via/src/setup_en.ts
    compileConfig('via_ext_node');
    await updateBootstrapTxidsEnv(network);
    load_from_file();
    console.log(`Setting up postgres (${cmd('via db setup')})`);
    await setupDb({ prover: false, core: true, verifier: false, indexer: false });
    await runEnIfAskedTo();
```

## 8. Operational notes (all grounded in the code above)

- Health checks: the healthcheck server binds `EN_HEALTHCHECK_PORT` (template: 3081) via `HealthCheckLayer`; storage initialization is a precondition in every component, so the node does not serve until the database is initialized.
- Pruning: `PruningLayer` is only added when `pruning_enabled` is set (`add_pruning_layer`, "Pruning is disabled" is logged otherwise); tree pruning is wired into the metadata calculator under the same flag.
- Snapshot recovery: gated by `snapshots_recovery_enabled` (template default: false); when enabled, `ExternalNodeInitStrategyLayer` gets a `SnapshotRecoveryConfig` including the `EN_SNAPSHOTS_OBJECT_STORE_` object store settings.
- Block reverter: `BlockReverterLayer::new(NodeRole::External)` allows rolling back executed batches, Postgres, the Merkle tree, and the state keeper cache; this is what executes the rollback when a reorg or snapshot mismatch requires it.
- Transaction submission: the EN never sequences; `ProxySinkLayer` forwards transactions to the main node.
- Rate limiting to the main node: `main_node_rate_limit_rps` is applied when constructing the main node client in `main()`.
- Main node dependency: startup requires a reachable main node (`fetch_remote` is unconditional). Claims in earlier versions of this document about a "PostgreSQL dump mode" with no live main node have no basis in this code.
- Standalone L1 indexer: a separate indexer workspace exists at `via_indexer/` (binary crate `via_indexer/bin/indexer`, Docker context `docker/via-l1-indexer/`). It is a different process from the EN's built-in BTC watch.
