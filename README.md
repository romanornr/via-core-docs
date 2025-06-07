# Via L2 Bitcoin ZK-Rollup Documentation

Welcome to the comprehensive documentation for Via, a Bitcoin-native Layer 2 ZK-Rollup built from the ground up specifically for Bitcoin's unique architecture. Unlike traditional Layer 2 solutions adapted from Ethereum, Via leverages a revolutionary three-layer separation of concerns: Bitcoin provides security and settlement, Celestia handles data availability, and Via executes transactions through a high-performance zkEVM environment. This innovative architecture enables Bitcoin-native scaling without compromising on security or decentralization.

Via's breakthrough design centers on Bitcoin-native primitives including Taproot integration, Bitcoin inscriptions for metadata attestation, and a MuSig2 multi-signature bridge for secure withdrawals. Rather than relying on smart contracts, Via employs a distributed verifier network where independent validators attest to system state through Bitcoin inscriptions, creating a truly decentralized verification mechanism with threshold consensus. This approach maintains Bitcoin's security model while enabling high-throughput, low-cost transactions that inherit the full security guarantees of the Bitcoin blockchain.

The system's separation of data availability to Celestia, combined with Bitcoin's role as the settlement layer, creates an unprecedented scaling solution that preserves Bitcoin's core principles while delivering modern Layer 2 capabilities. This documentation covers all aspects of Via's innovative architecture, from its Bitcoin-native bridge mechanisms to its distributed proof verification system.

## Why Choose Via?

Via represents the next evolution in Bitcoin scaling, offering unique advantages over both Ethereum-based solutions and other Bitcoin Layer 2 approaches. Here's why Via stands out as the premier choice for building on Bitcoin.

### 🛡️ Advantages over Ethereum

**True Bitcoin Security**
- Inherits Bitcoin's unmatched Proof-of-Work security model with over 15+ years of battle-tested resilience
- No reliance on Ethereum's newer Proof-of-Stake consensus mechanism
- Benefits from Bitcoin's $1+ trillion security budget and global mining infrastructure

**Superior Decentralization**
- Built on Bitcoin's truly decentralized network with thousands of independent miners worldwide
- Avoids Ethereum's validator concentration risks and staking centralization concerns
- No single points of failure or validator slashing risks

**Proven Censorship Resistance**
- Leverages Bitcoin's unprecedented track record of censorship resistance since 2009
- No exposure to potential regulatory pressures on Ethereum validators
- Maintains Bitcoin's neutral, apolitical monetary properties

**Store of Value Foundation**
- Built on the world's premier digital store of value asset
- No exposure to ETH price volatility for security guarantees
- Aligns with Bitcoin's long-term value proposition and institutional adoption

**No Gas Token Risk**
- Eliminates dependency on ETH for security and operations
- Users transact with Bitcoin, the most liquid and widely accepted cryptocurrency
- Reduces complexity and risk associated with multi-token ecosystems

### ⚡ Advantages over Other Bitcoin L2s

**Cryptographic Finality**
- ZK proofs provide instant mathematical finality vs optimistic rollups' 7-day challenge periods
- No waiting periods for withdrawals or transaction confirmation
- Immediate settlement certainty for users and applications

**Zero Fraud Risk**
- Mathematical proofs eliminate fraud proof assumptions and economic game theory
- No reliance on watchtowers, challengers, or economic incentives for security
- Cryptographic guarantees that are impossible to game or manipulate

**Enhanced Security Model**
- Distributed verifier network eliminates single operator risks common in other L2s
- Multiple independent validators must attest to system state
- Threshold consensus mechanism prevents single points of failure

**Bitcoin-Native Integration**
- Purpose-built for Bitcoin's UTXO model and scripting capabilities
- Native Taproot integration and Bitcoin inscription support
- Not an adapted solution from other blockchain architectures

**Scalable Data Architecture**
- Celestia integration provides unlimited data availability scaling
- Efficient data compression and batching mechanisms
- Avoids Bitcoin block space limitations while maintaining security

### 🚀 Unique Value Propositions

**Best of Both Worlds**
- Combines Bitcoin's unmatched security with Ethereum's rich execution environment
- Full EVM compatibility enables seamless migration of existing dApps
- Access to Bitcoin's liquidity with modern smart contract capabilities

**Future-Proof Architecture**
- Modular design that evolves with Bitcoin protocol upgrades
- Separation of concerns allows independent optimization of each layer
- Ready for Bitcoin's future enhancements like covenants and advanced scripting

**Superior Developer Experience**
- Complete Ethereum tooling compatibility (MetaMask, Hardhat, Remix, etc.)
- Familiar development environment with Bitcoin's security guarantees
- Extensive documentation and developer resources

**Institutional Grade**
- Enterprise-ready infrastructure with Bitcoin's regulatory clarity
- Proven security model trusted by institutions and governments
- Compliance-friendly architecture built on regulated Bitcoin foundation

**Unmatched Scalability**
- Theoretical throughput of 10,000+ TPS with sub-second finality
- Horizontal scaling through data availability layer separation
- Cost-effective transactions while maintaining security guarantees

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

- [**Verifier Network**](verifier_network.md) 🟡 ⏱️ 20 min  
  Distributed network of verifiers that validate proofs and sign withdrawals.

- [**Mempool**](mempool.md) 🟡 ⏱️ 15 min  
  Storage for pending transactions before they are processed by the sequencer.

- [**Bridge**](bridge.md) 🟡 ⏱️ 25 min  
  Mechanism for transferring assets between L1 (Bitcoin) and L2 (Via).

- [**L1 Watcher**](l1_watcher.md) 🟡 ⏱️ 15 min  
  Component that monitors the Bitcoin blockchain for relevant events.

- [**Celestia Integration**](celestia_integration.md) 🟡 ⏱️ 20 min  
  Integration with Celestia as a data availability layer.

- [**P2P Networking**](p2p_networking.md) 🟡 ⏱️ 15 min  
  Peer-to-peer communication between system components.

- [**RPC API Layer**](rpc_api_layer.md) 🟡 ⏱️ 15 min  
  API interfaces for interacting with the Via L2 system.

### ⚙️ Configuration and Operations

- [**Configuration**](configuration.md) 🟢 ⏱️ 20 min  
  Overview of configuration files, environment variables, and parameters.

- [**Fee Mechanism**](fee_mechanism.md) 🟢 ⏱️ 15 min  
  How transaction fees are calculated and processed.

- [**LoadNext Tool**](loadnext_tool.md) 🟢 ⏱️ 10 min
  Utility for testing and benchmarking the system.

- [**Examples and Utilities**](examples_and_utilities.md) 🟢 ⏱️ 25 min
  Comprehensive examples and utilities for ZK proof verification, fee rate fetching, MuSig2 operations, and development tools.

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

### New Examples and Utilities
- **ZK Proof Verification**: Complete examples for verifying ZK proofs from Data Availability
- **Fee Rate Management**: Advanced fee estimation examples with multiple strategies
- **MuSig2 Operations**: Partial signature verification and session management examples
- **Development Tools**: Testnet bootstrapping utilities and selective component execution

---

*This documentation is maintained by the Via L2 team and community contributors. Last updated: June 2025.*