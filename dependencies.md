# Via L2 Bitcoin ZK-Rollup: Dependencies

This document provides an overview of the major external dependencies used by the Via L2 Bitcoin ZK-Rollup system, categorized by type.

## Table of Contents

1. [Rust Crates](#rust-crates)
2. [JavaScript Packages](#javascript-packages)
3. [External Services](#external-services)
4. [Blockchain Interfaces](#blockchain-interfaces)
5. [Infrastructure Dependencies](#infrastructure-dependencies)

## Rust Crates

The Via L2 system relies on numerous Rust crates for its core functionality:

### Cryptography and Zero-Knowledge Proofs

| Crate | Version | Purpose |
|-------|---------|---------|
| `musig2` | 0.2.0 | Core MuSig2 protocol implementation for multi-signature transactions |
| `secp256k1` | 0.27.0, 0.30.0 | Elliptic curve cryptography for Bitcoin signatures |
| `blake2` | 0.10 | Cryptographic hashing algorithm |
| `sha2` | 0.10.8 | SHA-2 cryptographic hash functions |
| `sha3` | 0.10.8 | SHA-3 cryptographic hash functions |
| `tiny-keccak` | 2 | Keccak hash function implementation |
| `zk_evm` | Various (0.131.0 - 0.150.4) | Zero-knowledge EVM implementation |
| `circuit_sequencer_api` | Various (0.133 - 0.150.4) | API for the circuit sequencer |
| `kzg` | 0.150.4 | KZG polynomial commitments |

### Database and Storage

| Crate | Version | Purpose |
|-------|---------|---------|
| `sqlx` | 0.8.1 | Async SQL toolkit for Rust |
| `rocksdb` | 0.21.0 | Key-value storage engine |
| `lru` | 0.12.1 | LRU cache implementation |
| `mini-moka` | 0.10.0 | Concurrent caching library |
| `google-cloud-storage` | 0.20.0 | Google Cloud Storage client |

### Networking and API

| Crate | Version | Purpose |
|-------|---------|---------|
| `axum` | 0.7.5 | Web framework for building APIs |
| `reqwest` | 0.12 | HTTP client |
| `jsonrpsee` | 0.23 | JSON-RPC implementation |
| `tower` | 0.4.13 | Middleware for network services |
| `tower-http` | 0.5.2 | HTTP-specific middleware for Tower |
| `hyper` | 1.3 | HTTP implementation |
| `web3` | 0.19.0 | Ethereum JSON-RPC client |

### Serialization and Data Processing

| Crate | Version | Purpose |
|-------|---------|---------|
| `serde` | 1 | Serialization/deserialization framework |
| `serde_json` | 1 | JSON support for Serde |
| `serde_yaml` | 0.9 | YAML support for Serde |
| `bincode` | 1 | Binary serialization format |
| `prost` | 0.12.1 | Protocol Buffers implementation |

### Async Runtime and Utilities

| Crate | Version | Purpose |
|-------|---------|---------|
| `tokio` | 1 | Async runtime |
| `futures` | 0.3 | Async programming primitives |
| `rayon` | 1.3.1 | Data parallelism library |
| `backon` | 0.4.4 | Retry functionality |
| `dashmap` | 5.5.3 | Concurrent hash map |
| `governor` | 0.4.2 | Rate limiting |

### Monitoring and Observability

| Crate | Version | Purpose |
|-------|---------|---------|
| `tracing` | 0.1 | Application-level tracing |
| `tracing-subscriber` | 0.3 | Subscriber for tracing events |
| `opentelemetry` | 0.24.0 | OpenTelemetry implementation |
| `sentry` | 0.31 | Error monitoring with Sentry |
| `vise` | 0.2.0 | Metrics collection |
| `vise-exporter` | 0.2.0 | Metrics exporting |

### Consensus

| Crate | Version | Purpose |
|-------|---------|---------|
| `zksync_consensus_bft` | 0.1.0-rc.11 | BFT consensus implementation |
| `zksync_consensus_crypto` | 0.1.0-rc.11 | Cryptography for consensus |
| `zksync_consensus_executor` | 0.1.0-rc.11 | Consensus execution |
| `zksync_consensus_network` | 0.1.0-rc.11 | Network layer for consensus |
| `zksync_consensus_roles` | 0.1.0-rc.11 | Role definitions for consensus |
| `zksync_consensus_storage` | 0.1.0-rc.11 | Storage for consensus |

## JavaScript Packages

The Via L2 system uses JavaScript/TypeScript for various tools and infrastructure:

### Development and Testing

| Package | Purpose |
|---------|---------|
| `typescript-eslint` | TypeScript linting |
| `eslint` | JavaScript linting |
| `prettier` | Code formatting |
| `solhint` | Solidity linting |
| `markdownlint-cli` | Markdown linting |
| `sql-formatter` | SQL formatting |

## External Services

The Via L2 system integrates with several external services:

### Blockchain Networks

| Service | Purpose |
|---------|---------|
| **Bitcoin** | Base layer (L1) providing security and finality |
| **Celestia** | Data Availability (DA) layer for storing transaction data |

### Infrastructure Services

| Service | Purpose |
|---------|---------|
| **PostgreSQL** | Primary relational database for storing system state |
| **Google Cloud Storage** | Object storage for large data (witness vectors, proofs) |
| **Docker** | Containerization for deployment |

## Blockchain Interfaces

The Via L2 system interfaces with multiple blockchain networks:

### Bitcoin Interface

| Component | Purpose |
|-----------|---------|
| `via_btc_client` | Client for interacting with the Bitcoin network |
| `via_btc_watch` | Service for monitoring the Bitcoin blockchain |
| `via_btc_sender` | Service for sending transactions to the Bitcoin network |
| `BitcoinInscriptionIndexer` | Component for indexing inscriptions on Bitcoin |

### Celestia Interface

| Component | Purpose |
|-----------|---------|
| `CelestiaClient` | Client for interacting with the Celestia network |
| `ViaDataAvailabilityDispatcher` | Service for dispatching data to Celestia |

## Infrastructure Dependencies

The Via L2 system has several infrastructure dependencies:

### Compute Resources

| Resource | Purpose |
|----------|---------|
| **GPU** | Required for proof generation (CUDA-compatible with at least 24GB VRAM) |
| **CPU** | High-performance CPUs for witness generation and other operations |
| **Memory** | Minimum 64GB RAM for proof generation |
| **Storage** | 200GB+ for storing blockchain data and proofs |

### Networking

| Component | Purpose |
|-----------|---------|
| **Bitcoin Node** | Full Bitcoin node for interacting with the Bitcoin network |
| **Celestia Node** | Light node for interacting with the Celestia network |

### Development Tools

| Tool | Purpose |
|------|---------|
| **Nix** | Reproducible builds and development environments |
| **Cargo** | Rust package manager |
| **Yarn** | JavaScript package manager |
| **Docker Compose** | Multi-container Docker applications |

## Conclusion

The Via L2 Bitcoin ZK-Rollup system relies on a diverse set of dependencies to provide its functionality. These dependencies span from low-level cryptographic libraries to high-level infrastructure services, all working together to create a secure, scalable, and efficient Layer 2 solution for Bitcoin.