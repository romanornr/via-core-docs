# Via L2 Bitcoin ZK-Rollup: Configuration

This document provides an overview of the key configuration files, environment variables, and parameters that significantly affect component behavior and deployment in the Via L2 Bitcoin ZK-Rollup system.

## Table of Contents

1. [Configuration Architecture](#configuration-architecture)
2. [Global Configuration Files](#global-configuration-files)
3. [Component-Specific Configuration](#component-specific-configuration)
4. [Environment Variables](#environment-variables)
5. [Docker Deployment Configuration](#docker-deployment-configuration)
6. [Network Configuration](#network-configuration)

## Configuration Architecture

The Via L2 system uses a hierarchical configuration approach:

1. **Global Configuration**: System-wide settings defined in `ZkStack.yaml` and other top-level files
2. **Chain-Specific Configuration**: Settings for specific chains in `chains/*/ZkStack.yaml`
3. **Component Configuration**: Settings for individual components in `configs/` and `etc/`
4. **Environment Variables**: Runtime configuration through environment variables
5. **Docker Configuration**: Deployment settings in Docker Compose files

Configuration is loaded using a combination of file-based configuration and environment variables, with environment variables taking precedence over file-based configuration.

## Global Configuration Files

### ZkStack.yaml

The `ZkStack.yaml` file defines global settings for the entire system:

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

Key parameters:
- `name`: Name of the ZkStack instance
- `l1_network`: The L1 network to connect to (e.g., Localhost, Mainnet)
- `chains`: Directory containing chain-specific configurations
- `config`: Directory containing component configurations
- `default_chain`: The default chain to use
- `era_chain_id`: The chain ID for the ERA chain
- `prover_version`: The version of the prover to use

### Genesis Configuration (etc/env/file_based/genesis.yaml)

The genesis configuration defines the initial state of the rollup:

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

Key parameters:
- `genesis_root`: The root hash of the initial state
- `genesis_protocol_semantic_version`: The semantic version of the protocol (e.g., '0.26.0')
- `bootloader_hash`: Hash of the bootloader contract
- `default_aa_hash`: Hash of the default account abstraction contract
- `l1_chain_id`: The chain ID of the L1 network
- `l2_chain_id`: The chain ID of the L2 network
- `fee_account`: The account that receives fees
- `prover.recursion_scheduler_level_vk_hash`: Hash of the verification key for the recursion scheduler

## Component-Specific Configuration

### Chain Configuration (chains/era/ZkStack.yaml)

Each chain has its own configuration file:

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

Key parameters:
- `id`: Unique identifier for the chain
- `name`: Name of the chain
- `chain_id`: The chain ID for this chain
- `prover_version`: The version of the prover to use
- `configs`: Directory containing chain-specific component configurations
- `rocks_db_path`: Path to the RocksDB database for this chain

### Sequencer Configuration

The Sequencer is configured through files in `configs/` and environment variables:

Key parameters:
- `state_keeper_config`: Configuration for the State Keeper
- `mempool_config`: Configuration for the Mempool
- `batch_executor_config`: Configuration for the Batch Executor
- `seal_criteria_config`: Configuration for when to seal blocks and batches
- `btc_sender_config`: Configuration for sending data to Bitcoin

### Prover Configuration

The Prover is configured through files in `configs/` and environment variables:

Key parameters:
- `prover_gateway_config`: Configuration for the Prover Gateway
- `witness_generator_config`: Configuration for the Witness Generator
- `circuit_prover_config`: Configuration for the Circuit Prover
- `proof_compressor_config`: Configuration for the Proof Compressor

### Verifier Configuration

The Verifier is configured through files in `configs/` and environment variables:

Key parameters:
- `verifier_mode`: The mode of the Verifier (COORDINATOR or VERIFIER)
- `required_signers`: Number of signers required for MuSig2 signatures
- `verifiers_pub_keys`: Public keys of all Verifiers
- `verifier_request_timeout`: Timeout for Verifier requests

### Celestia Integration Configuration

The Celestia integration is configured through the `ViaCelestiaConfig` struct:

Key parameters:
- `api_node_url`: URL of the Celestia node API
- `auth_token`: Authentication token for the Celestia node
- `blob_size_limit`: Maximum size of a blob that can be dispatched

## Environment Variables

The Via L2 system uses environment variables for runtime configuration. Here are some of the key environment variables:

### General Configuration

- `VIA_GENERAL_CONFIG`: Path to the general configuration file
- `VIA_CHAIN_ID`: The chain ID to use
- `VIA_L1_NETWORK`: The L1 network to connect to
- `VIA_API_PORT`: Port for the API server
- `VIA_PROMETHEUS_PORT`: Port for Prometheus metrics

### Database Configuration

- `VIA_DATABASE_URL`: URL for the PostgreSQL database
- `VIA_DATABASE_POOL_SIZE`: Size of the database connection pool
- `VIA_ROCKS_DB_PATH`: Path to the RocksDB database

### Bitcoin Configuration

- `VIA_BTC_CLIENT_URL`: URL of the Bitcoin node
- `VIA_BTC_CLIENT_USER`: Username for Bitcoin node authentication
- `VIA_BTC_CLIENT_PASSWORD`: Password for Bitcoin node authentication
- `VIA_BTC_NETWORK`: Bitcoin network to use (mainnet, testnet, regtest)
- `VIA_BTC_BRIDGE_ADDRESS`: Bitcoin address for the bridge (MuSig2 multisig address)

### Celestia Configuration

- `VIA_CELESTIA_CLIENT_URL`: URL of the Celestia node
- `VIA_CELESTIA_CLIENT_AUTH_TOKEN`: Authentication token for the Celestia node
- `VIA_CELESTIA_CLIENT_TRUSTED_BLOCK_HASH`: Trusted block hash for Celestia light node

### Verifier Configuration

- `VIA_VERIFIER_MODE`: Mode of the Verifier (COORDINATOR or VERIFIER)
- `VIA_VERIFIER_PRIVATE_KEY`: Private key for the Verifier
- `VIA_VERIFIER_COORDINATOR_URL`: URL of the Coordinator (for regular Verifiers)
- `VIA_VERIFIER_REQUIRED_SIGNERS`: Number of signers required for MuSig2 signatures
- `VIA_VERIFIER_PUBLIC_KEYS`: Comma-separated list of Verifier public keys
- `VIA_VERIFIER_PROTOCOL_VERSION`: Current protocol version for the verifier

### Prover Configuration

- `VIA_PROVER_GATEWAY_URL`: URL of the Prover Gateway
- `VIA_PROVER_WITNESS_GENERATOR_URL`: URL of the Witness Generator
- `VIA_PROVER_CIRCUIT_PROVER_URL`: URL of the Circuit Prover
- `VIA_PROVER_PROOF_COMPRESSOR_URL`: URL of the Proof Compressor

## Docker Deployment Configuration

The Via L2 system uses Docker Compose for deployment. The main Docker Compose files are:

### docker-compose.yml

Basic services for development:

```yaml
version: '3.2'
services:
  reth:
    # Ethereum node for development
  postgres:
    # PostgreSQL database
  zk:
    # Development environment
```

### docker-compose-via.yml

Via-specific services:

```yaml
version: '3.8'
services:
  bitcoind:
    # Bitcoin node
  bitcoin-cli:
    # Bitcoin CLI for setup
  postgres:
    # PostgreSQL database
  celestia-node:
    # Celestia light node
```

Key configuration parameters:
- Bitcoin node configuration (regtest mode, RPC credentials)
- PostgreSQL configuration (password, connection limits)
- Celestia node configuration (network, trusted block hash)

## Network Configuration

### Bitcoin Network Configuration

The Via L2 system can connect to different Bitcoin networks:

- `mainnet`: Bitcoin mainnet
- `testnet`: Bitcoin testnet
- `regtest`: Bitcoin regtest (local development)

Configuration parameters:
- `rpcbind`: Bitcoin RPC bind address
- `rpcallowip`: Bitcoin RPC allowed IP addresses
- `rpcuser`: Bitcoin RPC username
- `rpcpassword`: Bitcoin RPC password
- `txindex`: Enable transaction indexing

### Celestia Network Configuration

The Via L2 system connects to the Celestia network:

- `mocha`: Celestia testnet
- `mainnet`: Celestia mainnet

Configuration parameters:
- `headers.trusted-hash`: Trusted block hash for light node
- `core.ip`: IP address of the Celestia full node
- `p2p.network`: Celestia network to connect to
- `keyring.backend`: Keyring backend to use
- `keyring.keyname`: Name of the key to use

## Protocol Version Configuration

The Via L2 system manages protocol versions through configuration and database storage:

### Protocol Version Structure

Protocol versions follow semantic versioning:
- `major`: Incompatible API changes
- `minor`: Backward-compatible functionality additions
- `patch`: Backward-compatible bug fixes

Example: `0.26.0`

### Protocol Version Storage

Protocol versions are stored in the database:
- `protocol_versions` table stores all protocol versions
- Each version has an `executed` flag indicating whether it has been applied
- Each batch is associated with a specific protocol version

### Protocol Version Environment Variables

- `VIA_PROTOCOL_VERSION`: Current protocol version for the system
- `VIA_VERIFIER_PROTOCOL_VERSION`: Current protocol version for the verifier
- `VIA_SEQUENCER_PROTOCOL_VERSION`: Current protocol version for the sequencer

## Conclusion

The Via L2 Bitcoin ZK-Rollup system uses a comprehensive configuration system that allows for flexible deployment and operation. By understanding the key configuration files, environment variables, and parameters, operators can effectively deploy and manage the system in various environments. The protocol version management system ensures smooth upgrades and backward compatibility.