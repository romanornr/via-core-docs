# Via L2 Bitcoin ZK-Rollup: Configuration

This document provides an overview of the key configuration files, environment variables, and parameters that significantly affect component behavior and deployment in the Via L2 Bitcoin ZK-Rollup system.

**Recent Configuration Updates:**
- **BTC Sender Configuration Refactoring**: Migration from hardcoded constants to environment variables
- **L1 Indexer Configuration**: New configuration system for the dedicated indexer service
- **Enhanced Environment Variable Support**: Comprehensive environment-based configuration management
- **Database Configuration Expansion**: Support for multiple database connections including indexer database

## Table of Contents

1. [Configuration Architecture](#configuration-architecture)
2. [Global Configuration Files](#global-configuration-files)
3. [Component-Specific Configuration](#component-specific-configuration)
4. [Environment Variables](#environment-variables)
5. [BTC Sender Configuration](#btc-sender-configuration)
6. [L1 Indexer Configuration](#l1-indexer-configuration)
7. [Database Configuration](#database-configuration)
8. [Docker Deployment Configuration](#docker-deployment-configuration)
9. [Network Configuration](#network-configuration)
10. [Wallet Configuration](#wallet-configuration)
11. [Fee Management Configuration](#fee-management-configuration)
12. [Server Component Selection](#server-component-selection)

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
- `genesis_batch_commitment`: The batch commitment hash for genesis
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
- `zksync_network_id`: The ZkSync network identifier
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

### BTC Sender Configuration

The BTC Sender component handles Bitcoin transaction broadcasting and is configured through files and environment variables:

Key parameters:
- `wallet_address`: Bitcoin wallet address for the BTC sender component
- `btc_rpc_url`: Bitcoin RPC URL, supports wallet-specific paths
- `fee_strategy`: Fee rate management configuration

Example configuration in `etc/env/base/via_private.toml`:
```toml
wallet_address = "bc1qsenderaddress..."
```

The BTC RPC URL can include wallet-specific paths for enhanced wallet management:
```toml
btc_rpc_url = "http://localhost:8332/wallet/mywallet"
```

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
- `wallet_address`: Bitcoin wallet address for the verifier component
- `required_signers`: Number of signers required for MuSig2 signatures
- `verifiers_pub_keys`: Public keys of all Verifiers
- `verifier_request_timeout`: Timeout for Verifier requests

Example configuration in `etc/env/configs/via_verifier.toml`:
```toml
wallet_address = "bc1qverifieraddress..."
verifier_mode = "VERIFIER"
required_signers = 3
```

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
- `VIA_BTC_SENDER_WALLET_ADDRESS`: Wallet address for the BTC sender component

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
- `VIA_VERIFIER_WALLET_ADDRESS`: Wallet address for the verifier component

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

## Wallet Configuration

The Via L2 system supports wallet-specific configuration for Bitcoin operations across different components.

### Wallet Address Configuration

Both BTC sender and verifier components require wallet address configuration:

```toml
# etc/env/base/via_private.toml
wallet_address = "bc1qexampleaddress..."

# etc/env/configs/via_coordinator.toml
wallet_address = "bc1qcoordinatoraddress..."

# etc/env/configs/via_verifier.toml
wallet_address = "bc1qverifieraddress..."
```

### Environment Variables

Wallet addresses can be configured via environment variables:
- `VIA_BTC_SENDER_WALLET_ADDRESS`: Wallet address for BTC sender operations
- `VIA_VERIFIER_WALLET_ADDRESS`: Wallet address for verifier operations

### Wallet-Specific RPC URLs

Bitcoin RPC URLs support wallet-specific paths for enhanced UTXO management:

```toml
# Standard RPC configuration
btc_rpc_url = "http://localhost:8332"

# Wallet-specific RPC configuration
btc_rpc_url = "http://localhost:8332/wallet/mywallet"
```

This enables:
- Wallet-specific UTXO fetching
- Enhanced wallet management capabilities
- Multi-wallet Bitcoin node setups

## Fee Management Configuration

The system provides comprehensive fee rate management for Bitcoin transactions across different networks.

### Fee Rate Limits

Configure network-specific Bitcoin transaction fee rate caps in `etc/env/base/via_btc_client.toml`:

```toml
[fee_strategy]
mainnet_max_fee_rate = 100  # satoshis per vbyte
testnet_max_fee_rate = 50   # satoshis per vbyte
regtest_max_fee_rate = 10   # satoshis per vbyte
```

### Network-Specific Configuration

- **Mainnet**: Higher fee rate limits for production use (typically 100+ sat/vbyte)
- **Testnet**: Moderate fee rate limits for testing (typically 50 sat/vbyte)
- **Regtest**: Lower fee rate limits for local development (typically 10 sat/vbyte)

### Environment Variables

Fee rate limits can be overridden via environment variables:
```bash
export VIA_BTC_MAINNET_MAX_FEE_RATE=100
export VIA_BTC_TESTNET_MAX_FEE_RATE=50
export VIA_BTC_REGTEST_MAX_FEE_RATE=10
```

## Server Component Selection

The via_server supports selective component execution for optimized deployments.

### Component Selection

Use the `--components` CLI flag to run specific components:

```bash
# Run specific components only
via_server --components sequencer,verifier
via_server --components btc_sender
via_server --components all  # Run all components (default)
```

### Available Components

- `sequencer`: Transaction sequencing and batch creation
- `verifier`: Proof verification and validation
- `btc_sender`: Bitcoin transaction broadcasting
- `prover`: Proof generation (if applicable)
- `all`: All available components

### Environment Variable

Component selection can be configured via environment variable:
```bash
export VIA_SERVER_COMPONENTS="sequencer,verifier"
```

### Use Cases

- **Resource optimization**: Run only required components for specific deployment scenarios
- **Debugging**: Isolate and test individual components
- **Distributed deployment**: Deploy components across multiple servers
- **Development**: Focus on specific functionality during development

## Conclusion

The Via L2 Bitcoin ZK-Rollup system uses a comprehensive configuration system that allows for flexible deployment and operation. The wallet configuration system enables component-specific Bitcoin operations, while fee management ensures cost-effective transaction processing across different networks. Server component selection provides deployment flexibility for various operational requirements. By understanding the key configuration files, environment variables, and parameters, operators can effectively deploy and manage the system in various environments. The protocol version management system ensures smooth upgrades and backward compatibility.

## L1 Indexer Configuration

The L1 Indexer is a dedicated service that monitors Bitcoin L1 transactions and maintains synchronized state with the Via L2 system.

### Database Configuration

The L1 Indexer uses a dedicated database schema with the following tables:

```sql
-- Deposit tracking
CREATE TABLE deposits (
    id SERIAL PRIMARY KEY,
    txid VARCHAR(64) NOT NULL,
    vout INTEGER NOT NULL,
    amount BIGINT NOT NULL,
    address VARCHAR(100) NOT NULL,
    block_height INTEGER,
    confirmed BOOLEAN DEFAULT FALSE,
    processed BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Bridge withdrawal tracking
CREATE TABLE bridge_withdrawals (
    id SERIAL PRIMARY KEY,
    txid VARCHAR(64) NOT NULL,
    inscription_data TEXT,
    block_height INTEGER,
    confirmed BOOLEAN DEFAULT FALSE,
    processed BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Withdrawal tracking
CREATE TABLE withdrawals (
    id SERIAL PRIMARY KEY,
    txid VARCHAR(64) NOT NULL,
    amount BIGINT NOT NULL,
    address VARCHAR(100) NOT NULL,
    block_height INTEGER,
    confirmed BOOLEAN DEFAULT FALSE,
    processed BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Indexer metadata
CREATE TABLE indexer_metadata (
    key VARCHAR(50) PRIMARY KEY,
    value TEXT NOT NULL,
    updated_at TIMESTAMP DEFAULT NOW()
);
```

### Configuration Parameters

The L1 Indexer is configured through environment variables and configuration files:

#### Database Configuration
```toml
# etc/env/base/via_l1_indexer.toml
database_url = "postgresql://user:password@localhost/via_l1_indexer"
database_pool_size = 10
```

#### Bitcoin Node Configuration
```toml
# Bitcoin RPC connection
btc_rpc_url = "http://localhost:8332"
btc_rpc_user = "bitcoin"
btc_rpc_password = "password"
btc_network = "regtest"  # mainnet, testnet, regtest
```

#### Indexing Configuration
```toml
# Block processing configuration
start_block_height = 0
confirmation_blocks = 6
batch_size = 100
polling_interval = 10  # seconds

# Bridge configuration
bridge_address = "bc1qbridgeaddress..."
inscription_prefix = "via:"
```

### Environment Variables

Key environment variables for L1 Indexer configuration:

#### Database Variables
- `VIA_L1_INDEXER_DATABASE_URL`: PostgreSQL connection URL for the indexer database
- `VIA_L1_INDEXER_DATABASE_POOL_SIZE`: Database connection pool size (default: 10)

#### Bitcoin Node Variables
- `VIA_L1_INDEXER_BTC_RPC_URL`: Bitcoin RPC URL
- `VIA_L1_INDEXER_BTC_RPC_USER`: Bitcoin RPC username
- `VIA_L1_INDEXER_BTC_RPC_PASSWORD`: Bitcoin RPC password
- `VIA_L1_INDEXER_BTC_NETWORK`: Bitcoin network (mainnet/testnet/regtest)

#### Processing Variables
- `VIA_L1_INDEXER_START_BLOCK`: Starting block height for indexing
- `VIA_L1_INDEXER_CONFIRMATION_BLOCKS`: Number of confirmation blocks required
- `VIA_L1_INDEXER_BATCH_SIZE`: Number of blocks to process in each batch
- `VIA_L1_INDEXER_POLLING_INTERVAL`: Polling interval in seconds

#### Bridge Variables
- `VIA_L1_INDEXER_BRIDGE_ADDRESS`: Bitcoin bridge address to monitor
- `VIA_L1_INDEXER_INSCRIPTION_PREFIX`: Prefix for inscription data filtering

### Service Configuration

The L1 Indexer can be run as a standalone service or integrated with the main Via server:

#### Standalone Service
```bash
# Run L1 Indexer as standalone service
via-indexer --config /path/to/l1_indexer.toml

# Run with specific components
via-indexer --components deposit_monitor,withdrawal_processor
```

#### Integrated Service
```bash
# Run as part of main Via server
via_server --components sequencer,verifier,l1_indexer
```

### Docker Configuration

The L1 Indexer can be deployed using Docker:

```yaml
# docker-compose-l1-indexer.yml
version: '3.8'
services:
  l1-indexer:
    image: via/l1-indexer:latest
    environment:
      - VIA_L1_INDEXER_DATABASE_URL=postgresql://postgres:password@postgres:5432/via_l1_indexer
      - VIA_L1_INDEXER_BTC_RPC_URL=http://bitcoind:8332
      - VIA_L1_INDEXER_BTC_RPC_USER=bitcoin
      - VIA_L1_INDEXER_BTC_RPC_PASSWORD=password
      - VIA_L1_INDEXER_BTC_NETWORK=regtest
    depends_on:
      - postgres
      - bitcoind
    restart: unless-stopped

  postgres:
    image: postgres:15
    environment:
      - POSTGRES_DB=via_l1_indexer
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
    volumes:
      - l1_indexer_db:/var/lib/postgresql/data

volumes:
  l1_indexer_db:
```

### Monitoring and Health Checks

The L1 Indexer provides health check endpoints and metrics:

#### Health Check Endpoint
```bash
curl http://localhost:8080/health
```

#### Metrics Endpoint
```bash
curl http://localhost:8080/metrics
```

#### Key Metrics
- `via_l1_indexer_blocks_processed_total`: Total blocks processed
- `via_l1_indexer_deposits_found_total`: Total deposits found
- `via_l1_indexer_withdrawals_processed_total`: Total withdrawals processed
- `via_l1_indexer_last_processed_block`: Last processed block height
- `via_l1_indexer_processing_lag_blocks`: Number of blocks behind tip

### CLI Management

The L1 Indexer provides CLI commands for management:

```bash
# Start the indexer
via-indexer start

# Stop the indexer
via-indexer stop

# Restart the indexer
via-restart-indexer

# Check indexer status
via-indexer status

# Reset indexer state (caution: destructive)
via-indexer reset --confirm
```

### Troubleshooting

Common configuration issues and solutions:

#### Database Connection Issues
```bash
# Test database connection
psql $VIA_L1_INDEXER_DATABASE_URL -c "SELECT 1;"

# Check database schema
via-indexer check-schema
```

#### Bitcoin Node Connection Issues
```bash
# Test Bitcoin RPC connection
bitcoin-cli -rpcconnect=localhost -rpcport=8332 -rpcuser=bitcoin -rpcpassword=password getblockchaininfo

# Check indexer Bitcoin connection
via-indexer test-btc-connection
```

#### Performance Tuning
- Increase `batch_size` for faster initial sync
- Adjust `polling_interval` based on network requirements
- Tune `database_pool_size` based on concurrent load
- Use appropriate `confirmation_blocks` for security vs. speed trade-off