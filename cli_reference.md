# Via L2 CLI Reference Guide

## Overview

The Via L2 system provides a comprehensive command-line interface (CLI) for managing and operating various components of the Bitcoin ZK-Rollup infrastructure. This guide covers all CLI commands, their options, and usage patterns for developers and operators.

## Table of Contents

1. [Installation and Setup](#installation-and-setup)
2. [Core Commands](#core-commands)
3. [L1 Indexer Commands](#l1-indexer-commands)
4. [Infrastructure Commands](#infrastructure-commands)
5. [Development Commands](#development-commands)
6. [Configuration Management](#configuration-management)
7. [Troubleshooting](#troubleshooting)

## Installation and Setup

### Prerequisites

- Rust 1.70 or later
- PostgreSQL 13 or later
- Bitcoin Core node (for mainnet/testnet operations)
- Docker (optional, for containerized deployment)

### Building CLI Tools

```bash
# Build all CLI tools
cargo build --release

# Build specific tools
cargo build --release --bin via_indexer_bin
cargo build --release --bin zksync_server

# Install to system PATH
cargo install --path .
```

### Environment Setup

```bash
# Set Via home directory
export VIA_HOME=/path/to/via-core

# Add Via binaries to PATH
export PATH=$VIA_HOME/target/release:$PATH

# Load environment configuration
source .env
```

## Core Commands

### via-server

Main server command for running Via L2 components.

#### Syntax

```bash
via-server [OPTIONS] --components <COMPONENTS>
```

#### Options

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `--components <COMPONENTS>` | Comma-separated list of components to run | Required | `api,tree,eth,data_fetcher` |
| `--genesis` | Run genesis setup | `false` | `--genesis` |
| `--chain <CHAIN>` | Specify chain name | `era` | `--chain era` |
| `--config <CONFIG>` | Configuration file path | Auto-detected | `--config config.yaml` |

#### Component Options

Available components for `--components` flag:

- `api` - JSON-RPC API server
- `tree` - Merkle tree component
- `eth` - Ethereum interaction layer
- `data_fetcher` - Data availability fetcher
- `state_keeper` - State management
- `housekeeper` - Maintenance tasks
- `proof_data_handler` - Proof processing
- `commitment_generator` - Commitment generation

#### Examples

```bash
# Start full server with all components
via-server --components api,tree,eth,data_fetcher,state_keeper

# Start API server only
via-server --components api

# Run genesis setup
via-server --genesis --components api,tree

# Start with custom configuration
via-server --components api,tree --config /path/to/config.yaml
```

## L1 Indexer Commands

### via-indexer

**New in this release** - Dedicated Bitcoin L1 indexer service for processing deposits and withdrawals.

#### Syntax

```bash
via-indexer [OPTIONS]
```

#### Options

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `--network <NETWORK>` | Bitcoin network to connect to | `regtest` | `--network mainnet` |
| `--config <CONFIG>` | Configuration file path | Auto-detected | `--config indexer.toml` |
| `--start-block <BLOCK>` | Starting block number for indexing | Latest | `--start-block 800000` |
| `--batch-size <SIZE>` | Number of blocks to process per batch | `100` | `--batch-size 500` |
| `--poll-interval <SECONDS>` | Polling interval in seconds | `10` | `--poll-interval 5` |

#### Network Options

- `mainnet` - Bitcoin mainnet
- `testnet` - Bitcoin testnet
- `signet` - Bitcoin signet
- `regtest` - Bitcoin regtest (development)

#### Examples

```bash
# Start indexer on regtest (development)
via-indexer --network regtest

# Start indexer on mainnet from specific block
via-indexer --network mainnet --start-block 800000

# Start with custom configuration and high throughput
via-indexer --network testnet --config indexer.toml --batch-size 500 --poll-interval 5

# Start indexer with verbose logging
RUST_LOG=via_indexer=debug via-indexer --network regtest
```

#### Configuration File Example

```toml
# indexer.toml
[indexer]
network = "regtest"
start_block = 800000
batch_size = 100
poll_interval = 10

[database]
url = "postgresql://user:password@localhost:5432/via_indexer"
max_connections = 20

[bitcoin]
rpc_url = "http://localhost:8332"
rpc_user = "rpcuser"
rpc_password = "rpcpassword"

[logging]
level = "info"
format = "json"
```

### via-restart-indexer

**New in this release** - Utility command for restarting the indexer service.

#### Syntax

```bash
via-restart-indexer [OPTIONS]
```

#### Options

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `--force` | Force restart without graceful shutdown | `false` | `--force` |
| `--wait <SECONDS>` | Wait time before restart | `5` | `--wait 10` |
| `--config <CONFIG>` | Configuration file path | Auto-detected | `--config indexer.toml` |

#### Examples

```bash
# Graceful restart
via-restart-indexer

# Force restart immediately
via-restart-indexer --force

# Restart with custom wait time
via-restart-indexer --wait 30

# Restart with specific configuration
via-restart-indexer --config /path/to/indexer.toml
```

## Infrastructure Commands

### Database Management

#### via-db-setup

Initialize and setup databases for Via components.

```bash
# Setup all databases
via-db-setup --all

# Setup specific databases
via-db-setup --core --indexer

# Setup with custom URLs
via-db-setup --core --url postgresql://localhost:5432/via_core
```

#### via-db-migrate

Run database migrations.

```bash
# Run all migrations
via-db-migrate

# Run migrations for specific component
via-db-migrate --component indexer

# Rollback migrations
via-db-migrate --rollback --steps 1
```

#### via-db-reset

Reset databases (development only).

```bash
# Reset all databases
via-db-reset --all

# Reset specific databases
via-db-reset --indexer --verifier

# Reset with confirmation
via-db-reset --core --confirm
```

### Docker Commands

#### via-docker-build

Build Docker images for Via components.

```bash
# Build all images
via-docker-build

# Build specific image
via-docker-build --image via-l1-indexer

# Build with custom tag
via-docker-build --image via-l1-indexer --tag latest
```

#### via-docker-run

Run Via components in Docker containers.

```bash
# Run indexer in Docker
via-docker-run --service indexer --network regtest

# Run with custom environment
via-docker-run --service indexer --env-file .env.docker

# Run with port mapping
via-docker-run --service api --port 3000:3000
```

## Development Commands

### Testing Commands

#### via-test

Run Via system tests.

```bash
# Run all tests
via-test

# Run specific test suite
via-test --suite integration

# Run tests with coverage
via-test --coverage

# Run tests for specific component
via-test --component indexer
```

#### via-loadtest

Load testing utility for Via L2 system.

```bash
# Basic load test
via-loadtest --duration 60s --rate 100

# Load test with custom configuration
via-loadtest --config loadtest.yaml --output results.json

# Test specific endpoints
via-loadtest --endpoint http://localhost:3000 --method eth_getBalance
```

### Development Utilities

#### via-dev-setup

Setup development environment.

```bash
# Full development setup
via-dev-setup --all

# Setup for indexer development
via-dev-setup --indexer

# Setup with sample data
via-dev-setup --with-data
```

#### via-generate-keys

Generate cryptographic keys for development.

```bash
# Generate verifier keys
via-generate-keys --type verifier --count 3

# Generate with custom output
via-generate-keys --type bridge --output keys/

# Generate for specific network
via-generate-keys --type verifier --network testnet
```

## Configuration Management

### Environment Configuration

#### via-config-validate

Validate configuration files.

```bash
# Validate all configurations
via-config-validate

# Validate specific configuration
via-config-validate --config indexer.toml

# Validate with schema
via-config-validate --schema config.schema.json
```

#### via-config-generate

Generate configuration templates.

```bash
# Generate indexer configuration
via-config-generate --type indexer --output indexer.toml

# Generate for specific network
via-config-generate --type server --network mainnet

# Generate with custom values
via-config-generate --type indexer --set database.url=postgresql://localhost/via
```

### Environment Variables

Key environment variables used by CLI commands:

#### Database Configuration

```bash
# Core database
export DATABASE_URL="postgresql://user:password@localhost:5432/via_core"

# Indexer database
export DATABASE_INDEXER_URL="postgresql://user:password@localhost:5432/via_indexer"

# Verifier database
export DATABASE_VERIFIER_URL="postgresql://user:password@localhost:5432/via_verifier"
```

#### Bitcoin Configuration

```bash
# Bitcoin RPC settings
export VIA_BTC_CLIENT_RPC_URL="http://localhost:8332"
export VIA_BTC_CLIENT_RPC_USER="rpcuser"
export VIA_BTC_CLIENT_RPC_PASSWORD="rpcpassword"
export VIA_BTC_CLIENT_NETWORK="regtest"
```

#### Logging Configuration

```bash
# Logging levels
export RUST_LOG="via_indexer=info,via_server=debug"

# Log format
export VIA_LOG_FORMAT="json"  # or "pretty"

# Log output
export VIA_LOG_FILE="/var/log/via/via.log"
```

## Command Workflows

### Development Workflow

Complete development setup and testing:

```bash
# 1. Setup development environment
via-dev-setup --all

# 2. Initialize databases
via-db-setup --all

# 3. Run migrations
via-db-migrate

# 4. Generate development keys
via-generate-keys --type verifier --count 3

# 5. Start indexer
via-indexer --network regtest &

# 6. Start server
via-server --components api,tree,eth,data_fetcher &

# 7. Run tests
via-test --suite integration

# 8. Run load tests
via-loadtest --duration 30s --rate 50
```

### Production Deployment

Production deployment workflow:

```bash
# 1. Validate configuration
via-config-validate --config production.toml

# 2. Setup production databases
via-db-setup --core --indexer --verifier

# 3. Run migrations
via-db-migrate --environment production

# 4. Start indexer service
via-indexer --network mainnet --config production.toml

# 5. Start main server
via-server --components api,tree,eth,data_fetcher,state_keeper --config production.toml
```

### Maintenance Workflow

Regular maintenance tasks:

```bash
# 1. Check system health
curl http://localhost:3000/health
curl http://localhost:3001/health  # indexer

# 2. Monitor indexer progress
via-indexer-status

# 3. Backup databases
pg_dump $DATABASE_URL > backup_$(date +%Y%m%d).sql
pg_dump $DATABASE_INDEXER_URL > indexer_backup_$(date +%Y%m%d).sql

# 4. Restart services if needed
via-restart-indexer
systemctl restart via-server

# 5. Check logs
tail -f /var/log/via/via.log
tail -f /var/log/via/indexer.log
```

## Troubleshooting

### Common Issues and Solutions

#### 1. Command Not Found

**Issue:** `via-indexer: command not found`

**Solutions:**
```bash
# Check if binary exists
ls -la target/release/via_indexer_bin

# Add to PATH
export PATH=$PWD/target/release:$PATH

# Or create symlink
ln -s $PWD/target/release/via_indexer_bin /usr/local/bin/via-indexer
```

#### 2. Database Connection Errors

**Issue:** Cannot connect to database

**Solutions:**
```bash
# Test database connection
psql $DATABASE_INDEXER_URL -c "SELECT 1;"

# Check database service
systemctl status postgresql

# Verify environment variables
echo $DATABASE_INDEXER_URL
```

#### 3. Bitcoin RPC Connection Issues

**Issue:** Cannot connect to Bitcoin node

**Solutions:**
```bash
# Test Bitcoin RPC
curl -u $VIA_BTC_CLIENT_RPC_USER:$VIA_BTC_CLIENT_RPC_PASSWORD \
  -d '{"jsonrpc":"1.0","id":"test","method":"getblockchaininfo","params":[]}' \
  -H 'content-type: text/plain;' \
  $VIA_BTC_CLIENT_RPC_URL

# Check Bitcoin node status
bitcoin-cli -rpcuser=$VIA_BTC_CLIENT_RPC_USER -rpcpassword=$VIA_BTC_CLIENT_RPC_PASSWORD getblockchaininfo
```

#### 4. Permission Errors

**Issue:** Permission denied when running commands

**Solutions:**
```bash
# Check file permissions
ls -la target/release/via_indexer_bin

# Make executable
chmod +x target/release/via_indexer_bin

# Check directory permissions
ls -la /var/log/via/
```

### Debug Mode

Enable debug mode for detailed logging:

```bash
# Enable debug logging
export RUST_LOG=debug

# Enable trace logging for specific component
export RUST_LOG=via_indexer=trace

# Run with debug output
via-indexer --network regtest 2>&1 | tee debug.log
```

### Performance Monitoring

Monitor command performance:

```bash
# Monitor system resources
htop

# Monitor database connections
psql $DATABASE_INDEXER_URL -c "SELECT * FROM pg_stat_activity;"

# Monitor Bitcoin RPC calls
tail -f ~/.bitcoin/regtest/debug.log | grep rpc

# Check indexer metrics
curl http://localhost:3001/metrics
```

### Log Analysis

Analyze logs for issues:

```bash
# Search for errors
grep -i error /var/log/via/via.log

# Search for specific patterns
grep "database connection" /var/log/via/indexer.log

# Monitor real-time logs
tail -f /var/log/via/via.log | grep -i "indexer"

# Analyze log patterns
awk '/ERROR/ {print $1, $2, $NF}' /var/log/via/via.log
```

## Best Practices

### Command Usage

1. **Always validate configuration** before starting services
2. **Use specific network flags** to avoid accidental mainnet operations
3. **Monitor logs** during initial startup
4. **Test database connections** before running migrations
5. **Use graceful shutdowns** when stopping services

### Security Considerations

1. **Protect RPC credentials** - never expose in command history
2. **Use environment files** for sensitive configuration
3. **Limit database permissions** to minimum required
4. **Monitor command execution** in production environments
5. **Regularly rotate credentials** and update configurations

### Performance Optimization

1. **Tune batch sizes** based on system resources
2. **Adjust polling intervals** for optimal performance
3. **Monitor database connections** and tune pool sizes
4. **Use appropriate logging levels** in production
5. **Regular maintenance** of databases and logs

The Via L2 CLI provides comprehensive tools for managing all aspects of the Bitcoin ZK-Rollup system, from development to production deployment and maintenance.