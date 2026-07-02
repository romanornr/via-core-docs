# Via L2 Bitcoin ZK-Rollup: Configuration

This document describes the configuration files, environment variables, and parameters that affect component behavior and deployment in the Via L2 Bitcoin ZK-Rollup system. Every struct, key, and variable below is taken verbatim from the via-core source tree.

## Table of Contents

1. [Configuration Architecture](#configuration-architecture)
2. [Global Configuration Files](#global-configuration-files)
3. [The via env Configuration Machinery](#the-via-env-configuration-machinery)
4. [Via Component Configuration](#via-component-configuration)
5. [Secrets Configuration](#secrets-configuration)
6. [Wallet Configuration](#wallet-configuration)
7. [Fee Rate Configuration](#fee-rate-configuration)
8. [Server Component Selection](#server-component-selection)
9. [Bridge Configuration](#bridge-configuration)
10. [Docker Deployment Configuration](#docker-deployment-configuration)
11. [External Node Configuration](#external-node-configuration)
12. [Conclusion](#conclusion)

## Configuration Architecture

The Via L2 system uses a layered configuration approach:

1. **Base TOML defaults**: `etc/env/base/*.toml` holds default values for every config section. Via-specific files are `via_bridge.toml`, `via_btc_client.toml`, `via_btc_sender.toml`, `via_btc_watch.toml`, `via_celestia.toml`, `via_genesis.toml`, `via_logs.toml`, `via_private.toml`, `via_reorg_detector.toml`, and `via_verifier.toml`, alongside inherited zkSync files such as `chain.toml`, `api.toml`, `database.toml`, `contracts.toml`, `eth_sender.toml`, `fri_prover.toml`, and others.
2. **Per-environment overlays**: `etc/env/configs/<env>.toml` files import the base layer and override specific keys. The Via environments are `via.toml`, `via_verifier.toml`, `via_coordinator.toml`, `via_ext_node.toml`, and `via_indexer.toml`.
3. **Compiled env files**: the `via` CLI compiles the merged TOML into a flat `etc/env/target/<env>.env` file. Each `[section] key` pair becomes an uppercase `SECTION_KEY` environment variable (for example `[via_btc_client] rpc_url` becomes `VIA_BTC_CLIENT_RPC_URL`; arrays are joined with commas).
4. **Process environment**: the Rust binaries read configuration exclusively from environment variables through `envy`-based `FromEnv` implementations in `core/lib/env_config/src/`.

The main server binary is env-only. It accepts YAML paths for secrets and wallets, but rejects config, genesis, and contracts files:

```rust
// core/bin/via_server/src/main.rs
    let genesis = match opt.genesis_path {
        Some(_path) => {
            return Err(anyhow::anyhow!(
                "The Via Server does not support configuration files at this point. Please use env variables."
            ));
        }
        None => GenesisConfig::from_env().context("Failed to load genesis from env")?,
    };
```

### Environment variable prefixes

Each Via config struct is loaded with a fixed `envy` prefix, defined in `core/lib/env_config/src/via_*.rs`:

| Config struct | Loader call | Env prefix | Base file |
|---|---|---|---|
| `ViaGenesisConfig` | `envy_load("via_genesis", "VIA_GENESIS_")` (via_consensus.rs) | `VIA_GENESIS_` | `etc/env/base/via_genesis.toml` |
| `ViaBridgeConfig` | `envy_load("via_bridge", "VIA_BRIDGE_")` | `VIA_BRIDGE_` | `etc/env/base/via_bridge.toml` |
| `ViaBtcClientConfig` | `envy_load("via_btc_client", "VIA_BTC_CLIENT_")` | `VIA_BTC_CLIENT_` | `etc/env/base/via_btc_client.toml` |
| `ViaBtcWatchConfig` | `envy_load("via_btc_watch", "VIA_BTC_WATCH_")` | `VIA_BTC_WATCH_` | `etc/env/base/via_btc_watch.toml` |
| `ViaBtcSenderConfig` | `envy_load("via_btc_sender", "VIA_BTC_SENDER_")` | `VIA_BTC_SENDER_` | `etc/env/base/via_btc_sender.toml` |
| `ViaVerifierConfig` | `envy_load("via_verifier", "VIA_VERIFIER_")` | `VIA_VERIFIER_` | `etc/env/base/via_verifier.toml` |
| `ViaCelestiaConfig` | `envy_load("via_celestia_client", "VIA_CELESTIA_CLIENT_")` | `VIA_CELESTIA_CLIENT_` | `etc/env/base/via_celestia.toml` |
| `ViaReorgDetectorConfig` | `envy_load("via_reorg_detector", "VIA_REORG_DETECTOR_")` | `VIA_REORG_DETECTOR_` | `etc/env/base/via_reorg_detector.toml` |
| `ViaL1Secrets` | `envy_load("via_l1_secrets", "VIA_BTC_CLIENT_")` | `VIA_BTC_CLIENT_` | `etc/env/base/via_private.toml` |
| `ViaL2Secrets` | `envy_load("via_l2_secrets", "VIA_L2_CLIENT_")` | `VIA_L2_CLIENT_` | `etc/env/base/via_private.toml` |
| `ViaDASecrets` | `envy_load("via_da_secrets", "VIA_CELESTIA_CLIENT_")` | `VIA_CELESTIA_CLIENT_` | `etc/env/base/via_private.toml` |

## Global Configuration Files

### ZkStack.yaml

The repository root contains a `ZkStack.yaml` inherited from the zkSync Era stack tooling:

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

### Chain configuration (chains/era/ZkStack.yaml)

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

### Genesis configuration (etc/env/file_based/genesis.yaml)

The file-based genesis config is part of the inherited zkSync file-based config set (`etc/env/file_based/` also contains `contracts.yaml`, `external_node.yaml`, `general.yaml`, `secrets.yaml`, and `wallets.yaml`). Note that `via_server` itself loads `GenesisConfig::from_env()`; the YAML file feeds the inherited tooling.

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

## The via env Configuration Machinery

The `via` CLI (in `infrastructure/via/`) selects, compiles, and loads environments.

`via env <name>` records the current environment in `etc/env/.current` and compiles the TOML overlay if no `.env` file exists yet:

```typescript
// infrastructure/via/src/env.ts
export function set(environment: string, print: boolean = false, soft: boolean = false) {
    if (!fs.existsSync(`etc/env/target/${environment}.env`) && !fs.existsSync(`etc/env/configs/${environment}.toml`)) {
        console.error(
            `Unknown environment: ${environment}.\nCreate an environment file etc/env/target/${environment}.env or etc/env/configs/${environment}.toml`
        );
        process.exit(1);
    }
    fs.writeFileSync('etc/env/.current', environment);
    process.env.VIA_ENV = environment;
    const envFile = (process.env.ENV_FILE = `etc/env/target/${environment}.env`);
    if (!fs.existsSync(envFile)) {
        // No .env file found - we should compile it!
        config.compileConfig(environment);
    }

    // Only reload if we did NOT pass the --soft flag
    if (!soft) {
        reload(environment);
    }
    get(print);
}
```

The compiler flattens the merged TOML tree into `SECTION_KEY=value` lines:

```typescript
// infrastructure/via/src/config.ts
export function compileConfig(environment?: string) {
    environment ??= process.env.VIA_ENV!;
    const config = loadConfig(environment);
    const variables = collectVariables(config);

    let outputFileContents = '';
    variables.forEach((value: string, key: string) => {
        outputFileContents += `${key}=${value}\n`;
    });

    const outputFileName = `etc/env/target/${environment}.env`;
    fs.writeFileSync(outputFileName, outputFileContents);
    console.log(`Configs compiled for ${environment}`);
}
```

Environment overlay files import the base layer via `__imports__`:

```toml
# etc/env/configs/via.toml
__imports__ = ["base", "l2-inits/via.init.env"]
```

### Bootstrap transaction injection

After the bootstrap inscriptions are created, the CLI writes their transaction IDs into the compiled env file as `VIA_GENESIS_BOOTSTRAP_TXIDS`:

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

## Via Component Configuration

### ViaGenesisConfig

```rust
// core/lib/config/src/configs/via_consensus.rs
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

```toml
# etc/env/base/via_genesis.toml
[via_genesis]
# The bootstraping inscriptions
bootstrap_txids = []
```

Env prefix: `VIA_GENESIS_` (so the list arrives as `VIA_GENESIS_BOOTSTRAP_TXIDS`, comma-separated, normally injected by `via bootstrap`).

### ViaBridgeConfig

```rust
// core/lib/config/src/configs/via_bridge.rs
#[derive(Debug, Serialize, Default, Deserialize, Clone, PartialEq)]
pub struct ViaBridgeConfig {
    /// The verifiers public keys.
    pub verifiers_pub_keys: Vec<String>,

    /// The bridge address.
    pub bridge_address: String,
}

impl ViaBridgeConfig {
    pub fn bridge_address(&self) -> anyhow::Result<Address> {
        Ok(Address::from_str(&self.bridge_address)
            .expect("Invalid bridge address")
            .assume_checked())
    }
}
```

```toml
# etc/env/base/via_bridge.toml
[via_bridge]
# The bridge address
bridge_address = "bcrt1pnsuff8wtl0e9qkmy0h6ve0l9wkkgdpfgefy9nn8y7jdde9lpa9sqej29cs"
# The verifiers public keys
verifiers_pub_keys = [
    "03d8e2443ef58aa80fb6256bf3b94d2ecf9117f19cb17661ec60ad35fd84ff4a8b",
    "02043f839b8ecd9ffd79f26ec7d05750555cd0d1e0777cfc84a29b7e38e6324662",
]
```

Env prefix: `VIA_BRIDGE_`. Real-world override examples from the upgrade guide:

```bash
# docs/via_guides/upgrade.md
VIA_BRIDGE_VERIFIERS_PUB_KEYS=025b3c069378f860cc4dae864a491e0cd33cc559b9f82fc856d4dcc74d3d763241,03c2871e18d4fb503ead90461da747b40df5e28da0fd3e067f3731f1a28da60ddf
VIA_BRIDGE_BRIDGE_ADDRESS=bcrt1pfk264lnycy2v48h3we2jajyg7kyuvha9yfkd4qmxfrgywz3meyhqhdhmj8
```

### ViaBtcClientConfig

```rust
// core/lib/config/src/configs/via_btc_client.rs
#[derive(Debug, Deserialize, Default, Serialize, Clone, PartialEq)]
pub struct ViaBtcClientConfig {
    /// Name of the used Bitcoin network
    pub network: String,
    /// External fee APIs
    pub external_apis: Vec<String>,
    /// Fee strategies
    pub fee_strategies: Vec<String>,
    /// Use the RPC node as main fee rate provider.
    pub use_rpc_for_fee_rate: Option<bool>,
}

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

```toml
# etc/env/base/via_btc_client.toml
[via_btc_client]
# 'btc_rpc_user' is defined in the `via_private.toml`
# 'btc_rpc_password' is defined in the `via_private.toml`

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
```

Env prefix: `VIA_BTC_CLIENT_`. Note that `rpc_url`, `rpc_user`, and `rpc_password` under the same prefix are consumed by `ViaL1Secrets` (see [Secrets Configuration](#secrets-configuration)). On non-regtest networks, `rpc_url()` appends `wallet/{wallet}` to the base RPC URL so UTXOs are fetched from a specific Bitcoin Core wallet.

### ViaBtcWatchConfig

```rust
// core/lib/config/src/configs/via_btc_watch.rs
/// Total L1 blocks to process at a time.
pub const L1_BLOCKS_CHUNK: u32 = 50;

/// Configuration for the Bitcoin watch crate.
#[derive(Debug, Deserialize, Default, Serialize, Clone, PartialEq)]
pub struct ViaBtcWatchConfig {
    /// Service interval in milliseconds.
    pub poll_interval: u64,

    /// Minimum confirmation blocks for an inscription to be processed.
    pub block_confirmations: u64,

    /// The starting L1 block number from which indexing begins
    pub start_l1_block_number: u32,

    /// When set to true, the btc_watch starts indexing L1 blocks from the "start_l1_block_number".
    pub restart_indexing: bool,
}
```

```toml
# etc/env/base/via_btc_watch.toml
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
restart_indexing=false
```

Env prefix: `VIA_BTC_WATCH_`. The `btc_blocks_lag` key exists in the TOML but has no counterpart field in `ViaBtcWatchConfig`; it is ignored by the struct loader.

### ViaBtcSenderConfig

```rust
// core/lib/config/src/configs/via_btc_sender.rs
const DEFAULT_DA_LAYER: &str = "celestia";

#[derive(Debug, Deserialize, Serialize, Clone, PartialEq)]
pub struct ViaBtcSenderConfig {
    /// Service interval in milliseconds.
    pub poll_interval: u64,

    // Number of blocks to commit at time, should be 'one'.
    pub max_aggregated_blocks_to_commit: i32,

    // Number of proofs to commit at time, should be 'one'.
    pub max_aggregated_proofs_to_commit: i32,

    // The max number of inscription in flight
    pub max_txs_in_flight: i64,

    /// Number of block confirmations required to mark the inscription request as confirmed.
    pub block_confirmations: u32,

    /// The identifier of the DA layer.
    pub da_identifier: Option<String>,

    /// The btc sender wallet address.
    pub wallet_address: String,

    /// The number of blocks to wait before considering an inscription stuck.
    pub stuck_inscription_block_number: Option<u32>,

    /// The required time (seconds) to wait before create a commit inscription.
    pub block_time_to_commit: Option<u32>,

    /// The required time (seconds) to wait before create a proof inscription.
    pub block_time_to_proof: Option<u32>,
}
```

```toml
# etc/env/base/via_btc_sender.toml
[via_btc_sender]
# The interval in milliseconds between btc sender cycles.
poll_interval = 5000
# The max aggregated commit batches to process in one inscription.
max_aggregated_blocks_to_commit = 1
# The max aggregated proof batches to process in one inscription.
max_aggregated_proofs_to_commit = 1
# The max number of inscriptions in flight.
max_txs_in_flight = 1
# The DA layer identifier.
da_identifier = "celestia"
# The number of L1 block to mark the inscription as finalized. 
block_confirmations = 0
# The number of blocks to wait before considering an inscription stuck.
stuck_inscription_block_number = 6
# The required time (seconds) to wait before create a commit inscription.
block_time_to_commit = 0
# The required time (seconds) to wait before create a proof inscription.
block_time_to_proof = 0
```

Env prefix: `VIA_BTC_SENDER_`. The `wallet_address` (and the corresponding `private_key`) come from `etc/env/base/via_private.toml` (see [Wallet Configuration](#wallet-configuration)).

### ViaVerifierConfig

```rust
// core/lib/config/src/configs/via_verifier.rs
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct ViaVerifierConfig {
    /// The verifier role.
    pub role: ViaNodeRole,

    /// Service interval in milliseconds.
    pub poll_interval: u64,

    /// Port to which the coordinator server is listening.
    pub coordinator_port: u16,

    /// The coordinator url.
    pub coordinator_http_url: String,

    /// Verifier Request Timeout (in seconds)
    pub verifier_request_timeout: u8,

    /// The verifier btc wallet address.
    pub wallet_address: String,

    /// The bridge address merkle root.
    pub bridge_address_merkle_root: Option<String>,

    /// The session timeout.
    pub session_timeout: u64,

    /// Transaction weight limit.
    pub max_tx_weight: Option<u64>,
}
```

The transaction weight limit defaults to Bitcoin's standardness ceiling minus a safety buffer:

```rust
// core/lib/config/src/configs/via_verifier.rs
    pub fn max_tx_weight(&self) -> u64 {
        // Reserve 20000 weight units below Bitcoin's standard limit as a safety buffer
        // to account for witness data variations, signature size differences, and
        // potential rounding errors during transaction construction, ensuring we stay
        // well within node acceptance limits
        self.max_tx_weight
            .unwrap_or((MAX_STANDARD_TX_WEIGHT - 20000).into())
    }
```

```toml
# etc/env/base/via_verifier.toml
[via_verifier]
# Interval between polling db for verification requests (in ms).
poll_interval = 10000
# Coordinator server port.
coordinator_port = 6060
# Coordinator server url.
coordinator_http_url = "http://0.0.0.0:6060"
# Verifier Request Timeout (in seconds)
verifier_request_timeout = 10
# The verifier_mode can be simple verifier or coordinator.
role = "Coordinator"
# The session timeout (seconds).
session_timeout = 30
# The transaction weight limit
max_tx_weight = 380000
# The bridge address merkle root.
bridge_address_merkle_root = "0e45879bc42970c9b60ddeafa369472405ab87293380e79c4af3dad99623c5ad"
```

Env prefix: `VIA_VERIFIER_`. The `role` value is a `ViaNodeRole` and is set to `"Verifier"` or `"Coordinator"` in the environment overlays (see [Wallet Configuration](#wallet-configuration)).

### ViaCelestiaConfig

```rust
// core/lib/config/src/configs/via_celestia.rs
#[derive(Debug, Clone, Deserialize, PartialEq, Eq)]
#[serde(rename_all = "lowercase")]
pub enum DaBackend {
    Celestia,
    Http,
}

#[derive(Debug, Deserialize, Serialize, Clone, Copy, PartialEq)]
pub enum ProofSendingMode {
    OnlyRealProofs,
    SkipEveryProof,
}

#[derive(Debug, Deserialize, Clone, PartialEq)]
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
    #[serde(default)]
    pub fallback_external_node_url: Option<String>,

    /// Whether to verify data consistency between Celestia and fallback
    #[serde(default)]
    pub verify_consistency: bool,
}
```

```toml
# etc/env/base/via_celestia.toml
[via_celestia_client]
# The DA backend "http", "celestia"
da_backend = "http"
# The da node API URL.
api_node_url = "http://0.0.0.0:3001"
# The da blob size limit.
blob_size_limit = 1973786
# The mode in which proofs are sent.
proof_sending_mode = "SkipEveryProof"
# Optional external node URL for fallback when Celestia data is not available
fallback_external_node_url = "http://0.0.0.0:3050"
# Whether to verify data consistency between Celestia and fallback
verify_consistency = false
```

Env prefix: `VIA_CELESTIA_CLIENT_`. The infra tooling additionally writes `VIA_CELESTIA_CLIENT_TRUSTED_BLOCK_HEIGHT` and `VIA_CELESTIA_CLIENT_TRUSTED_BLOCK_HASH` into the compiled env file (`infrastructure/via/src/config.ts`).

### ViaReorgDetectorConfig

```rust
// core/lib/config/src/configs/via_reorg_detector.rs
#[derive(Debug, Deserialize, Default, Serialize, Clone, PartialEq)]
pub struct ViaReorgDetectorConfig {
    /// Service interval in milliseconds.
    pub poll_interval_ms: u64,
}
```

```toml
# etc/env/base/via_reorg_detector.toml
[via_reorg_detector]
poll_interval_ms = 5000
```

Env prefix: `VIA_REORG_DETECTOR_`.

## Secrets Configuration

Secrets are grouped in `ViaSecrets`, which extends the inherited zkSync `Secrets`:

```rust
// core/lib/config/src/configs/via_secrets.rs
#[derive(Clone, Deserialize, PartialEq)]
pub struct ViaL1Secrets {
    /// URL of the Bitcoin node RPC.
    pub rpc_url: SensitiveUrl,

    /// Username for the Bitcoin node RPC.
    pub rpc_user: String,

    /// Password for the Bitcoin node RPC.
    pub rpc_password: String,
}

#[derive(Clone, Deserialize, PartialEq)]
pub struct ViaL2Secrets {
    /// URL of the Bitcoin node RPC.
    pub rpc_url: SensitiveUrl,
}

#[derive(Clone, Deserialize, PartialEq)]
pub struct ViaDASecrets {
    /// URL for the Celestia node RPC.
    pub api_node_url: SensitiveUrl,

    /// AUTH token for the Celestia node RPC.
    pub auth_token: String,
}

#[derive(Debug, Clone, PartialEq)]
pub struct ViaSecrets {
    pub base_secrets: Secrets,
    pub via_l1: Option<ViaL1Secrets>,
    pub via_l2: Option<ViaL2Secrets>,
    pub via_da: Option<ViaDASecrets>,
}
```

The same file also contains `Debug` implementations that redact `rpc_user`, `rpc_password`, and `auth_token` from logs.

The env loaders map each secrets struct onto an existing prefix:

```rust
// core/lib/env_config/src/via_btc_sender.rs
impl FromEnv for ViaL1Secrets {
    fn from_env() -> anyhow::Result<Self> {
        envy_load("via_l1_secrets", "VIA_BTC_CLIENT_")
    }
}

impl FromEnv for ViaL2Secrets {
    fn from_env() -> anyhow::Result<Self> {
        envy_load("via_l2_secrets", "VIA_L2_CLIENT_")
    }
}

impl FromEnv for ViaDASecrets {
    fn from_env() -> anyhow::Result<Self> {
        envy_load("via_da_secrets", "VIA_CELESTIA_CLIENT_")
    }
}
```

So the effective secret variables are `VIA_BTC_CLIENT_RPC_URL`, `VIA_BTC_CLIENT_RPC_USER`, `VIA_BTC_CLIENT_RPC_PASSWORD`, `VIA_L2_CLIENT_RPC_URL`, `VIA_CELESTIA_CLIENT_API_NODE_URL`, and `VIA_CELESTIA_CLIENT_AUTH_TOKEN`.

The development defaults live in `via_private.toml`:

```toml
# etc/env/base/via_private.toml
# Sensitive values which MUST be different for production
# Values provided here are valid for the development infrastructure only.

database_url = "postgres://postgres:notsecurepassword@localhost/via"
database_prover_url = "postgres://postgres:notsecurepassword@localhost/via_prover"
test_database_url = "postgres://postgres:notsecurepassword@localhost:5433/via_test"
test_database_prover_url = "postgres://postgres:notsecurepassword@localhost:5433/via_prover_test"

[via_btc_sender]
# The btc sender private key.
private_key = "cVZduZu265sWeAqFYygoDEE1FZ7wV9rpW5qdqjRkUehjaUMWLT1R"
wallet_address = "bcrt1qx2lk0unukm80qmepjp49hwf9z6xnz0s73k9j56"

[via_celestia_client]
# The celestia auth token.
auth_token = ""

[via_btc_client]
# The btc client user.
rpc_user = "rpcuser"
# The btc client password.
rpc_password = "rpcpassword"

[via_l2_client]
# The l1 client user.
rpc_url = "http://0.0.0.0:3050"
```

The top-level `database_url` and related keys compile to `DATABASE_URL`, `DATABASE_PROVER_URL`, `TEST_DATABASE_URL`, and `TEST_DATABASE_PROVER_URL` (no `VIA_` prefix).

`via_server` can alternatively load secrets and wallets from YAML files via `--secrets-path` and `--wallets-path`; when those flags are absent it falls back to the environment:

```rust
// core/bin/via_server/src/main.rs
    let secrets: ViaSecrets = match opt.secrets_path {
        Some(path) => {
            read_yaml_repr::<zksync_protobuf_config::proto::via_secrets::ViaSecrets>(&path)
                .context("failed decoding secrets YAML config")?
        }
        None => ViaSecrets {
            base_secrets: Secrets {
                consensus: None,
```

## Wallet Configuration

Bitcoin wallets are represented by `ViaWallet` and grouped in `ViaWallets`:

```rust
// core/lib/config/src/configs/via_wallets.rs
#[derive(Default, Clone, PartialEq)]
pub struct ViaWallet {
    pub address: String,
    pub private_key: String,
}

#[derive(Default, Debug, Clone, PartialEq)]
pub struct ViaWallets {
    pub state_keeper: Option<StateKeeper>,
    pub token_multiplier_setter: Option<TokenMultiplierSetter>,

    /// Via wallets
    pub btc_sender: Option<ViaWallet>,
    pub vote_operator: Option<ViaWallet>,
}
```

Wallets are loaded from four dedicated environment variables:

```rust
// core/lib/env_config/src/via_wallets.rs
impl FromEnv for ViaWallets {
    fn from_env() -> anyhow::Result<Self> {
        let wallets = Wallets::from_env()?;

        let btc_sender_address = pk_from_env(
            "VIA_BTC_SENDER_WALLET_ADDRESS",
            "Malformed operator address",
        )?;
        let btc_sender_pk = pk_from_env("VIA_BTC_SENDER_PRIVATE_KEY", "Malformed operator pk")?;

        let verifier_address =
            pk_from_env("VIA_VERIFIER_WALLET_ADDRESS", "Malformed verifier address")?;
        let verifier_pk = pk_from_env("VIA_VERIFIER_PRIVATE_KEY", "Malformed verifier pk")?;

        Ok(Self {
            state_keeper: wallets.state_keeper,
            token_multiplier_setter: wallets.token_multiplier_setter,
            btc_sender: Some(ViaWallet {
                address: btc_sender_address.unwrap_or_default(),
                private_key: btc_sender_pk.clone().unwrap_or_default(),
            }),
            vote_operator: Some(ViaWallet {
                address: verifier_address.unwrap_or_default(),
                private_key: verifier_pk.unwrap_or_default(),
            }),
        })
    }
}
```

The verifier and coordinator environment overlays set the wallets per role:

```toml
# etc/env/configs/via_verifier.toml
__imports__ = ["base", "l2-inits/via_verifier.init.env"]

[via_verifier]
# The verifier role.
role = "Verifier"
# The wallet used in musig2 sessions.
private_key = "cQ4UHjdsGWFMcQ8zXcaSr7m4Kxq9x7g9EKqguTaFH7fA34mZAnqW"
wallet_address = "bcrt1qk8mkhrmgtq24nylzyzejznfzws6d98g4kmuuh4"

[via_btc_sender]
# The wallet used in the btc sender.
private_key = "cQ4UHjdsGWFMcQ8zXcaSr7m4Kxq9x7g9EKqguTaFH7fA34mZAnqW"
wallet_address = "bcrt1qk8mkhrmgtq24nylzyzejznfzws6d98g4kmuuh4"
```

```toml
# etc/env/configs/via_coordinator.toml
__imports__ = ["base", "l2-inits/via_coordinator.init.env"]

[via_verifier]
# The verifier role.
role = "Coordinator"
# The wallet used in musig2 sessions.
private_key = "cRaUbRSn8P8cXUcg6cMZ7oTZ1wbDjktYTsbdGw62tuqqD9ttQWMm"
wallet_address = "bcrt1qw2mvkvm6alfhe86yf328kgvr7mupdx4vln7kpv"

[via_btc_sender]
# The wallet used in the btc sender.
private_key = "cRaUbRSn8P8cXUcg6cMZ7oTZ1wbDjktYTsbdGw62tuqqD9ttQWMm"
wallet_address = "bcrt1qw2mvkvm6alfhe86yf328kgvr7mupdx4vln7kpv"
```

After compilation these become `VIA_VERIFIER_ROLE`, `VIA_VERIFIER_PRIVATE_KEY`, `VIA_VERIFIER_WALLET_ADDRESS`, `VIA_BTC_SENDER_PRIVATE_KEY`, and `VIA_BTC_SENDER_WALLET_ADDRESS`.

## Fee Rate Configuration

Bitcoin fee rate sourcing is configured entirely through `ViaBtcClientConfig` (see [ViaBtcClientConfig](#viabtcclientconfig)):

- `use_rpc_for_fee_rate`: when `true` (the default when unset), the Bitcoin RPC node is the main fee rate provider.
- `external_apis`: list of external fee estimation endpoints, e.g. `https://mempool.space/testnet/api/v1/fees/recommended`.
- `fee_strategies`: which fields of the external API response to use, e.g. `fastestFee`.

There are no per-network maximum fee rate caps in the configuration; keys such as `mainnet_max_fee_rate` do not exist in the codebase.

## Server Component Selection

`via_server` supports selective component execution through the `--components` CLI flag:

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
```

The valid component names and their mapping:

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

Example usage:

```bash
# Run everything (this is also the default component set)
via_server --components api,btc,tree,tree_api,state_keeper,housekeeper,proof_data_handler,commitment_generator,celestia,da_dispatcher,vm_runner_protective_reads,vm_runner_bwip

# Run only the Bitcoin-facing components
via_server --components btc,celestia
```

There is no environment variable for component selection; the only mechanism is the `--components` flag. Component names like `sequencer`, `verifier`, or `prover` are not valid values (the verifier runs as a separate `via_verifier` binary).

## Bridge Configuration

Bridge parameters are held in `ViaBridgeConfig` (`verifiers_pub_keys` and `bridge_address`, see [ViaBridgeConfig](#viabridgeconfig)) with the `VIA_BRIDGE_` env prefix. Keys such as `coordinator_pub_key`, `required_signers`, or `zk_agreement_threshold` do not exist in the configuration.

### RPC API surface

The `via` JSON-RPC namespace currently exposes two methods; `via_getBridgeAddress` exists only as commented-out code:

```rust
// core/lib/web3_decl/src/namespaces/via.rs
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

### Deposit CLI

The deposit helper requires an explicit bridge address:

```typescript
// infrastructure/via/src/token.ts
command
    .command('deposit')
    .requiredOption('--amount <amount>', 'amount of BTC to deposit', parseFloat)
    .requiredOption('--receiver-l2-address <receiverL2Address>', 'receiver l2 address')
    .requiredOption('--bridge-address <bridgeAddress>', 'The bridge address')
    .option('--sender-private-key <senderPrivateKey>', 'sender private key', DEFAULT_DEPOSITOR_PRIVATE_KEY)
    .option('--network <network>', 'network', DEFAULT_NETWORK)
    .option('--l1-rpc-url <l1RcpUrl>', 'RPC URL', DEFAULT_L1_RPC_URL)
    .option('--l2-rpc-url <l2RcpUrl>', 'RPC URL', DEFAULT_L2_RPC_URL)
    .option('--rpc-username <rcpUsername>', 'RPC username', DEFAULT_RPC_USERNAME)
    .option('--rpc-password <rpcPassword>', 'RPC password', DEFAULT_RPC_PASSWORD)
```

A `deposit-with-op-return` subcommand with the same required options also exists, as well as a `withdraw` subcommand (`--amount`, `--receiver-l1-address`, `--user-private-key`).

## Docker Deployment Configuration

### docker-compose.yml

Basic development services:

- `reth`: Ethereum node for development
- `postgres`: PostgreSQL database
- `zk`: development environment container

### docker-compose-via.yml

Via-specific services:

- `bitcoind`: Bitcoin Core regtest node (`lightninglabs/bitcoin-core:27`)
- `bitcoin-cli`: setup and block generation loop against `bitcoind`
- `bitcoind2` / `bitcoin-cli2`: second node pair, only active in the `reorg` profile
- `postgres`: PostgreSQL 14 (`postgres:14`, `max_connections=200`, port 5432)
- `celestia-node-init` / `celestia-node`: Celestia light node (`ghcr.io/celestiaorg/celestia-node:v0.29.3-mocha`)
- `da-proxy`: DA proxy built from `./via-core-ext`, only active in the `da-proxy` profile (port 3001)

The Bitcoin node flags:

```yaml
# docker-compose-via.yml
x-bitcoind-common: &bitcoind-common
  image: "lightninglabs/bitcoin-core:27"
  command:
    - -regtest
    - -server
    - -rpcbind=0.0.0.0
    - -rpcallowip=0.0.0.0/0
    - -rpcuser=rpcuser
    - -rpcpassword=rpcpassword
    - -fallbackfee=0.0002
    - -txindex
    - -printtoconsole
    - -dustrelayfee=0.0
    - -minrelaytxfee=0
    - -acceptnonstdtxn=1
```

The Celestia init container fetches a trusted height and hash from the Mocha RPC endpoint and patches the light node config; the node then starts against the `mocha` network:

```yaml
# docker-compose-via.yml
  celestia-node-init:
    image: &celestia-image "ghcr.io/celestiaorg/celestia-node:v0.29.3-mocha"
    entrypoint:
      - /bin/bash
      - -c
      - |
        cp -r /keys /home/celestia
        celestia light init

        # https://docs.celestia.org/how-to-guides/celestia-node-trusted-hash
        read -r TRUSTED_HEIGHT TRUSTED_HASH <<<"$(curl -s https://rpc-mocha.pops.one/header | jq -r '.result.header | "\(.height) \(.last_block_id.hash)"')" && export TRUSTED_HEIGHT TRUSTED_HASH
        sed -i "s/SyncFromHeight = .*/SyncFromHeight = $$TRUSTED_HEIGHT/" ~/config.toml
        sed -i "s/SyncFromHash = .*/SyncFromHash = \"$$TRUSTED_HASH\"/" ~/config.toml

  celestia-node:
    image: *celestia-image
    command: celestia light start --core.ip=rpc-mocha.pops.one --p2p.network=mocha --keyring.backend=test --keyring.keyname=via
    ports:
      - '26658:26658'
```

## External Node Configuration

The External Node (EN) is a read-only replica of the main node serving JSON-RPC locally. Configuration comes from three places:

- Baseline template: `etc/env/configs/ext-node.toml`
- Via operator template: `etc/env/configs/via_ext_node.toml` (self-contained; it intentionally does not import the `base` layer)
- Interactive setup: `infrastructure/via/src/setup_en.ts`

The core `[en]` settings of the Via template:

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
```

The template also contains `[en.consensus]` (`config_path = "etc/env/en_consensus_config.yaml"`, `secrets_path = "etc/env/en_consensus_secrets.yaml"`), `[en.database]`, `[en.snapshots.object_store]`, `[en.main_node]` (`url = "http://127.0.0.1:3050"`), `[en.gateway]` (`url = "http://127.0.0.1:3052"`), the full Via config sections (`[via_bridge]`, `[via_genesis]`, `[via_btc_client]`, `[via_btc_watch]`, `[via_reorg_detector]`), and a `[rust]` section with log directives and backtrace settings. The baseline `ext-node.toml` differs in a few defaults: `pruning_data_retention_hours = 1`, `snapshots_recovery_enabled = true`, `l1_chain_id = 9`, RocksDB paths under `./db/ext-node/`, and `api_namespaces` without `"via"`.

### Environment variables

The EN binary reads its configuration from `EN_`-prefixed variables:

```rust
// core/bin/via_external_node/src/config/mod.rs
        let mut result: OptionalENConfig = envy::prefixed("EN_")
```

Object store, experimental, and component-specific settings use nested prefixes `EN_SNAPSHOTS_OBJECT_STORE_`, `EN_EXPERIMENTAL_`, `EN_API_`, and `EN_TREE_` (same file). Notable optional fields in `OptionalENConfig` include `pruning_data_retention_sec` (env: `EN_PRUNING_DATA_RETENTION_SEC`, default 7 days), `extended_rpc_tracing` (env: `EN_EXTENDED_RPC_TRACING`), and `main_node_rate_limit_rps` (env: `EN_MAIN_NODE_RATE_LIMIT_RPS`). Note that the TOML template key is `pruning_data_retention_hours`, which compiles to `EN_PRUNING_DATA_RETENTION_HOURS`; the binary's own field is expressed in seconds.

### Setup flow

`setup_en.ts` interactively edits `etc/env/configs/via_ext_node.toml` (via `changeConfigKey`/`removeConfigKey`), compiles the env, sets up the database, and injects the bootstrap txids (`updateBootstrapTxidsEnv`). The data retention prompt writes `pruning_data_retention_hours` in hours, or removes the key for "Forever":

```typescript
// infrastructure/via/src/setup_en.ts
async function selectDataRetentionDurationHours(): Promise<number | null> {
    const question = {
        type: 'select',
        name: 'retention',
        message: 'Select how long do you want to keep newest transactions data',
        choices: [
            { name: DataRetentionDuration.Hour, message: 'Hour', value: 1 },
            { name: DataRetentionDuration.Day, message: 'Day', value: 24 },
            { name: DataRetentionDuration.Week, message: 'Week', value: 24 * 7 },
            { name: DataRetentionDuration.Month, message: 'Month', value: 24 * 31 },
            { name: DataRetentionDuration.Year, message: 'Year', value: 24 * 366 },
            { name: DataRetentionDuration.Forever, message: 'Forever', value: null }
        ]
    };
```

### CLI flags

`core/bin/via_external_node/src/main.rs` accepts `--enable-consensus` plus optional file-based configuration: `--config-path` (requires `--external-node-config-path`), `--secrets-path`, and `--consensus-path` (required when both `--config-path` and `--enable-consensus` are set).

## Conclusion

Via's configuration is TOML-templated but env-delivered: `etc/env/base/*.toml` provides defaults, `etc/env/configs/<env>.toml` overlays them per environment, and the `via` CLI compiles everything into flat `SECTION_KEY=value` env files that the Rust binaries consume through `envy` prefixes such as `VIA_BTC_CLIENT_`, `VIA_VERIFIER_`, and `VIA_BRIDGE_`. The main server is strictly env-driven (with optional YAML for secrets and wallets), component selection happens through the `--components` flag, and the external node runs from its own self-contained `via_ext_node` template with `EN_`-prefixed variables. When in doubt, the authoritative definitions are the structs in `core/lib/config/src/configs/via_*.rs` and the loaders in `core/lib/env_config/src/`.
