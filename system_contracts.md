# Via L2 Bitcoin ZK-Rollup System Contracts

This document provides a comprehensive overview of the system contracts that define the core logic and rules of the Via L2 Bitcoin ZK-Rollup.

## Table of Contents

1. [Introduction](#introduction)
2. [System Architecture Overview](#system-architecture-overview)
3. [Bootloader](#bootloader)
4. [Core System Contracts](#core-system-contracts)
5. [State Transition Rules](#state-transition-rules)
6. [Interactions with Other Components](#interactions-with-other-components)
7. [Upgrade Mechanisms](#upgrade-mechanisms)

## Introduction

System contracts are privileged smart contracts that form the foundation of the Via L2 Bitcoin ZK-Rollup. They define the core rules and logic of the rollup, including state transitions, account abstraction, fee handling, and interactions with the execution environment. These contracts are deployed at predefined addresses and have special permissions within the system.

The Via L2 system is built on the zkEVM architecture, which consists of three main components:
1. The zkEVM (Virtual Machine)
2. System Contracts (deployed at predefined addresses)
3. The Bootloader (a special program that processes transactions)

## System Architecture Overview

The Via L2 system follows a multi-VM architecture, allowing for different VM versions to coexist. This is implemented through the `multivm` crate, which provides a wrapper over several versions of the VM. The system contracts and bootloader are integral parts of this architecture.

From the VM's perspective, it runs a single program called the bootloader, which internally processes multiple transactions. The bootloader interacts with system contracts to execute transactions and enforce the rules of the rollup.

## Bootloader

The bootloader is a special "quasi" system contract that serves as the entry point for transaction execution. Unlike other system contracts, the bootloader's code is not stored on L2 but is loaded directly by its hash.

### Role and Purpose

The bootloader serves two main purposes:
1. **Batch Processing**: Instead of processing transactions one by one, the bootloader processes an entire batch of transactions as a single program, making the system more efficient.
2. **Transaction Validation**: The bootloader validates transactions, enforces rules, and manages the execution flow.

### Implementation Details

The bootloader is written in Yul and is loaded into memory at a specific address (`BOOTLOADER_ADDRESS = 0x8001`). It has a special heap that acts as an interface between the server and the zkEVM. The server fills the bootloader's heap with transaction data, and the bootloader executes these transactions.

The bootloader's execution flow includes:
1. Reading batch information and initializing the environment
2. Processing transactions one by one:
   - Validating transaction format and parameters
   - Deducting fees
   - Executing the transaction
   - Handling refunds
3. Finalizing the batch and publishing state changes

The bootloader's hash is stored on L1 and can only be changed as part of a system upgrade.

## Core System Contracts

Via L2 includes 25 system contracts that handle various aspects of the rollup's functionality. These contracts are deployed at predefined addresses and have special permissions.

### Key System Contracts

1. **AccountCodeStorage** (`ACCOUNT_CODE_STORAGE_ADDRESS`)
   - Stores and manages account code
   - Provides functions to get and set account code

2. **NonceHolder** (`NONCE_HOLDER_ADDRESS`)
   - Manages account nonces for transactions and deployments
   - Provides `validateNonceUsage` which the bootloader uses to check whether a nonce has been used

3. **KnownCodesStorage** (`KNOWN_CODES_STORAGE_ADDRESS`)
   - Stores known bytecode hashes
   - Used to verify bytecode during contract deployment

4. **ImmutableSimulator** (`IMMUTABLE_SIMULATOR_STORAGE_ADDRESS`)
   - Simulates immutable variables in contracts

5. **ContractDeployer** (`CONTRACT_DEPLOYER_ADDRESS`)
   - Handles contract deployment logic
   - Manages deployment fees and validation

6. **L1Messenger** (`L1_MESSENGER_ADDRESS`)
   - Facilitates communication from L2 to L1
   - Manages L2→L1 logs and messages

7. **MsgValueSimulator** (`MSG_VALUE_SIMULATOR_ADDRESS`)
   - Simulates BTC transfers since EraVM doesn't support passing Bitcoin natively

8. **L2BaseToken** (`L2_BASE_TOKEN_ADDRESS`)
   - Implementation of the L2 native token
   - Handles withdrawals to L1

9. **SystemContext** (`SYSTEM_CONTEXT_ADDRESS`)
   - Provides environmental data like block number, timestamp, etc.
   - Manages L2 block information

10. **EventWriter** (`EVENT_WRITER_ADDRESS`)
    - Handles event emission
    - Stores event payloads

11. **Precompiles**
    - Various precompiled contracts for cryptographic operations:
      - `Keccak256` (`KECCAK256_PRECOMPILE_ADDRESS`)
      - `SHA256` (`SHA256_PRECOMPILE_ADDRESS`)
      - `Ecrecover` (`ECRECOVER_PRECOMPILE_ADDRESS`)
      - `EcAdd` (`EC_ADD_PRECOMPILE_ADDRESS`)
      - `EcMul` (`EC_MUL_PRECOMPILE_ADDRESS`)
      - `EcPairing` (`EC_PAIRING_PRECOMPILE_ADDRESS`)
      - `P256Verify` (`P256VERIFY_PRECOMPILE_ADDRESS`)
      - `CodeOracle` (`CODE_ORACLE_ADDRESS`)

12. **BootloaderUtilities** (`BOOTLOADER_UTILITIES_ADDRESS`)
    - Contains methods needed for bootloader functionality
    - Moved out from the bootloader for convenience

13. **Compressor** (`COMPRESSOR_ADDRESS`)
    - Handles bytecode compression
    - Validates compressed bytecode before sending to L1

14. **ComplexUpgrader** (`COMPLEX_UPGRADER_ADDRESS`)
    - Manages complex upgrade processes

15. **PubdataChunkPublisher** (`PUBDATA_CHUNK_PUBLISHER_ADDRESS`)
    - Publishes pubdata chunks to L1

16. **Create2Factory** (`CREATE2_FACTORY_ADDRESS`)
    - Implements the CREATE2 opcode functionality

### Default Account

The system also includes a `DefaultAccount` contract that implements the default account behavior. This contract is not explicitly listed in the system contracts array but is loaded as part of the base system contracts. It provides the default account abstraction implementation.

## State Transition Rules

The state transition function in Via L2 is primarily defined by the interaction between the bootloader and system contracts. This section outlines how state transitions are defined and enforced.

### Transaction Processing

1. **Transaction Validation**:
   - The bootloader validates transaction format and parameters
   - For account abstraction, the account's validation method is called
   - Nonces are validated through the NonceHolder contract

2. **State Changes**:
   - Transactions modify state through storage writes
   - The VM tracks all storage changes
   - System contracts enforce rules on state changes (e.g., nonce increments)

3. **Fee Handling**:
   - The bootloader deducts fees upfront
   - Refunds are processed after transaction execution
   - Fee distribution is handled at the end of batch processing

### L2 Block Management

The SystemContext contract manages L2 block information, including:
- Block number and timestamp
- Previous block hash
- Transaction hashes within the block

The bootloader interacts with SystemContext to:
- Set L2 block information before processing transactions
- Append transaction hashes to the current L2 block
- Create new L2 blocks as needed

## Interactions with Other Components

System contracts interact with various components of the Via L2 system:

### Interaction with VM

- The VM executes the bootloader, which in turn interacts with system contracts
- System contracts use special opcodes (via SystemContractHelper) to access VM functionality
- The VM enforces that only the bootloader can call certain system contract functions

### Interaction with Sequencer/State Keeper

- The Sequencer builds batches of transactions and feeds them to the bootloader
- The State Keeper tracks state changes and ensures consistency
- System contracts provide interfaces for the Sequencer to manage blocks and transactions

### Interaction with Prover

- The Prover generates zero-knowledge proofs based on the execution trace
- System contracts (especially the bootloader) are designed to be efficiently provable
- The bootloader's hash is included in the block header for verification

### Interaction with L1

- The L1Messenger contract facilitates communication from L2 to L1
- Priority operations from L1 are processed by the bootloader
- The bootloader maintains a rolling hash of priority operations for verification on L1

## Upgrade Mechanisms

Via L2 includes mechanisms for upgrading system contracts and the bootloader through Bitcoin inscriptions, providing a secure and transparent upgrade process.

### System Contract Upgrade Inscription

The primary mechanism for upgrading system contracts is through a specialized Bitcoin inscription type called `SystemContractUpgrade`. This inscription contains all the necessary information to upgrade the protocol version, bootloader, default account, and system contracts:

```rust
pub struct SystemContractUpgradeInput {
    /// New protocol version ID.
    pub version: ProtocolSemanticVersion,
    /// New bootloader code hash.
    pub bootloader_code_hash: H256,
    /// New default account code hash.
    pub default_account_code_hash: H256,
    /// The system contracts to upgrade (address and new hash pairs).
    pub system_contracts: Vec<(EVMAddress, H256)>,
}
```

The upgrade process follows these steps:
1. An upgrade inscription is created with the new protocol version and contract hashes
2. The inscription is submitted to the Bitcoin blockchain
3. The Verifier Network and Sequencer detect and validate the upgrade inscription
4. The upgrade is scheduled for activation at a specific block
5. When the activation block is reached, the new contracts are deployed and the protocol version is updated

### Base System Contracts Upgrade

The base system contracts (bootloader and default account) are upgraded through the `SystemContractUpgrade` inscription:
1. The new bootloader code hash and default account code hash are specified in the inscription
2. These hashes are verified against the expected values
3. The new contracts are activated as part of the protocol upgrade

### System Contract Upgrades

Individual system contracts can be upgraded through:
1. The `SystemContractUpgrade` inscription for coordinated upgrades of multiple contracts
2. The ComplexUpgrader contract for complex upgrades that require special handling
3. Direct upgrades through governance for simpler changes

### Protocol Version Management

The system supports multiple protocol versions through the multiVM architecture:
- Each version has its own set of base system contracts
- The appropriate VM version is selected based on the protocol version
- Protocol versions follow semantic versioning (major.minor.patch)
- The protocol version is stored in the database and tracked for each batch
- This allows for backward compatibility and smooth upgrades

### Upgrade Verification and Security

The upgrade process includes several security measures:
1. Only authorized signers can create valid upgrade inscriptions
2. The Verifier Network validates the upgrade before it is applied
3. The upgrade process is transparent and can be audited on the Bitcoin blockchain
4. The system maintains a history of protocol versions for verification purposes