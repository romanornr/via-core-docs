# Genesis and Bootstrapping Process in Via L2 Bitcoin ZK-Rollup

This document details the genesis and bootstrapping process for the Via L2 Bitcoin ZK-Rollup system, focusing on how the initial state is defined, generated, and committed to the network.

## Table of Contents

1. [Overview](#overview)
2. [Primary Code Modules](#primary-code-modules)
3. [Genesis Configuration](#genesis-configuration)
4. [Genesis Process](#genesis-process)
5. [System Bootstrapping](#system-bootstrapping)
6. [Interactions with Other Components](#interactions-with-other-components)
7. [Configuration Files](#configuration-files)
8. [Command-Line Tools](#command-line-tools)

## Overview

The genesis process in Via L2 is responsible for initializing the rollup's state, including deploying system contracts, setting up the initial Merkle tree, and creating the genesis block. This process is critical for establishing the foundation of the L2 network and ensuring that all nodes start with the same initial state.

The bootstrapping process, which follows genesis, involves committing the initial state to the Bitcoin L1 blockchain through a special inscription and setting up the necessary infrastructure for the network to begin processing transactions.

## Primary Code Modules

The genesis and bootstrapping functionality is primarily implemented in the following modules:

1. **`core/bin/genesis_generator/`**: A tool for generating and updating the genesis configuration
   - `main.rs`: The main entry point for the genesis generator tool

2. **`core/node/genesis/`**: The core library for genesis-related functionality
   - `lib.rs`: Contains the main genesis logic
   - `utils.rs`: Helper functions for the genesis process

3. **`infrastructure/zk/src/`**: Scripts for initializing and running the system
   - `init.ts`: Contains initialization logic
   - `server.ts`: Contains functions for generating genesis data

## Genesis Configuration

The genesis configuration is defined in YAML files and contains parameters that control the initial state of the rollup:

### Main Genesis Configuration (`etc/env/file_based/genesis.yaml`):

```yaml
genesis_root: 0x5d1b8d6f250d89b77b66a918e3e0f60f64ca959aee0c5f1031575491fd00bfa6
genesis_rollup_leaf_index: 54
genesis_batch_commitment: 0xe81e1a4727269fe1ef3e2f8c3f5cfb9aab7c073722c278331b7e017033c13f8f
genesis_protocol_semantic_version: '0.26.0'
# deprecated
genesis_protocol_version: 26
default_aa_hash: 0x010005630848b5537f934eea6bd8c61c50648a162b90c82f316454f4109462b1
bootloader_hash: 0x010008e79c154523aa30981e598b73c4a33c304bef9c82bae7d2ca4d21daedc7
l1_chain_id: 9
l2_chain_id: 270
fee_account: '0x0000000000000000000000000000000000000001'
prover:
  recursion_scheduler_level_vk_hash: 0x14f97b81e54b35fe673d8708cc1a19e1ea5b5e348e12d31e39824ed4f42bbca2
  dummy_verifier: true
l1_batch_commit_data_generator_mode: Rollup
```

### Chain-Specific Configuration (`chains/era/ZkStack.yaml`):

```yaml
id: 1
name: era
chain_id: 271
prover_version: NoProofs
configs: ./chains/era/configs/
rocks_db_path: ./chains/era/db/
external_node_config_path: ./chains/era/configs/external_node
l1_batch_commit_data_generator_mode: Rollup
base_token:
  address: '0x0000000000000000000000000000000000000001'
  nominator: 1
  denominator: 1
wallet_creation: Localhost
```

### Global Configuration (`ZkStack.yaml`):

```yaml
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

The genesis process involves several key steps:

### 1. Loading Genesis Parameters

The `GenesisParams` struct in `core/node/genesis/src/lib.rs` encapsulates the parameters needed for genesis:

```rust
pub struct GenesisParams {
    base_system_contracts: BaseSystemContracts,
    system_contracts: Vec<DeployedContract>,
    config: GenesisConfig,
}
```

These parameters are loaded from the genesis configuration file:

```rust
pub fn load_genesis_params(config: GenesisConfig) -> Result<GenesisParams, GenesisError> {
    let base_system_contracts = BaseSystemContracts::load_from_disk();
    let system_contracts = get_system_smart_contracts();
    Self::from_genesis_config(config, base_system_contracts, system_contracts)
}
```

### 2. Creating the Genesis L1 Batch

The `create_genesis_l1_batch` function in `core/node/genesis/src/lib.rs` creates the genesis L1 batch:

```rust
pub async fn create_genesis_l1_batch(
    storage: &mut Connection<'_, Core>,
    protocol_version: ProtocolSemanticVersion,
    base_system_contracts: &BaseSystemContracts,
    system_contracts: &[DeployedContract],
    l1_verifier_config: L1VerifierConfig,
) -> Result<(), GenesisError>
```

This function:
- Creates a protocol version record
- Creates the genesis L1 batch header
- Creates the genesis L2 block header
- Inserts the base system contracts (bootloader and default account abstraction)
- Inserts the system contracts
- Adds the BTC token

### 3. Building the Initial Merkle Tree

The `insert_genesis_batch` function in `core/node/genesis/src/lib.rs` builds the initial Merkle tree:

```rust
pub async fn insert_genesis_batch(
    storage: &mut Connection<'_, Core>,
    genesis_params: &GenesisParams,
) -> Result<GenesisBatchParams, GenesisError>
```

This function:
- Creates the genesis L1 batch
- Extracts storage logs from the system contracts
- Processes these logs to build the initial Merkle tree
- Calculates the genesis root hash and commitment
- Saves the genesis metadata to the database

### 4. Generating Genesis Data via Server

The `genesisFromSources` function in `infrastructure/zk/src/server.ts` triggers the genesis generation process:

```typescript
export async function genesisFromSources() {
    // Note that that all the chains have the same chainId at genesis. It will be changed
    // via an upgrade transaction during the registration of the chain.
    await create_genesis('cargo run --bin zksync_server --release -- --genesis');
}
```

This function runs the `zksync_server` binary with the `--genesis` flag, which executes the genesis process and logs the output.

## System Bootstrapping

After the genesis process, the system needs to be bootstrapped on the Bitcoin L1 blockchain. This is done through a special inscription type called `SystemBootstrapping`.

### SystemBootstrapping Inscription

The `SystemBootstrapping` inscription type is defined in the Via L2 custom inscription protocol:

```rust
pub struct SystemBootstrappingInput {
    pub start_block_height: u32,
    pub verifier_p2wpkh_addresses: Vec<BitcoinAddress<NetworkUnchecked>>,
    pub bridge_musig2_address: BitcoinAddress<NetworkUnchecked>,
    pub governance_address: BitcoinAddress<NetworkUnchecked>,
    pub bootloader_hash: H256,
    pub abstract_account_hash: H256,
}
```

This inscription contains:
- The starting block height for the L2 network
- The addresses of the verifiers
- The bridge MuSig2 address for cross-chain operations
- The governance address for system upgrades and administrative operations
- The bootloader hash
- The abstract account hash

The inscription is created using the commit-reveal pattern:
1. A commit transaction creates a Taproot output that commits to the inscription script
2. A reveal transaction spends the Taproot output, revealing the inscription data

### Bootstrapping Process

The bootstrapping process involves:

1. **Creating the SystemBootstrapping Inscription**: This is done by the network operators when initializing the network.

2. **Indexing the Inscription**: The `BitcoinInscriptionIndexer` in `core/lib/via_btc_client/indexer/mod.rs` monitors the Bitcoin blockchain for inscriptions, including the `SystemBootstrapping` inscription.

3. **Processing the Inscription**: When the `SystemBootstrapping` inscription is detected, it is parsed and processed by the `MessageParser` in `core/lib/via_btc_client/indexer/parser.rs`.

4. **Initializing the Network**: The data from the `SystemBootstrapping` inscription is used to initialize the network, including setting up the verifiers and the bridge.

## Interactions with Other Components

The genesis and bootstrapping process interacts with several other components of the Via L2 system:

### 1. System Contracts

The genesis process deploys the system contracts, which are essential for the functioning of the L2 network. These contracts are loaded from disk and their bytecode is stored in the database.

### 2. State Management

The genesis process initializes the state Merkle tree, which is a fundamental component of the state management system. The initial state includes the storage of the system contracts.

### 3. L1 Watcher

The L1 Watcher component monitors the Bitcoin blockchain for inscriptions, including the `SystemBootstrapping` inscription. It is responsible for detecting and processing this inscription to initialize the network.

### 4. Verifier Network

The `SystemBootstrapping` inscription includes the addresses of the verifiers, which are used to initialize the verifier network. These verifiers are responsible for validating L2 blocks and committing them to the L1 blockchain.

### 5. Governance

The `SystemBootstrapping` inscription includes the governance address, which is used for system upgrades and administrative operations. This address is authorized to create and sign system upgrade inscriptions, which are used to update the protocol version, bootloader, default account, and system contracts.

## Configuration Files

Several configuration files are involved in the genesis and bootstrapping process:

### 1. Genesis Configuration (`etc/env/file_based/genesis.yaml`)

This file contains the parameters for the genesis process, including:
- The genesis root hash
- The genesis rollup leaf index
- The genesis batch commitment
- The protocol version
- The bootloader and default account abstraction hashes
- The L1 and L2 chain IDs
- The fee account address
- The prover configuration

### 2. Chain Configuration (`chains/era/ZkStack.yaml`)

This file contains the configuration for a specific chain, including:
- The chain ID
- The prover version
- The paths to the configuration and database directories
- The base token configuration

### 3. Global Configuration (`ZkStack.yaml`)

This file contains global configuration parameters, including:
- The L1 network
- The paths to the chains and configuration directories
- The default chain
- The ERA chain ID
- The prover version

## Command-Line Tools

Several command-line tools are available for the genesis and bootstrapping process:

### 1. Genesis Generator

The `genesis_generator` tool in `core/bin/genesis_generator/` is used to generate and update the genesis configuration:

```bash
cargo run --bin genesis_generator -- [--config-path <path>] [--check]
```

Options:
- `--config-path`: The path to the configuration file
- `--check`: Check if the genesis configuration is up to date

### 2. Server Genesis Command

The `server` command with the `--genesis` flag in `infrastructure/zk/src/server.ts` is used to generate the genesis data:

```bash
zk server --genesis
```

This command runs the `zksync_server` binary with the `--genesis` flag, which executes the genesis process and logs the output.

### 3. Initialization Commands

The `init` command in `infrastructure/zk/src/init.ts` provides several subcommands for initializing the system:

```bash
zk init [options]
zk init lightweight
zk init shared-bridge [options]
zk init hyper [options]
```

These commands perform various initialization tasks, including:
- Setting up the environment
- Deploying contracts
- Initializing the database
- Running the genesis process

## Conclusion

The genesis and bootstrapping process in Via L2 Bitcoin ZK-Rollup is a complex but well-structured procedure that initializes the L2 network and connects it to the Bitcoin L1 blockchain. It involves generating the initial state, deploying system contracts, building the Merkle tree, and committing the initial state to the L1 blockchain through a special inscription.

This process ensures that all nodes start with the same initial state and that the network is properly connected to the L1 blockchain, providing the security and verifiability that are essential for a Layer 2 solution.

## Bootstrapping and Upgrade Relationship

The bootstrapping process establishes the foundation for future system upgrades:

1. **Governance Address**: The governance address specified in the `SystemBootstrapping` inscription is authorized to create and sign system upgrade inscriptions.

2. **Initial Protocol Version**: The initial protocol version is set during genesis and recorded in the `SystemBootstrapping` inscription.

3. **Base System Contracts**: The bootloader and default account abstraction hashes are included in the `SystemBootstrapping` inscription, establishing the initial versions of these critical components.

4. **Upgrade Path**: After bootstrapping, the system can be upgraded through `SystemContractUpgrade` inscriptions, which must be signed by the governance address.

This relationship ensures a secure and controlled upgrade path from the initial bootstrapped state to future protocol versions.