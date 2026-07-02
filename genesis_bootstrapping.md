# Genesis and Bootstrapping Process in Via L2 Bitcoin ZK-Rollup

This document details the genesis and bootstrapping process for the Via L2 Bitcoin ZK-Rollup system, focusing on how the initial state is defined, generated, and committed to the network.

Two distinct mechanisms are covered:

1. **L2 genesis**: creating the genesis L1 batch, deploying system contracts, and building the initial Merkle tree in Postgres. This machinery is largely inherited from zksync-era (`core/node/genesis/`), with Via-specific modifications such as inserting the BTC token.
2. **Bitcoin bootstrapping**: a `SystemBootstrapping` inscription on Bitcoin L1 that publishes the system wallets (governance, sequencer, bridge, verifiers), protocol version, and base contract hashes. Every Via node (sequencer, verifier, indexer) reads this inscription at first startup to initialize its storage.

## Table of Contents

1. [Overview](#overview)
2. [Primary Code Modules](#primary-code-modules)
3. [Genesis Configuration](#genesis-configuration)
4. [Genesis Process](#genesis-process)
5. [System Bootstrapping](#system-bootstrapping)
6. [Bootstrap Processing at Node Startup](#bootstrap-processing-at-node-startup)
7. [Configuration Files](#configuration-files)
8. [Command-Line Tools](#command-line-tools)
9. [Bootstrapping and Upgrade Relationship](#bootstrapping-and-upgrade-relationship)

## Overview

The genesis process in Via L2 is responsible for initializing the rollup's state, including deploying system contracts, setting up the initial Merkle tree, and creating the genesis block. This process is critical for establishing the foundation of the L2 network and ensuring that all nodes start with the same initial state.

The bootstrapping process involves committing the initial network parameters to the Bitcoin L1 blockchain through a special inscription. The transaction IDs of the bootstrap inscriptions are distributed to nodes via configuration (`VIA_GENESIS_BOOTSTRAP_TXIDS`), and each node fetches and parses the referenced transactions from its Bitcoin RPC node during storage initialization.

## Primary Code Modules

The genesis and bootstrapping functionality is implemented in the following modules:

1. **`core/node/genesis/`**: The core library for L2 genesis (inherited from zksync-era, with Via modifications)
   - `lib.rs`: `GenesisParams`, `insert_genesis_batch`, `create_genesis_l1_batch`
   - `utils.rs`: helpers, including `add_btc_token` (Via-specific)

2. **`core/bin/genesis_generator/`**: A tool for regenerating `etc/env/file_based/genesis.yaml` (inherited from zksync-era)

3. **`core/lib/via_btc_client/`**: The Bitcoin client library
   - `src/types.rs`: `SystemBootstrappingInput`, `SystemBootstrapping`, inscription message tags
   - `src/indexer/parser.rs`: `MessageParser::parse_system_bootstrapping`
   - `src/bootstrap/mod.rs`: `ViaBootstrap::process_bootstrap_messages`
   - `examples/bootstrap.rs`: the tool that creates and sends the `SystemBootstrapping` inscription

4. **`core/lib/types/src/`**: shared types
   - `via_bootstrap.rs`: `BootstrapState` and its validation rules
   - `via_wallet.rs`: `SystemWallets`, `SystemWalletsDetails`, `WalletRole`

5. **`core/lib/config/src/configs/via_consensus.rs`**: `ViaGenesisConfig` (the `bootstrap_txids` list)

6. **Storage initializers** (one per node type):
   - `core/node/via_node_storage_init/`: sequencer / main node
   - `via_verifier/node/via_verifier_storage_init/`: verifier node
   - `via_indexer/node/via_indexer_storage_init/`: indexer node

7. **`infrastructure/via/src/`**: TypeScript CLI (`via`)
   - `bootstrap.ts`: `via bootstrap system-bootstrapping` and `via bootstrap update-bootstrap-tx`
   - `server.ts`: `via server --genesis`

## Genesis Configuration

### Main Genesis Configuration

```yaml
# etc/env/file_based/genesis.yaml
genesis_root: '0xe5130f36c084e419c99f8268d499b19e50622f4c41506439e334c2a38ac4a6f7'
genesis_rollup_leaf_index: 54
genesis_batch_commitment: '0x82b8670fcc6149d66ccb53705b408b66ca868b50b95cfc05a22bccef65563b6e'
genesis_protocol_semantic_version: '0.28.0'
# deprecated
genesis_protocol_version: 28
default_aa_hash: '0x0100055d4fb8cb4bf017843c74ea924928235a8954b327a1e2a88a7568a04b10'
bootloader_hash: '0x010008c3be57ae5800e077b6c2056d9d75ad1a7b4f0ce583407961cc6fe0b678'
l1_chain_id: 9
l2_chain_id: 270
fee_account: '0x0000000000000000000000000000000000000001'
prover:
  dummy_verifier: true
  snark_wrapper_vk_hash: '0x14f97b81e54b35fe673d8708cc1a19e1ea5b5e348e12d31e39824ed4f42bbca2'
l1_batch_commit_data_generator_mode: Rollup
# TODO: uncomment once EVM emulator is present in the `contracts` submodule
# evm_emulator_hash: 0x01000e53aa35d9d19fa99341c2e2901cf93b3668f01569dd5c6ca409c7696b91
```

### Chain-Specific Configuration (inherited zksync-era scaffolding)

The `ZkStack.yaml` files are inherited from the zksync-era zk_toolbox layout. They are present in the repository but the Via node itself is configured through env variables and the file-based configs above.

```yaml
# chains/era/ZkStack.yaml
id: 1
name: era
chain_id: 271
prover_version: NoProofs
configs: ./chains/era/configs/
rocks_db_path: ./chains/era/db/
external_node_config_path: ./chains/era/configs/external_node
artifacts_path: ./chains/era/artifacts/
l1_batch_commit_data_generator_mode: Rollup
base_token:
  address: '0x0000000000000000000000000000000000000001'
  nominator: 1
  denominator: 1
wallet_creation: Localhost
```

```yaml
# ZkStack.yaml
name: zk
l1_network: Localhost
link_to_code: .
chains: ./chains
config: ./configs/
default_chain: era
era_chain_id: 270
prover_version: NoProofs
wallet_creation: Localhost
```

## Genesis Process

The L2 genesis machinery lives in `core/node/genesis/src/lib.rs` and is inherited from zksync-era. The key entities, shown verbatim from this repository:

### 1. Genesis Parameters

```rust
// core/node/genesis/src/lib.rs
#[derive(Debug, Clone)]
pub struct GenesisParams {
    base_system_contracts: BaseSystemContracts,
    system_contracts: Vec<DeployedContract>,
    config: GenesisConfig,
}
```

Parameters are loaded from the genesis configuration:

```rust
// core/node/genesis/src/lib.rs
    pub fn load_genesis_params(config: GenesisConfig) -> Result<GenesisParams, GenesisError> {
        let mut base_system_contracts = BaseSystemContracts::load_from_disk();
        if config.evm_emulator_hash.is_some() {
            base_system_contracts = base_system_contracts.with_latest_evm_emulator();
        }
        let system_contracts = get_system_smart_contracts(config.evm_emulator_hash.is_some());
        Self::from_genesis_config(config, base_system_contracts, system_contracts)
    }
```

### 2. Creating the Genesis L1 Batch

```rust
// core/node/genesis/src/lib.rs
#[allow(clippy::too_many_arguments)]
pub async fn create_genesis_l1_batch(
    storage: &mut Connection<'_, Core>,
    l2_chain_id: L2ChainId,
    protocol_version: ProtocolSemanticVersion,
    base_system_contracts: &BaseSystemContracts,
    system_contracts: &[DeployedContract],
    l1_verifier_config: L1VerifierConfig,
) -> Result<(), GenesisError> {
    let storage_logs = get_storage_logs(l2_chain_id, system_contracts);

    let factory_deps = system_contracts
        .iter()
        .map(|c| {
            (
                BytecodeHash::for_bytecode(&c.bytecode).value(),
                c.bytecode.clone(),
            )
        })
        .collect();

    create_genesis_l1_batch_from_storage_logs_and_factory_deps(
```

This flow creates the protocol version record, the genesis L1 batch header, the genesis L2 block header, inserts the base system contracts (bootloader and default account abstraction) and the system contracts. As a Via-specific modification, the genesis transaction also inserts the BTC token via `add_btc_token(&mut transaction).await?;` (imported from `core/node/genesis/src/utils.rs`).

### 3. Building the Initial Merkle Tree

```rust
// core/node/genesis/src/lib.rs
// Insert genesis batch into the database
pub async fn insert_genesis_batch(
    storage: &mut Connection<'_, Core>,
    genesis_params: &GenesisParams,
) -> Result<GenesisBatchParams, GenesisError> {
    insert_genesis_batch_with_custom_state(storage, genesis_params, None).await
}
```

`insert_genesis_batch_with_custom_state` creates the genesis L1 batch, extracts the storage logs from the system contracts, processes these logs to build the initial Merkle tree, calculates the genesis root hash and commitment, and saves the genesis metadata to the database.

### 4. Generating Genesis Data via Server

The `via server --genesis` command triggers genesis generation:

```typescript
// infrastructure/via/src/server.ts
export async function genesisFromSources() {
    // Note that all the chains have the same chainId at genesis. It will be changed
    // via an upgrade transaction during the registration of the chain.
    await create_genesis('cargo run --bin via_server --release -- --genesis');
}
```

In the server binary, the `--genesis` flag builds a node containing only the genesis-related layers:

```rust
// core/bin/via_server/src/node_builder.rs
    /// Builds the node with the genesis initialization task only.
    pub fn only_genesis(mut self) -> anyhow::Result<ZkStackService> {
        self = self
            .add_pools_layer()?
            .add_query_eth_client_layer()?
            .add_btc_client_layer()?
            .add_storage_initialization_layer(LayerKind::Task)?
            .add_init_node_storage_layer()?;

        Ok(self.node.build())
    }
```

Note that this runs both the era-inherited genesis (`add_storage_initialization_layer`) and the Via bootstrap-driven storage initialization (`add_init_node_storage_layer`, described below).

## System Bootstrapping

The system is bootstrapped on the Bitcoin L1 blockchain through an inscription tagged `SystemBootstrappingMessage`:

```rust
// core/lib/via_btc_client/src/types.rs
    pub static ref SYSTEM_BOOTSTRAPPING_MSG: PushBytesBuf =
        PushBytesBuf::from(b"SystemBootstrappingMessage");
```

### SystemBootstrapping Inscription

```rust
// core/lib/via_btc_client/src/types.rs
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
pub struct SystemBootstrappingInput {
    pub start_block_height: u32,
    pub protocol_version: ProtocolSemanticVersion,
    pub bootloader_hash: H256,
    pub abstract_account_hash: H256,
    pub snark_wrapper_vk_hash: H256,
    pub evm_emulator_hash: H256,
    pub governance_address: BitcoinAddress<NetworkUnchecked>,
    pub sequencer_address: BitcoinAddress<NetworkUnchecked>,
    pub bridge_musig2_address: BitcoinAddress<NetworkUnchecked>,
    pub verifier_p2wpkh_addresses: Vec<BitcoinAddress<NetworkUnchecked>>,
}

#[derive(Clone, Debug, PartialEq)]
pub struct SystemBootstrapping {
    pub common: CommonFields,
    pub input: SystemBootstrappingInput,
}
```

This inscription contains:

- The Bitcoin block height at which the L2 network starts indexing
- The initial protocol semantic version
- The bootloader hash and the default account abstraction (abstract account) hash
- The SNARK wrapper verification key hash
- The EVM emulator hash (zero when no emulator is deployed)
- The governance address (system upgrades and administrative operations)
- The sequencer address
- The bridge MuSig2 (Taproot) address for deposits and withdrawals
- The P2WPKH addresses of the verifiers

The inscription is created using the standard commit-reveal pattern (see `core/lib/via_btc_client/src/inscriber/`): a commit transaction creates a Taproot output committing to the inscription script, and a reveal transaction spends it, revealing the inscription data.

### Parsing the Inscription

The `MessageParser` in `core/lib/via_btc_client/src/indexer/parser.rs` recognizes the message tag and parses the payload. Shown verbatim:

```rust
// core/lib/via_btc_client/src/indexer/parser.rs
    fn parse_system_bootstrapping(
        &mut self,
        instructions: &[Instruction],
        common_fields: &CommonFields,
    ) -> Option<FullInscriptionMessage> {
        if instructions.len() < MIN_SYSTEM_BOOTSTRAPPING_INSTRUCTIONS {
            warn!("Insufficient instructions for system bootstrapping");
            return None;
        }

        let start_block_height = u32::from_be_bytes(
            instructions
                .get(2)?
                .push_bytes()?
                .as_bytes()
                .try_into()
                .ok()?,
        );
        debug!("Parsed start block height: {}", start_block_height);

        let protocol_version = ProtocolSemanticVersion::try_from_packed(U256::from_big_endian(
            instructions.get(3)?.push_bytes()?.as_bytes(),
        ))
        .ok()?;
        debug!("Parsed protocol version");

        let bootloader_hash = H256::from_slice(instructions.get(4)?.push_bytes()?.as_bytes());
        debug!("Parsed bootloader hash");

        let abstract_account_hash = H256::from_slice(instructions.get(5)?.push_bytes()?.as_bytes());
        debug!("Parsed abstract account hash");

        let snark_wrapper_vk_hash = H256::from_slice(instructions.get(6)?.push_bytes()?.as_bytes());

        let evm_emulator_hash = H256::from_slice(instructions.get(7)?.push_bytes()?.as_bytes());

        let network_unchecked_governance_address = instructions.get(8).and_then(|instr| {
            if let Instruction::PushBytes(bytes) = instr {
                std::str::from_utf8(bytes.as_bytes())
                    .ok()
                    .and_then(|s| s.parse::<Address<NetworkUnchecked>>().ok())
            } else {
                None
            }
        })?;

        debug!("Parsed governance address");

        let network_unchecked_sequencer_address = instructions.get(9).and_then(|instr| {
            if let Instruction::PushBytes(bytes) = instr {
                std::str::from_utf8(bytes.as_bytes())
                    .ok()
                    .and_then(|s| s.parse::<Address<NetworkUnchecked>>().ok())
            } else {
                None
            }
        })?;

        debug!("Parsed sequencer address");

        let network_unchecked_bridge_address = instructions.get(10).and_then(|instr| {
            if let Instruction::PushBytes(bytes) = instr {
                std::str::from_utf8(bytes.as_bytes())
                    .ok()
                    .and_then(|s| s.parse::<Address<NetworkUnchecked>>().ok())
            } else {
                None
            }
        })?;

        debug!("Parsed bridge address");

        // network unchecked is required to enable serde serialization and deserialization on the library structs
        let network_unchecked_verifier_addresses = instructions[11..]
            .iter()
            .filter_map(|instr| {
                if let Instruction::PushBytes(bytes) = instr {
                    std::str::from_utf8(bytes.as_bytes())
                        .ok()
                        .and_then(|s| s.parse::<Address<NetworkUnchecked>>().ok())
                } else {
                    None
                }
            })
            .collect::<Vec<_>>();

        debug!(
            "Parsed {} verifier addresses",
            network_unchecked_verifier_addresses.len()
        );

        Some(FullInscriptionMessage::SystemBootstrapping(
            SystemBootstrapping {
                common: common_fields.clone(),
                input: SystemBootstrappingInput {
                    start_block_height,
                    protocol_version,
                    bootloader_hash,
                    abstract_account_hash,
                    snark_wrapper_vk_hash,
                    evm_emulator_hash,
                    governance_address: network_unchecked_governance_address,
                    sequencer_address: network_unchecked_sequencer_address,
                    bridge_musig2_address: network_unchecked_bridge_address,
                    verifier_p2wpkh_addresses: network_unchecked_verifier_addresses,
                },
            },
        ))
    }
```

The minimum instruction count is defined at the top of the parser:

```rust
// core/lib/via_btc_client/src/indexer/parser.rs
const MIN_SYSTEM_BOOTSTRAPPING_INSTRUCTIONS: usize = 11;
```

### ViaBootstrap: Fetching and Validating the Bootstrap State

`ViaBootstrap` fetches the bootstrap transaction referenced in `ViaGenesisConfig`, parses it, and builds a validated `BootstrapState`. Shown verbatim in full:

```rust
// core/lib/via_btc_client/src/bootstrap/mod.rs
use std::{str::FromStr, sync::Arc};

use bitcoin::Txid;
use zksync_config::configs::via_consensus::ViaGenesisConfig;
use zksync_types::{via_bootstrap::BootstrapState, via_wallet::SystemWallets};

use crate::{
    client::BitcoinClient, indexer::MessageParser, traits::BitcoinOps,
    types::FullInscriptionMessage,
};

#[derive(Debug, Clone)]
pub struct ViaBootstrap {
    pub config: ViaGenesisConfig,
    pub client: Arc<BitcoinClient>,
}

impl ViaBootstrap {
    pub fn new(client: Arc<BitcoinClient>, config: ViaGenesisConfig) -> Self {
        Self { client, config }
    }

    pub async fn process_bootstrap_messages(&self) -> anyhow::Result<BootstrapState> {
        let network = self.client.get_network();
        let mut parser = MessageParser::new(network);

        let bootstrap_txid = self
            .config
            .bootstrap_txids
            .first()
            .ok_or_else(|| anyhow::anyhow!("Bootstrap transaction not found"))?;

        let txid = Txid::from_str(&bootstrap_txid)?;
        let tx = self.client.get_transaction(&txid).await?;
        let block_height = self.client.fetch_block_height().await? as u32;
        let messages = parser.parse_system_transaction(&tx, block_height, None);

        let message = messages
            .first()
            .ok_or_else(|| anyhow::anyhow!("Bootstrap message not found"))?;

        let state = match message {
            FullInscriptionMessage::SystemBootstrapping(sb) => sb.clone(),
            _ => anyhow::bail!("Invalid Bootstrap message"),
        };

        let verifiers = state
            .input
            .verifier_p2wpkh_addresses
            .iter()
            .map(|addr| addr.clone().require_network(network).unwrap())
            .collect::<Vec<_>>();

        let bootstrap_state = BootstrapState {
            wallets: SystemWallets {
                sequencer: state
                    .input
                    .sequencer_address
                    .require_network(network)
                    .unwrap(),
                bridge: state
                    .input
                    .bridge_musig2_address
                    .require_network(network)
                    .unwrap(),
                governance: state
                    .input
                    .governance_address
                    .require_network(network)
                    .unwrap(),
                verifiers,
            },
            bootstrap_tx_id: state.common.tx_id,
            starting_block_number: state.input.start_block_height,
            bootloader_hash: state.input.bootloader_hash,
            abstract_account_hash: state.input.abstract_account_hash,
            snark_wrapper_vk_hash: state.input.snark_wrapper_vk_hash,
            evm_emulator_hash: state.input.evm_emulator_hash,
            protocol_version: state.input.protocol_version,
        };

        // Validate the final state
        bootstrap_state.validate()?;

        Ok(bootstrap_state)
    }
}
```

### BootstrapState and Its Validation Rules

```rust
// core/lib/types/src/via_bootstrap.rs
use bitcoin::Txid;
use zksync_basic_types::{protocol_version::ProtocolSemanticVersion, H256};

use crate::via_wallet::SystemWallets;

#[derive(Debug, Clone)]
pub struct BootstrapState {
    pub wallets: SystemWallets,
    pub bootstrap_tx_id: Txid,
    pub starting_block_number: u32,
    pub bootloader_hash: H256,
    pub abstract_account_hash: H256,
    pub snark_wrapper_vk_hash: H256,
    pub evm_emulator_hash: H256,
    pub protocol_version: ProtocolSemanticVersion,
}

impl BootstrapState {
    pub fn validate(&self) -> anyhow::Result<()> {
        if !self.wallets.sequencer.script_pubkey().is_p2wpkh() {
            anyhow::bail!("Sequencer must be P2WPKH");
        }

        if !self.wallets.bridge.script_pubkey().is_p2tr() {
            anyhow::bail!("Bridge must be Taproot");
        }

        if !self.wallets.governance.script_pubkey().is_p2wsh()
            && !self.wallets.governance.script_pubkey().is_p2wpkh()
        {
            anyhow::bail!("Governance must be P2WSH or P2WPKH");
        }

        if !self
            .wallets
            .verifiers
            .iter()
            .all(|a| a.script_pubkey().is_p2wpkh())
        {
            anyhow::bail!("All verifiers must be P2WPKH");
        }

        if self.starting_block_number == 0 {
            anyhow::bail!("Starting block number must be > 0");
        }

        Ok(())
    }
}
```

### SystemWallets

The wallet set extracted from the bootstrap inscription:

```rust
// core/lib/types/src/via_wallet.rs
#[derive(Debug, Clone, PartialEq)]
pub struct SystemWallets {
    pub sequencer: Address,
    pub verifiers: Vec<Address>,
    pub governance: Address,
    pub bridge: Address,
}
```

The same file defines `WalletRole` (`Sequencer`, `Verifier`, `Bridge`, `Gov`) and `SystemWalletsDetails(pub HashMap<WalletRole, WalletInfo>)`, plus `TryFrom<&BootstrapState> for SystemWalletsDetails`, which is what the storage initializers persist to the database.

## Bootstrap Processing at Node Startup

Each node type has a storage initializer that runs `ViaBootstrap::process_bootstrap_messages` on first startup and persists the results.

### Main Node (Sequencer)

The layer is wired in the server node builder:

```rust
// core/bin/via_server/src/node_builder.rs
    fn add_init_node_storage_layer(mut self) -> anyhow::Result<Self> {
        let via_genesis_config = try_load_config!(self.configs.via_genesis_config);
        let via_btc_watch_config = try_load_config!(self.configs.via_btc_watch_config);

        self.node.add_layer(ViaNodeStorageInitializerLayer {
            via_genesis_config,
            via_btc_watch_config,
        });
        Ok(self)
    }
```

The layer itself:

```rust
// core/node/node_framework/src/implementations/layers/via_node_storage_init.rs
#[derive(Debug)]
pub struct ViaNodeStorageInitializerLayer {
    pub via_genesis_config: ViaGenesisConfig,
    pub via_btc_watch_config: ViaBtcWatchConfig,
}

#[derive(Debug, FromContext)]
#[context(crate = crate)]
pub struct Input {
    pub master_pool: PoolResource<MasterPool>,
    pub btc_client_resource: BtcClientResource,
}

#[derive(Debug, IntoContext)]
#[context(crate = crate)]
pub struct Output {}

#[async_trait::async_trait]
impl WiringLayer for ViaNodeStorageInitializerLayer {
    type Input = Input;
    type Output = Output;

    fn layer_name(&self) -> &'static str {
        "Via_node_storage_initializer"
    }

    async fn wire(self, input: Self::Input) -> Result<Self::Output, WiringError> {
        let client = input.btc_client_resource.default;
        let pool = input.master_pool.get().await?;

        ViaMainNodeStorageInitializer::new(
            pool,
            client.clone(),
            self.via_genesis_config,
            self.via_btc_watch_config,
        )
        .await?;

        Ok(Output {})
    }
}
```

The initializer runs a wallets step and an indexer step:

```rust
// core/node/via_node_storage_init/src/lib.rs
#[derive(Debug, Clone)]
pub struct ViaMainNodeStorageInitializer {}

impl ViaMainNodeStorageInitializer {
    pub async fn new(
        pool: ConnectionPool<Core>,
        client: Arc<BitcoinClient>,
        via_genesis_config: ViaGenesisConfig,
        btc_watch_config: ViaBtcWatchConfig,
    ) -> anyhow::Result<Self> {
        let bootstrap = ViaBootstrap::new(client, via_genesis_config);

        let wallets = ViaWalletsInitializer::new(pool.clone(), bootstrap.clone());
        let indexer = ViaIndexerInitializer::new(pool, bootstrap, btc_watch_config);

        wallets.initialize_storage().await?;
        indexer.initialize_storage().await?;

        Ok(Self {})
    }
}
```

The wallets step persists the system wallets to the database if they are not yet stored:

```rust
// core/node/via_node_storage_init/src/wallets.rs
    pub async fn initialize_storage(&self) -> anyhow::Result<()> {
        if !self.is_initialized().await? {
            let state = self.bootstrap.process_bootstrap_messages().await?;

            let indexer_wallets_details = SystemWalletsDetails::try_from(&state)?;

            self.pool
                .connection()
                .await?
                .via_wallet_dal()
                .insert_wallets(&indexer_wallets_details, state.starting_block_number as i64)
                .await?;

            tracing::info!("System wallets initialized");
        }
        Ok(())
    }
```

The indexer step records the Bitcoin block height at which the `via_btc_watch` component starts indexing:

```rust
// core/node/via_node_storage_init/src/indexer.rs
    pub async fn initialize_storage(&self) -> anyhow::Result<()> {
        let is_initialized = self.is_initialized().await?;

        if !is_initialized {
            let state = self.bootstrap.process_bootstrap_messages().await?;
            self.pool
                .connection()
                .await?
                .via_indexer_dal()
                .init_indexer_metadata("via_btc_watch", state.starting_block_number)
                .await?;
            tracing::info!("Indexer storage initialized");
        } else if is_initialized && self.btc_watch_config.restart_indexing {
            self.pool
                .connection()
                .await?
                .via_indexer_dal()
                .update_last_processed_l1_block(
                    "via_btc_watch",
                    self.btc_watch_config.start_l1_block_number,
                )
                .await?;
            tracing::info!(
                "Update the indexer start block number {}",
                self.btc_watch_config.start_l1_block_number
            );
        }

        Ok(())
    }
```

### Verifier Node

The verifier has its own initializer (wired via `ViaVerifierInitLayer` in `via_verifier/bin/verifier_server/src/node_builder.rs` through `core/node/node_framework/src/implementations/layers/via_verifier_storage_init.rs`). It additionally seeds the verifier's protocol version table from the bootstrap state:

```rust
// via_verifier/node/via_verifier_storage_init/src/lib.rs
#[derive(Debug, Clone)]
pub struct ViaVerifierStorageInitializer {}

impl ViaVerifierStorageInitializer {
    pub async fn new(
        pool: ConnectionPool<Verifier>,
        client: Arc<BitcoinClient>,
        via_genesis_config: ViaGenesisConfig,
        btc_watch_config: ViaBtcWatchConfig,
    ) -> anyhow::Result<Self> {
        // Check if already initialized
        if pool
            .connection()
            .await?
            .via_protocol_versions_dal()
            .latest_protocol_semantic_version()
            .await?
            .is_some()
        {
            tracing::info!("Verifier storage already initialized");
            return Ok(Self {});
        }

        let bootstrap = ViaBootstrap::new(client, via_genesis_config);
        let bootstrap_state = bootstrap.process_bootstrap_messages().await?;

        let genesis = Arc::new(VerifierGenesis {
            bootstrap: bootstrap_state.clone(),
            pool: pool.clone(),
        });

        let indexer =
            ViaIndexerInitializer::new(pool.clone(), bootstrap_state.clone(), btc_watch_config);
        let wallets = ViaWalletsInitializer::new(pool, bootstrap_state);

        genesis.initialize_storage().await?;
        wallets.initialize_storage().await?;
        indexer.initialize_storage().await?;

        Ok(Self {})
    }
}
```

```rust
// via_verifier/node/via_verifier_storage_init/src/genesis.rs
impl VerifierGenesis {
    pub async fn initialize_storage(&self) -> anyhow::Result<()> {
        if self.is_initialized().await? {
            return Ok(());
        }

        let mut storage = self.pool.connection_tagged("verifier_genesis").await?;
        let mut transaction = storage.start_transaction().await?;

        transaction
            .via_protocol_versions_dal()
            .save_protocol_version(
                self.bootstrap.protocol_version,
                self.bootstrap.bootloader_hash.as_bytes(),
                self.bootstrap.abstract_account_hash.as_bytes(),
                H256::zero().as_bytes(),
                self.bootstrap.snark_wrapper_vk_hash.as_bytes(),
            )
            .await?;

        transaction
            .via_protocol_versions_dal()
            .mark_upgrade_as_executed(H256::zero().as_bytes())
            .await?;

        transaction.commit().await?;

        Ok(())
    }
```

### Indexer Node

The standalone indexer binary uses `ViaIndexerStorageInitializerLayer` (`core/node/node_framework/src/implementations/layers/via_indexer_storage_init.rs`, wired in `via_indexer/bin/indexer/src/node_builder.rs`), backed by:

```rust
// via_indexer/node/via_indexer_storage_init/src/lib.rs
#[derive(Debug)]
pub struct ViaIndexerStorageInitializer {
    pool: ConnectionPool<Indexer>,
    bootstrap: ViaBootstrap,
}

impl ViaIndexerStorageInitializer {
    pub fn new(
        pool: ConnectionPool<Indexer>,
        client: Arc<BitcoinClient>,
        config: ViaGenesisConfig,
    ) -> Self {
        let bootstrap = ViaBootstrap::new(client, config);

        Self { pool, bootstrap }
    }

    pub async fn indexer_wallets(&self) -> anyhow::Result<SystemWallets> {
        if let Some(system_wallets) = self.fetch_indexer_wallets_from_db().await? {
            return Ok(system_wallets);
        }
        Ok(self.init_indexer_wallets().await?)
    }

    pub async fn init_indexer_wallets(&self) -> anyhow::Result<SystemWallets> {
        let state = self.bootstrap.process_bootstrap_messages().await?;

        let indexer_wallets_details = SystemWalletsDetails::try_from(&state)?;

        self.pool
            .connection()
            .await?
            .via_wallet_dal()
            .insert_wallets(&indexer_wallets_details)
            .await?;

        let wallets = state.wallets;

        tracing::info!("Loaded the indexer wallets from bootstrap inscriptions");

        Ok(wallets)
    }
```

## Configuration Files

### 1. ViaGenesisConfig: Bootstrap Transaction IDs

The only Via-specific genesis configuration is the list of bootstrap transaction IDs. Shown verbatim in full:

```rust
// core/lib/config/src/configs/via_consensus.rs
use std::str::FromStr;

use bitcoin::Txid;
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Default, Deserialize, Clone, PartialEq)]
pub struct ViaGenesisConfig {
    /// List of transaction IDs to bootstrap the indexer.
    pub bootstrap_txids: Vec<String>,
}

impl ViaGenesisConfig {
    /// Get the bootstrap transaction IDs.
    pub fn bootstrap_txids(&self) -> anyhow::Result<Vec<Txid>> {
        self.bootstrap_txids
            .iter()
            .map(|txid| Txid::from_str(txid).map_err(anyhow::Error::from))
            .collect()
    }
}
```

It is loaded from the environment with the `VIA_GENESIS_` prefix:

```rust
// core/lib/env_config/src/via_consensus.rs
use zksync_config::configs::via_consensus::ViaGenesisConfig;

use crate::{envy_load, FromEnv};

impl FromEnv for ViaGenesisConfig {
    fn from_env() -> anyhow::Result<Self> {
        envy_load("via_genesis", "VIA_GENESIS_")
    }
}
```

So the env variable is `VIA_GENESIS_BOOTSTRAP_TXIDS`, a comma-separated list of txids.

### 2. Bootstrap Metadata Files (`etc/env/via/genesis/<network>/`)

When the bootstrap inscription is created, its metadata is written to `etc/env/via/genesis/<network>/`. For testnet the repository contains:

```
etc/env/via/genesis/testnet/
Attest_37361c8374bb6f7628af8b956b465aa16e4c9f0f8796bf87027b5b08ffafb804.json
Attest_60e38f3819e1f6046611d1ff7adadfec7e5ba62798e1d6e9f48bbe06bbca69b6.json
SystemBootstrapping.json
```

```json
// etc/env/via/genesis/testnet/SystemBootstrapping.json
{
  "tx_type": "SystemBootstrapping",
  "system_tx_id": "d942dec4042c8f4db718abb0471785481dd857e6060728a4b3d34c5776a62e0a",
  "propose_sequencer_tx_id": "f46263a9db0354a0ea8f8687d08fe0b543c98dbfbdce61a4e63e620cdab3897f"
}
```

The `via bootstrap update-bootstrap-tx` command reads these files and injects the txids into the node's env file:

```typescript
// infrastructure/via/src/bootstrap.ts
export async function updateBootstrapTxidsEnv(network: string) {
    let genesisTxIds = process.env.VIA_GENESIS_BOOTSTRAP_TXIDS;

    if (!genesisTxIds || genesisTxIds === '""') {
        const genesisDir = path.join(process.env.VIA_HOME!, `etc/env/via/genesis/${network}`);
        const files = await fs.readdir(genesisDir);

        const txids = [];
        // Process first the System inscriptions
        const data = JSON.parse(await fs.readFile(path.join(genesisDir, 'SystemBootstrapping.json'), 'utf-8'));
        if (data.tx_type != 'SystemBootstrapping') {
            throw Error('Invalid System Bootstrapping');
        }
        txids.push(data.system_tx_id);
        txids.push(data.propose_sequencer_tx_id);

        // Process the Attestation
        for (let i = 0; i < files.length; i++) {
            const data = JSON.parse(await fs.readFile(path.join(genesisDir, files[i]), 'utf-8'));
            if (data.tx_type == 'Attest') {
                txids.push(data.tx_id);
            }
        }
        genesisTxIds = txids.join(',');
    }

    const envFilePath = path.join(process.env.VIA_HOME!, `etc/env/target/${process.env.VIA_ENV}.env`);
    console.log(`Updating file ${envFilePath}`);

    await updateEnvVariable(envFilePath, 'VIA_GENESIS_BOOTSTRAP_TXIDS', genesisTxIds);

    console.log(`Updated VIA_GENESIS_BOOTSTRAP_TXIDS with: ${genesisTxIds}`);
}
```

Note that `ViaBootstrap::process_bootstrap_messages` currently only reads the **first** txid in the list (the `SystemBootstrapping` inscription itself); the sequencer proposal and attestation txids are kept in the list for the indexer components.

### 3. L2 Genesis Configuration (`etc/env/file_based/genesis.yaml`)

Described in [Genesis Configuration](#genesis-configuration) above: genesis root hash, rollup leaf index, batch commitment, protocol semantic version, bootloader / default AA hashes, L1 and L2 chain IDs, fee account, and the prover's SNARK wrapper VK hash.

## Command-Line Tools

### 1. Creating the Bootstrap Inscription

The inscription is sent by the `bootstrap` example in the Bitcoin client library, driven by the `via` CLI:

```bash
via bootstrap system-bootstrapping \
    --network <network> \
    --rpc-url <rpcUrl> \
    --rpc-username <rpcUsername> \
    --rpc-password <rpcPassword> \
    --start-block <startBlock> \
    --private-key <privateKey> \
    --bridge-wallet-path <bridgeWalletPath> \
    --verifiers-pub-keys <verifiersPubKeys> \
    --governance-address <governanceAddress> \
    --sequencer-address <sequencerAddress>
```

Under the hood this reads the hashes from `etc/env/file_based/genesis.yaml` and the bridge MuSig2 wallet file, then runs the Rust example:

```typescript
// infrastructure/via/src/bootstrap.ts
    let cmd = `cargo run --example bootstrap ${network} ${rpcUrl} ${rpcUsername} ${rpcPassword} SystemBootstrapping ${privateKey} `;
    cmd += `${startBlock} ${protocolVersion} ${bootloader_hash} ${default_aa_hash} ${snark_wrapper_vk_hash} ${evm_emulator_hash} `;
    cmd += `${governanceAddress} ${sequencerAddress} ${bridgeAddress} "${merkleRoot}" ${bridgeVerifiersPubKeys} ${verifiersPubKeys}`;

    await utils.spawn(cmd);
```

The example (`core/lib/via_btc_client/examples/bootstrap.rs`) builds the input, recomputes the bridge Taproot address from the MuSig2 public keys to check consistency, and inscribes:

```rust
// core/lib/via_btc_client/examples/bootstrap.rs
    let computed_bridge_musig2_address =
        compute_bridge_address(bridge_verifier_public_keys, network, merkle_root)?
            .as_unchecked()
            .clone();

    if bridge_musig2_address != computed_bridge_musig2_address {
        anyhow::bail!(
            "Bridge address mismatch: expected {:?}, got {:?}",
            bridge_musig2_address,
            computed_bridge_musig2_address
        );
    }
    // Bootstrapping message
    let input = SystemBootstrappingInput {
        start_block_height,
        protocol_version,
        bootloader_hash,
        abstract_account_hash,
        snark_wrapper_vk_hash,
        evm_emulator_hash,
        governance_address,
        sequencer_address,
        verifier_p2wpkh_addresses,
        bridge_musig2_address,
    };

    let bootstrap_info = inscriber
        .inscribe(InscriptionMessage::SystemBootstrapping(input))
        .await?;
    info!(
        "Bootstrapping tx sent: {:?}",
        &bootstrap_info.final_reveal_tx.txid
    );

    Ok(bootstrap_info.final_reveal_tx.txid)
```

After sending, the example parses the transaction back with `MessageParser::parse_system_transaction` as a sanity check and writes `etc/env/via/genesis/<network>/SystemBootstrapping.json`. There is also a `parse_bootstrap.rs` example for inspecting an existing bootstrap transaction.

### 2. Injecting the Bootstrap Txids

```bash
via bootstrap update-bootstrap-tx --network <network>
```

This runs `updateBootstrapTxidsEnv` (shown above) and writes `VIA_GENESIS_BOOTSTRAP_TXIDS` into `etc/env/target/${VIA_ENV}.env`.

### 3. Server Genesis Command

```bash
via server --genesis
```

This runs `cargo run --bin via_server --release -- --genesis`, which builds the `only_genesis()` node (era genesis layers plus `ViaNodeStorageInitializerLayer`) and exits after initialization.

### 4. Genesis Generator (inherited from zksync-era)

The `genesis_generator` tool in `core/bin/genesis_generator/` regenerates `etc/env/file_based/genesis.yaml` against a temporary database:

```bash
cargo run --bin genesis_generator -- [--config-path <path>] [--check]
```

```rust
// core/bin/genesis_generator/src/main.rs
#[derive(Debug, Parser)]
#[command(author = "Matter Labs", version, about = "Genesis config generator", long_about = None)]
struct Cli {
    #[arg(long)]
    config_path: Option<std::path::PathBuf>,
    #[arg(long, default_value = "false")]
    check: bool,
}
```

- `--config-path`: path to a secrets YAML (otherwise database secrets are read from env)
- `--check`: verify that the committed genesis config matches the regenerated one

## Bootstrapping and Upgrade Relationship

The bootstrapping process establishes the foundation for future system upgrades:

1. **Governance Address**: The governance address specified in the `SystemBootstrapping` inscription is the wallet authorized to send system upgrade inscriptions (see `parse_system_contract_upgrade_message` and the `SYSTEM_CONTRACT_UPGRADE_MSG` tag in `core/lib/via_btc_client/`).

2. **Initial Protocol Version**: The `protocol_version` field of `SystemBootstrappingInput` is recorded on-chain and seeded into the node databases (see `VerifierGenesis::initialize_storage` above).

3. **Base System Contracts**: The bootloader hash and abstract account hash in the inscription establish the initial versions of these components; the SNARK wrapper VK hash pins the verifier key.

4. **Upgrade Path**: After bootstrapping, the system is upgraded through `SystemContractUpgrade` inscriptions (`SystemContractUpgradeProposalInput` in `core/lib/via_btc_client/src/types.rs`), which carry a new protocol version, new bootloader / default account code hashes, an optional EVM emulator hash, and system contract deployments. The system wallets themselves can be rotated via `UpdateSequencer`, `UpdateGovernance`, `UpdateBridgeProposal`, and `UpdateBridge` messages defined in the same file.

This relationship ensures a secure and controlled upgrade path from the initial bootstrapped state to future protocol versions.
