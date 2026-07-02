# Via L2 Bitcoin ZK-Rollup: Dependencies

This document lists the major external dependencies of the Via L2 Bitcoin ZK-Rollup system, verified against the Cargo manifests, `package.json` files, and `.gitmodules` in the `via-core` repository. All dependency lines are quoted verbatim from the manifests.

## Table of Contents

1. [Relationship to zksync-era](#relationship-to-zksync-era)
2. [Bitcoin and MuSig2 Stack](#bitcoin-and-musig2-stack)
3. [Celestia Data Availability Client](#celestia-data-availability-client)
4. [Workspace Rust Crates](#workspace-rust-crates)
5. [Prover Dependencies](#prover-dependencies)
6. [Git Submodules](#git-submodules)
7. [JavaScript Packages](#javascript-packages)
8. [External Services](#external-services)

## Relationship to zksync-era

`via-core` is a fork of Matter Labs' `zksync-era`, not a consumer of it as an external dependency. The workspace metadata still carries the upstream identity:

```toml
# Cargo.toml
authors = ["The Matter Labs Team <hello@matterlabs.dev>"]
homepage = "https://zksync.io/"
repository = "https://github.com/matter-labs/zksync-era"
```

Consequently, all `zksync_*` core crates (`zksync_dal`, `zksync_types`, `zksync_state`, `zksync_multivm`, and so on) are in-tree path dependencies, for example:

```toml
# Cargo.toml
zksync_multivm = { version = "0.1.0", path = "core/lib/multivm" }
zksync_dal = { version = "0.1.0", path = "core/lib/dal" }
zksync_object_store = { version = "0.1.0", path = "core/lib/object_store" }
```

The consensus stack is pulled from crates.io with exact pins:

```toml
# Cargo.toml
zksync_concurrency = "=0.7.0"
zksync_consensus_bft = "=0.7.0"
zksync_consensus_crypto = "=0.7.0"
zksync_consensus_executor = "=0.7.0"
zksync_consensus_network = "=0.7.0"
zksync_consensus_roles = "=0.7.0"
zksync_consensus_storage = "=0.7.0"
zksync_consensus_utils = "=0.7.0"
zksync_protobuf = "=0.7.0"
zksync_protobuf_build = "=0.7.0"
```

The new VM is pinned to a specific commit:

```toml
# Cargo.toml
zksync_vm2 = { git = "https://github.com/matter-labs/vm2.git", rev = "457d8a7eea9093af9440662e33e598c13ba41633" }
```

## Bitcoin and MuSig2 Stack

These are the Via-specific additions on top of the zksync-era base.

Workspace-level Bitcoin crates:

```toml
# Cargo.toml
bitcoin = { version = "0.32.7", features = ["serde"] }
bitcoincore-rpc = "0.19.0"
secp256k1 = { version = "0.27.0", features = ["recovery", "global-context"] }
```

The MuSig2 implementation is the upstream `musig2` crate from crates.io (it resolves to 0.2.1 in `Cargo.lock`; it is not a fork or a git dependency). It is used by `via_verifier/lib/via_musig2`, `core/lib/via_btc_client`, and `via_verifier/node/via_verifier_coordinator`, together with a newer `secp256k1` aliased as `secp256k1_musig2`:

```toml
# via_verifier/lib/via_musig2/Cargo.toml
musig2 = "0.2.0"
secp256k1_musig2 = { package = "secp256k1", version = "0.30.0", features = [
    "rand",
    "hashes",
] }
```

`via_btc_client` additionally pins its own `secp256k1`:

```toml
# core/lib/via_btc_client/Cargo.toml
bitcoin.workspace = true
bitcoincore-rpc.workspace = true
secp256k1 = "0.29.0"
musig2 = "0.2.0"
```

So three `secp256k1` versions coexist in the build (confirmed in `Cargo.lock`): 0.27.0 (workspace, zksync-inherited), 0.29.0 (`via_btc_client`), and 0.30.0 (aliased `secp256k1_musig2` for MuSig2 code).

## Celestia Data Availability Client

The Via DA client (`core/lib/via_da_clients`) uses the Celestia client crates from Eiger's lumina repository, pinned by git:

```toml
# core/lib/via_da_clients/Cargo.toml
celestia-rpc = { version = "=0.4.0", git = "https://github.com/eigerco/lumina.git" }
celestia-types = { version = "=0.4.0", git = "https://github.com/eigerco/lumina.git" }
```

`Cargo.lock` resolves these to lumina commit `f11959550851afb8c2062b30e38cc87242467cdc`.

Separately, the inherited zksync DA clients declare a crates.io version at the workspace level (used by `core/node/da_clients`):

```toml
# Cargo.toml
# Celestia
celestia-types = "0.6.1"
bech32 = "0.11.0"
ripemd = "0.1.3"
tonic = { version = "0.11.0", default-features = false }
pbjson-types = "0.6.0"
```

## Workspace Rust Crates

Selected entries from `[workspace.dependencies]` in the root `Cargo.toml`, quoted verbatim.

### Cryptography and hashing

```toml
# Cargo.toml
blake2 = "0.10"
sha2 = "0.10.8"
sha3 = "0.10.8"
tiny-keccak = "2"
```

### Protocol crates (zk_evm, circuit APIs, KZG)

```toml
# Cargo.toml
circuit_encodings = { package = "circuit_encodings", version = "=0.150.18" }
circuit_sequencer_api = { package = "circuit_sequencer_api", version = "=0.150.18" }
crypto_codegen = { package = "zksync_solidity_vk_codegen", version = "=0.30.11" }
kzg = { package = "zksync_kzg", version = "=0.150.18" }
zk_evm = { version = "=0.133.0" }
zk_evm_1_3_1 = { package = "zk_evm", version = "0.131.0-rc.2" }
zk_evm_1_3_3 = { package = "zk_evm", version = "0.133" }
zk_evm_1_4_0 = { package = "zk_evm", version = "0.140" }
zk_evm_1_4_1 = { package = "zk_evm", version = "0.141" }
zk_evm_1_5_0 = { package = "zk_evm", version = "=0.150.18"  }
```

### Database and storage

```toml
# Cargo.toml
sqlx = "0.8.1"
rocksdb = "0.21"
lru = { version = "0.12.1", default-features = false }
mini-moka = "0.10.0"
google-cloud-storage = "0.20.0"
```

### Networking and API

```toml
# Cargo.toml
axum = "0.7.5"
reqwest = "0.12"
jsonrpsee = { version = "0.23", default-features = false }
tower = "0.4.13"
tower-http = "0.5.2"
hyper = "1.3"
web3 = "0.19.0"
```

### Serialization

```toml
# Cargo.toml
serde = "1"
serde_json = "1"
serde_yaml = "0.9"
bincode = "1"
prost = "0.12.6"
```

### Async runtime and utilities

```toml
# Cargo.toml
tokio = "1"
futures = "0.3"
rayon = "1.3.1"
backon = "0.4.4"
dashmap = "5.5.3"
governor = "0.4.2"
```

### Monitoring and observability

```toml
# Cargo.toml
tracing = "0.1"
tracing-subscriber = "0.3"
opentelemetry = "0.24.0"
sentry = "0.31"
vise = "0.2.0"
vise-exporter = "0.2.0"
```

## Prover Dependencies

The prover workspace (`prover/Cargo.toml`) has its own dependency section. The proving stack is pinned exactly:

```toml
# prover/Cargo.toml
# Proving dependencies
circuit_definitions = "=0.150.18"
circuit_sequencer_api = "=0.150.18"
zkevm_test_harness = "=0.150.18"

# GPU proving dependencies
wrapper_prover = { package = "zksync-wrapper-prover", version = "=0.152.9" }
shivini = "=0.152.9"
boojum-cuda = "=0.152.9"
```

`boojum` and `franklin-crypto` are not declared directly in any manifest; they arrive transitively through `circuit_definitions` / `zkevm_test_harness` and resolve in `Cargo.lock` to version 0.30.11 from crates.io for both.

## Git Submodules

```
# .gitmodules
[submodule "contracts"]
    path = contracts
    url = https://github.com/vianetwork/era-contracts.git
    branch = main

[submodule "via-core-ext"]
	path = via-core-ext
	url = https://github.com/vianetwork/via-core-ext.git
	branch = main
```

The L1/L2 contracts come from Via's fork of `era-contracts`, and `via-core-ext` holds Via-specific extensions.

## JavaScript Packages

Development tooling from the root `package.json` (devDependencies):

```json
// package.json
"@typescript-eslint/eslint-plugin": "^6.7.4",
"@typescript-eslint/parser": "^4.10.0",
"babel-eslint": "^10.1.0",
"eslint": "^7.16.0",
"eslint-config-alloy": "^3.8.2",
"markdownlint-cli": "^0.24.0",
"prettier": "^3.3.3",
"prettier-plugin-solidity": "=1.0.0-dev.22",
"solhint": "^3.3.2",
"sql-formatter": "^13.1.0"
```

The `infrastructure/` directory contains TypeScript tooling packages (`infrastructure/via`, `infrastructure/zk`, `infrastructure/local-setup-preparation`, `infrastructure/protocol-upgrade`, `infrastructure/via-protocol-upgrade`), all built with TypeScript 4.x and managed with Yarn.

## External Services

Grounded in the dependencies above:

| Service | Evidence in manifests | Purpose |
|---------|----------------------|---------|
| Bitcoin node | `bitcoincore-rpc = "0.19.0"` | Base layer (L1); inscriptions, deposits, withdrawals |
| Celestia node | `celestia-rpc` / `celestia-types` (lumina git pin) | Data availability layer for pubdata and proofs |
| PostgreSQL | `sqlx = "0.8.1"` | Primary relational database |
| Google Cloud Storage | `google-cloud-storage = "0.20.0"` | Object store backend (`zksync_object_store`) |
| CUDA GPU | `boojum-cuda`, `shivini`, `zksync-wrapper-prover` (all `=0.152.9`) | GPU proof generation |
