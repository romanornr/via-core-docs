---
description: Dive into VIA Protocol and navigate through different sections.
---

# Introduction

Welcome to the **VIA Protocol** :wave:, this documentation serves as a starting point and a comprehensive resource for interacting with VIA. Whether you're a developer exploring our [developer docs](../developer-docs), or a user getting started through our [user guide](../user-guide), this knowledge base has you covered. Additionally, you can learn more about VIA Protocol's architecture and security properties by exploring the links in the sidebar.

# Via L2: A Bitcoin L2 EVM-compatible Modular Sovereign ZK‑Rollup

Welcome to the comprehensive documentation for Via

## Simple intro to Via 

Via is a Bitcoin Layer 2 Modular Sovereign ZK-Rollup that combines zero-knowledge proofs, zkEVM, Bitcoin's PoW security, and Celestia data availability layer by adopting a modular approach without compromises.

Our zkEVM enables developers to write, test, deploy, or migrate existing smart contracts, using a framework and developer tools they're already familiar with, by using our SDK.

Celestia acts as our data availability layer, stores each batch's calldata and zk-proofs with scalable throughput and verifiability.

Our Verifier Network ingests the data, validates every proof, and finalizes batches before periodically anchoring a compact state root to the Bitcoin blockchain.

By anchoring our rollup's state root into Bitcoin, Via inherits Bitcoin's finality guarantees and Proof-of-Work security without bloating Bitcoin's UTXO set or incurring massive fees every batch.

Via operates as a modular sovereign rollup, meaning it retains full autonomy over off-chain execution and on-chain validation.Unlike monolithic blockchains, where the rollup is integrated into the base layer, sovereign rollups decouple execution from consensus and settlement.

This modular design allows us to innovate and evolve independently. Because execution data, data availability, and settlement are separated, each layer can evolve at its own pace. Unlike traditional rollups, where users may be forced to exit a compromised chain, sovereign rollups, such as Via, allow the network to reject invalid state transitions altogether.

In the event of a faulty execution due to malicious behavior, honest nodes can coordinate a fork, preserving network integrity without relying on a fallback Layer 1 withdrawal mechanism. This gives our sovereign rollup strong autonomy, fault recovery, and resistance.

Until very recently, there weren't many credible options to denominate gas fees in hard assets like BTC. Ethereum isn't built for that. Solana doesn't support that.

Most blockchains are not modular or flexible enough to decouple execution from fee logic.
Via, with its modular architecture and Bitcoin as a settlement layer, can charge BTC for gas fees and accumulate a meaningful reserve. Most L1s and other rollups can't.

### How does Via L2 work?

Via is a modular sovereign, validity-proof zk-rollup for Bitcoin with a zkEVM execution layer that scales throughput without introducing new L1 programmability or custodial trust. 

It relies on Bitcoin-native primitives: Taproot for key aggregation, inscriptions/OP_RETURN to record state commitments and governance messages, and a MuSig2 threshold bridge for BTC withdrawals. 

For each L2 batch, Via posts a single 32-byte state root to Bitcoin along with a pointer to the batch data on the data-availability layer. A zero-knowledge validity proof is produced off-chain that anyone can verify.

The proof is succinct, small and fast to verify with cost independent of batch size, so Bitcoin’s on-chain role remains minimal to durable commitments and governance while execution happens off-chain at high throughput and low cost.

Data availability is separated from settlement. Full batch data is made available on Celestia (DA), while Bitcoin serves as the commitment and settlement anchor where L2 state roots and governance actions are periodically recorded.

A distributed verifier network independently validates ZK proofs against the data posted to Celestia and only recognizes batches whose commitments and proofs are consistent. Inscriptions function as state commitment posts and governance instructions, not as an L1 consensus mechanism, allowing Via to build on Bitcoin for securing durable commitments and reorg-bounded finality without embedding complex logic on-chain.

Security and trust assumptions are explicit. L2 correctness relies on the soundness of the validity proofs and on data availability from Celestia; clients can verify proofs and sample DA independently. Economic finality of the posted commitments inherits Bitcoin’s reorg resistance once inscriptions confirm.

The BTC bridge is trust-minimized via [MuSig2](https://eprint.iacr.org/2020/1261): withdrawals require a threshold of keys that are governed on-chain (via governance inscriptions) and operationally constrained to sign only for finalized, proven exits. 

This approach avoids merged mining and is secured by Bitcoin by anchoring zk-verified batch roots and governance messages on L1, minimizing on-chain footprint, and achieving low fees and high throughput through off-chain execution and data, with validity proofs ensuring correctness.

Via's design minimizes Bitcoin's on-chain footprint by committing only batch roots and governance updates, keeping costs low while avoiding miner dependencies, while delivering full EVM programmability and targeting throughput and cost efficiency well beyond Ethereum L1, all while maintaining Bitcoin‑level settlement guarantees and end‑user verifiability.

## How Via L2 Works: Batch Lifecycle and Anchoring

For each L2 batch, Via posts a single 32-byte state root to Bitcoin along with a pointer to the batch data on the data-availability layer. A zero-knowledge validity proof is produced off-chain that anyone can verify. The proof is succinct, small and fast to verify with cost independent of batch size. So Bitcoin’s on-chain role remains limited to durable commitments and governance while execution happens off-chain at high throughput and low cost.

Celestia provides data availability, while Bitcoin serves as the commitment and settlement anchor, where state roots and governance actions are periodically recorded. A distributed verifier network, along with end users, fetches batch data from Celestia, verifies the ZK proof against the posted state root, and only accepts batches that match the commitments and proofs. In this setup, inscriptions are authenticated posts for state commitments and governance activities (such as system wallet updates), rather than an L1 validity consensus mechanism. This separation allows Via to anchor to Bitcoin for reorg-bounded finality and censorship resistance, without embedding complex logic on-chain.

Security and trust assumptions are explicit. L2 correctness depends on the soundness of the validity proofs and the availability of batch data on Celestia; clients can verify proofs and independently sample or audit DA. Once inscriptions confirm, the economic finality of posted commitments inherits Bitcoin’s reorg resistance.

BTC custody is secured through a MuSig2 threshold signer set, with keys and policies defined by on-chain inscriptions. This ensures that custody rules are transparent, auditable, and upgradeable without trusting any single party. Operators are cryptographically constrained to sign only for finalized, validity-proven exits, which eliminates arbitrary withdrawals and protects user funds.

Via is a sovereign rollup anchored by commitments. Via is not AuxPoW, merged mining, or a sidechain like Rootstock, Botanix Spiderchain, or Tether Plasma XPL. Sidechains depend on external guardian or validator sets, often using rotating threshold multisigs, which do not enforce state correctness or withdrawals in their L1 checkpoints.

In contrast to these sidechains, Via's custody and exits are enforced by posted commitments and ZK validity proofs, with a MuSig2 threshold signer set bound by on‑chain policies.

## Why Choose Via?

Via is a Bitcoin L2: an EVM-compatible, modular sovereign ZK rollup anchored to Bitcoin for settlement and using Celestia for data availability. It delivers EVM programmability on a Bitcoin‑anchored stack without adopting Ethereum’s consensus, security assumptions, or token economics.

### 🛡️ Advantages over Ethereum

**Anchored to Bitcoin’s PoW**
- Via periodically commits state roots to Bitcoin, deriving settlement assurances from the most battle-tested PoW network
- Avoids Ethereum’s Proof-of-Stake dynamics and validator slashing risks

**Decentralization & Censorship Resistance**
- Benefits from Bitcoin’s miner diversity and neutral, apolitical base layer
- No exposure to Ethereum validator policies or staking concentration

**Store-of-Value Base**
- Built around Bitcoin’s fixed-supply monetary foundation
- Fee economics are BTC-denominated rather than tied to ETH

**BTC Gas by Default**
- Users pay gas in Bitcoin (an L2-native/bridged representation), so no separate gas token is required
- Reduces friction and avoids multi-token operational risk

### ⚡ Advantages over Other Bitcoin “L2s”

**Validity-Proved Finality**
- ZK proofs + verifier attestations provide fast, provable finality without 7-day fraud windows
- Withdrawals finalize once proofs are verified (no fraud-challenge waiting game)

**Reduced Game-Theory Assumptions**
- No challengers/watchtowers; security rests on proof correctness and the verifier set
- Eliminates reliance on economic incentives that can be gamed

**Verifier-Network Security**
- Threshold attestation by multiple independent verifiers; no single operator can finalize alone
- Permissioned at launch with a path to greater decentralization

**Bitcoin-Native Anchoring, EVM Execution**
- Account-based zkEVM for apps and tooling; Taproot anchoring/inscription support on Bitcoin
- Purpose-built integration for Bitcoin settlement rather than adapting an Ethereum rollup

**Scalable Data Availability**
- Celestia provides horizontal DA scaling, avoiding Bitcoin block-space bottlenecks
- Efficient batching and compression for throughput

### 🚀 Unique Value Propositions

**Best of Both Worlds**
- Bitcoin settlement assurances + full EVM compatibility (MetaMask, Hardhat, Remix, etc.)
- Seamless migration of existing dApps with access to BTC liquidity

**Future-Proof Modular Architecture**
- Execution, DA, and settlement evolve independently (faster provers, DA upgrades, conservative Bitcoin base layer)

**BTC-Denominated Economics**
- Protocol revenue accrues in BTC, avoiding perpetual sell-pressure on VIA
- Enables strategic buybacks/revenue-sharing without reflexive tokenomics

**Developer- and Institution-Friendly**
- Familiar Ethereum tooling; clear separation of concerns can simplify risk and compliance reviews

---

*Terminology & accuracy notes*: Via is a **sovereign** rollup that anchors compact state roots to Bitcoin and uses **Celestia** for DA; it does **not** post full transaction data to Bitcoin. “Immediate” finality depends on prover/attestation latency and anchoring cadence. Gas is paid in **BTC on Via** (a bridged/L2-native representation).

Via doesn't just offer another Layer 2 solution. It represents a fundamental advancement in how we scale Bitcoin while preserving its core principles of security, decentralization, and censorship resistance.

## 📚 How to Use This Documentation

This README serves as the main entry point to all documentation. Documents are organized in a logical learning progression, with foundational concepts presented first, followed by more advanced topics. Each document builds upon knowledge introduced in previous documents.

**Legend:**
- 🟢 Beginner - Essential concepts for all users
- 🟡 Intermediate - Deeper understanding for developers and operators
- 🔴 Advanced - Complex topics for system architects and core developers
- ⏱️ Estimated reading time

## 📋 Table of Contents

### 🚀 Getting Started (Start Here)

- [**Architecture Overview**](architecture_overview.md) 🟢 ⏱️ 20 min  
  High-level overview of the Via L2 system architecture, components, and interactions.

- [**Dependencies**](dependencies.md) 🟢 ⏱️ 10 min  
  Overview of external dependencies and libraries used in the Via L2 system.

- [**Design Patterns**](design_patterns.md) 🟢 ⏱️ 15 min  
  Common design patterns used throughout the codebase.

### 🏗️ System Architecture

- [**Execution Environment**](execution_environment.md) 🟡 ⏱️ 25 min  
  Details of the Virtual Machine (VM) that executes transactions and smart contracts.

- [**State Management**](state_management.md) 🟡 ⏱️ 20 min  
  How state is represented, updated, and committed in the Via L2 system.

- [**System Contracts**](system_contracts.md) 🟡 ⏱️ 25 min  
  Core system contracts that provide essential functionality.

- [**Genesis Bootstrapping**](genesis_bootstrapping.md) 🟡 ⏱️ 15 min  
  Process of initializing the system and creating the genesis state.

- [**System Upgrade Process**](system_upgrade_process.md) 🟡 ⏱️ 20 min  
  How protocol upgrades are proposed, approved, and implemented.

### 🔄 Core Components

- [**Sequencer**](sequencer.md) 🟡 ⏱️ 20 min  
  Component responsible for processing transactions and creating L2 blocks.

- [**Prover**](prover.md) 🟡 ⏱️ 25 min  
  System that generates zero-knowledge proofs for state transitions.

- [**Prover Gateway**](prover_gateway.md) 🟡 ⏱️ 15 min  
  Interface between core system and prover subsystem.

- [**Verifier**](verifier.md) 🟡 ⏱️ 20 min
  Component that verifies ZK proofs and ensures system integrity.
  Implements canonical chain gating, duplicate proof‑reveal elimination, and wallet‑update reprocessing; see [verifier.md](verifier.md).

- [**Verifier Network**](verifier_network.md) 🟡 ⏱️ 20 min  
  Distributed network of verifiers that validate proofs and sign withdrawals.

- [**Mempool**](mempool.md) 🟡 ⏱️ 15 min  
  Storage for pending transactions before they are processed by the sequencer.

- [**Bridge**](bridge.md) 🟡 ⏱️ 25 min  
  Mechanism for transferring assets between L1 (Bitcoin) and L2 (Via).

- [**L1 Watcher**](l1_watcher.md) 🟡 ⏱️ 15 min
  Component that monitors the Bitcoin blockchain for relevant events.
  Applies runtime wallet updates (sequencer/governance/bridge) and re‑parses the current window; see [l1_watcher.md](l1_watcher.md).

- [**Celestia Integration**](celestia_integration.md) 🟡 ⏱️ 20 min  
  Integration with Celestia as a data availability layer.

- [**P2P Networking**](p2p_networking.md) 🟡 ⏱️ 15 min  
  Peer-to-peer communication between system components.

- [**RPC API Layer**](rpc_api_layer.md) 🟡 ⏱️ 15 min
  API interfaces for interacting with the Via L2 system.
- [**External Node**](external_node.md) 🟡 ⏱️ 25 min
  Read replica that syncs from the main node and serves API requests independently. It can optionally run the BTC Watch in follower mode (governance execution processing disabled) using [via_bridge], [via_btc_client], [via_btc_watch] blocks; see [external_node.md](external_node.md).

### ⚙️ Configuration and Operations

- [**Configuration**](configuration.md) 🟢 ⏱️ 20 min
  Overview of configuration files, environment variables, and parameters.
  Note: Bridge configuration is split into a dedicated section. The API resolves the active bridge address from the DB (SystemWallets), and deposit helpers require an explicit --bridge-address. See [configuration.md](configuration.md).

- [**Fee Mechanism**](fee_mechanism.md) 🟢 ⏱️ 15 min  
  How transaction fees are calculated and processed.

- [**LoadNext Tool**](loadnext_tool.md) 🟢 ⏱️ 10 min
  Utility for testing and benchmarking the system.

- [**Examples and Utilities**](examples_and_utilities.md) 🟢 ⏱️ 25 min
  Comprehensive examples and utilities for ZK proof verification, fee rate fetching, MuSig2 operations, and development tools.
- Via test utils crate: end‑to‑end helpers for generating chained inscriptions, constructing a test indexer, and wallet update fixtures. See [core/lib/via_test_utils/](https://github.com/vianetwork/via-core/blob/main/core/lib/via_test_utils/)

- [**Workflow Analysis**](workflow_analysis.md) 🟡 ⏱️ 15 min
  Analysis of common workflows and operational patterns.

### 🔐 Security and Cryptography

- [**Key Management**](key_management.md) 🟡 ⏱️ 20 min  
  How cryptographic keys are managed and secured.

- [**Cryptography Primitives**](cryptography_primitives.md) 🟡 ⏱️ 25 min  
  Cryptographic building blocks used in the system.

- [**MuSig2 Implementation**](musig2_implementation.md) 🔴 ⏱️ 30 min  
  Details of the MuSig2 multi-signature scheme implementation.

- [**Tapscript Usage**](tapscript_usage.md) 🔴 ⏱️ 20 min  
  How Bitcoin's Tapscript is used in the system.

### 🧠 Advanced Topics

- [**Batch Proofs**](batch_proofs.md) 🔴 ⏱️ 30 min  
  Technical details of batch proof generation and verification.

- [**ZKProof Handling**](zkproof_handling.md) 🔴 ⏱️ 30 min  
  How zero-knowledge proofs are generated, processed, and verified.

- [**Inscription Interaction**](inscription_interaction.md) 🔴 ⏱️ 25 min  
  How the system interacts with Bitcoin inscriptions.

- [**Withdrawal Finalization**](withdrawal_finalization.md) 🔴 ⏱️ 25 min
  Technical details of the withdrawal finalization process.

- [System wallet governance flows (sequencer, governance, bridge)](inscription_interaction.md) 🔴 ⏱️ 20 min
  Two‑phase governance inscriptions (proposal via witness, execution via OP_RETURN) update the active sequencer, governance and bridge wallets. UpdateBridge references a proposal txid. See watchers’ runtime wallet application in [l1_watcher.md](l1_watcher.md) and [verifier.md](verifier.md); operator walkthroughs and multisig tooling in [/docs/via_guides/upgrade.md](https://github.com/vianetwork/via-core/blob/main/docs/via_guides/upgrade.md) and [/docs/via_guides/musig2.md](https://github.com/vianetwork/via-core/blob/main/docs/via_guides/musig2.md).

## 🆘 Getting Help

If you encounter issues or have questions while working with the Via L2 system, here are resources to help you:

1. **Check the Documentation**: First, ensure you've read the relevant documentation sections. Many common questions are answered in these documents.

2. **GitHub Issues**: For bugs or feature requests, check existing GitHub issues or create a new one in the [Via L2 repository](https://github.com/vianetwork/via-core).

3. **Developer Community**: Join the Via L2 developer community on Discord or Telegram can be found on our site https://buildonvia.org/.

4. **Contributing**: If you'd like to contribute to the documentation or codebase, please refer to the contribution guidelines in the repository.

## 📝 Documentation Conventions

- Code examples are provided in Rust, the primary language of the Via L2 implementation.
- Configuration examples use YAML format.
- Diagrams are created using Mermaid syntax for consistency and maintainability.
- Technical terms are defined when first introduced and collected in a glossary.

## 🔄 Recent Updates

This documentation has been updated to reflect the latest changes in the Via L2 system, including:

### Enhanced Configuration Support
- **Wallet Address Configuration**: New wallet address fields for verifier and BTC sender components
- **Fee Rate Limits**: Network-specific fee rate limits for Bitcoin transactions (Mainnet, Testnet, Regtest)
- **Server Component Selection**: Selective component execution via `--components` CLI flag
- **Enhanced Genesis Configuration**: Updated genesis values and l2_chain_id usage

### Improved Fee Management
- **External API Fallback**: Bitcoin client fallback to external APIs for fee rate estimation
- **Mempool Integration**: Enhanced fee calculation using mempool data via `get_mempool_info()` method
- **Network-Aware Fee Strategies**: Dynamic fee adjustment based on network conditions

### Enhanced API and RPC Layer
- **Wallet-Specific RPC URLs**: Support for Bitcoin RPC URLs with wallet-specific paths
- **Coordinator API Validation**: Stricter cryptographic validation for partial signature submissions
- **Database Query Improvements**: Enhanced L1 batch details queries for better data availability

### Advanced Verifier Features
- **Synchronization Improvements**: Enhanced verifier synchronization with L1 pause mechanism
- **MuSig2 Enhancements**: New partial signature verification utilities and examples
- **Multi-Client Support**: Multiple Bitcoin client instances for different operational roles

### Bridge Enhancements
- **Enhanced Deposit Validation**: Stricter L2 receiver address validation and minimum deposit requirements
- **Simplified Transaction Creation**: Standardized empty calldata for deposit transactions
- **Testnet Bootstrap Support**: Comprehensive testnet configuration and bootstrap transaction IDs

### External Node
- Added External Node component: read-replica that syncs from the main validator node and serves API requests independently. See [external_node.md](external_node.md).

### New Examples and Utilities
- **ZK Proof Verification**: Complete examples for verifying ZK proofs from Data Availability
- **Fee Rate Management**: Advanced fee estimation examples with multiple strategies
- **MuSig2 Operations**: Partial signature verification and session management examples
- **Development Tools**: Testnet bootstrapping utilities and selective component execution

- Bridge config & API
  Active bridge address is resolved from DB (SystemWallets) by via_getBridgeAddress; deposit helpers now require --bridge-address. See [configuration.md](configuration.md).

- System wallet governance flows & CLI
  Sequencer/Governance/Bridge updates use proposal (witness) + execution (OP_RETURN). Multisig subcommands cover upgrade and wallet updates. See [inscription_interaction.md](inscription_interaction.md), [/docs/via_guides/upgrade.md](https://github.com/vianetwork/via-core/blob/main/docs/via_guides/upgrade.md), [/docs/via_guides/musig2.md](https://github.com/vianetwork/via-core/blob/main/docs/via_guides/musig2.md).

- Watchers
  Canonical chain gating, duplicate proof‑reveal checks, and wallet‑update reprocessing documented in [verifier.md](verifier.md) and [l1_watcher.md](l1_watcher.md).

- External Node
  Optional BTC Watch follower mode with [via_bridge], [via_btc_client], [via_btc_watch] blocks; template keys in [external_node.md](external_node.md).

---

* As of right now, this documentation is maintained by a single person. Last updated: August 19, 2025.*