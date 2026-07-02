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

The bootloader is written in Yul and has a formal address defined in via-core:

```rust
// core/lib/constants/src/contracts.rs
pub const BOOTLOADER_ADDRESS: Address = H160([
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x80, 0x01,
]);
```

That is, `0x0000000000000000000000000000000000008001`. It has a special heap that acts as an interface between the server and the zkEVM. The server fills the bootloader's heap with transaction data, and the bootloader executes these transactions.

The bootloader's execution flow includes:
1. Reading batch information and initializing the environment
2. Processing transactions one by one:
   - Validating transaction format and parameters
   - Deducting fees
   - Executing the transaction
   - Handling refunds
3. Finalizing the batch and publishing state changes

Via settles to Bitcoin, so there is no Ethereum L1 contract holding the bootloader hash. Instead, the bootloader hash (and the default account hash) are committed on Bitcoin: at genesis they arrive in the `SystemBootstrapping` inscription, and afterwards they can only change through a protocol upgrade inscription (see [Upgrade Mechanisms](#upgrade-mechanisms)):

```rust
// core/lib/via_btc_client/src/types.rs
pub struct SystemBootstrappingInput {
    pub start_block_height: u32,
    pub protocol_version: ProtocolSemanticVersion,
    pub bootloader_hash: H256,
    pub abstract_account_hash: H256,
    pub snark_wrapper_vk_hash: H256,
    pub evm_emulator_hash: H256,
    pub governance_address: BitcoinAddress<NetworkUnchecked>,
    pub sequencer_address: BitcoinAddress<NetworkUnchecked>,
    pub bridge_musig2_address: BitcoinAddress<NetworkUnchecked>,
    pub verifier_p2wpkh_addresses: Vec<BitcoinAddress<NetworkUnchecked>>,
}
```

## Core System Contracts

The canonical list of system contracts lives in via-core at `core/lib/types/src/system_contracts.rs`:

```rust
// core/lib/types/src/system_contracts.rs
static SYSTEM_CONTRACT_LIST: [(&str, &str, Address, ContractLanguage); 26] = [
```

The array has 26 entries (including two `EmptyContract` placeholders for the zero address and the bootloader address). Genesis deploys them via:

```rust
// core/lib/types/src/system_contracts.rs
/// Gets default set of system contracts, based on Cargo workspace location.
pub fn get_system_smart_contracts(use_evm_emulator: bool) -> Vec<DeployedContract> {
    SYSTEM_CONTRACT_LIST
        .iter()
        .filter_map(|(path, name, address, contract_lang)| {
            if *name == "EvmGasManager" && !use_evm_emulator {
                None
            } else {
                Some(DeployedContract {
                    account_id: AccountTreeId::new(*address),
                    bytecode: read_sys_contract_bytecode(path, name, contract_lang.clone()),
                })
            }
        })
        .collect()
}
```

The Solidity/Yul sources of these contracts are not part of via-core itself. They live in the `contracts` git submodule, which `.gitmodules` pins to `https://github.com/vianetwork/era-contracts.git` (a fork of Matter Labs' era-contracts; the sources are under `system-contracts/contracts/` in that repo).

### Address Table

Every entry below is taken from `SYSTEM_CONTRACT_LIST` in `core/lib/types/src/system_contracts.rs`, with the address values resolved from the constants in `core/lib/constants/src/contracts.rs`:

| Contract | Address constant | Address | Language |
|---|---|---|---|
| AccountCodeStorage | `ACCOUNT_CODE_STORAGE_ADDRESS` | `0x0000000000000000000000000000000000008002` | Sol |
| NonceHolder | `NONCE_HOLDER_ADDRESS` | `0x0000000000000000000000000000000000008003` | Sol |
| KnownCodesStorage | `KNOWN_CODES_STORAGE_ADDRESS` | `0x0000000000000000000000000000000000008004` | Sol |
| ImmutableSimulator | `IMMUTABLE_SIMULATOR_STORAGE_ADDRESS` | `0x0000000000000000000000000000000000008005` | Sol |
| ContractDeployer | `CONTRACT_DEPLOYER_ADDRESS` | `0x0000000000000000000000000000000000008006` | Sol |
| L1Messenger | `L1_MESSENGER_ADDRESS` | `0x0000000000000000000000000000000000008008` | Sol |
| MsgValueSimulator | `MSG_VALUE_SIMULATOR_ADDRESS` | `0x0000000000000000000000000000000000008009` | Sol |
| L2BaseToken | `L2_BASE_TOKEN_ADDRESS` | `0x000000000000000000000000000000000000800a` | Sol |
| Keccak256 (precompile) | `KECCAK256_PRECOMPILE_ADDRESS` | `0x0000000000000000000000000000000000008010` | Yul |
| SHA256 (precompile) | `SHA256_PRECOMPILE_ADDRESS` | `0x0000000000000000000000000000000000000002` | Yul |
| Ecrecover (precompile) | `ECRECOVER_PRECOMPILE_ADDRESS` | `0x0000000000000000000000000000000000000001` | Yul |
| EcAdd (precompile) | `EC_ADD_PRECOMPILE_ADDRESS` | `0x0000000000000000000000000000000000000006` | Yul |
| EcMul (precompile) | `EC_MUL_PRECOMPILE_ADDRESS` | `0x0000000000000000000000000000000000000007` | Yul |
| EcPairing (precompile) | `EC_PAIRING_PRECOMPILE_ADDRESS` | `0x0000000000000000000000000000000000000008` | Yul |
| P256Verify (precompile) | `P256VERIFY_PRECOMPILE_ADDRESS` | `0x0000000000000000000000000000000000000100` | Yul |
| CodeOracle (precompile) | `CODE_ORACLE_ADDRESS` | `0x0000000000000000000000000000000000008012` | Yul |
| SystemContext | `SYSTEM_CONTEXT_ADDRESS` | `0x000000000000000000000000000000000000800b` | Sol |
| EventWriter | `EVENT_WRITER_ADDRESS` | `0x000000000000000000000000000000000000800d` | Yul |
| BootloaderUtilities | `BOOTLOADER_UTILITIES_ADDRESS` | `0x000000000000000000000000000000000000800c` | Sol |
| Compressor | `COMPRESSOR_ADDRESS` | `0x000000000000000000000000000000000000800e` | Sol |
| ComplexUpgrader | `COMPLEX_UPGRADER_ADDRESS` | `0x000000000000000000000000000000000000800f` | Sol |
| EvmGasManager | `EVM_GAS_MANAGER_ADDRESS` | `0x0000000000000000000000000000000000008013` | Yul |
| EmptyContract | `Address::zero()` | `0x0000000000000000000000000000000000000000` | Sol |
| EmptyContract | `BOOTLOADER_ADDRESS` | `0x0000000000000000000000000000000000008001` | Sol |
| PubdataChunkPublisher | `PUBDATA_CHUNK_PUBLISHER_ADDRESS` | `0x0000000000000000000000000000000000008011` | Sol |
| Create2Factory | `CREATE2_FACTORY_ADDRESS` | `0x0000000000000000000000000000000000010000` | Sol |

`EvmGasManager` is only deployed when the EVM emulator is enabled (see the `use_evm_emulator` filter above). On `Create2Factory`, `core/lib/constants/src/contracts.rs` notes:

```rust
// core/lib/constants/src/contracts.rs
/// Note, that the `Create2Factory` and higher are explicitly deployed on a non-system-contract address
/// as they don't require any kernel space features.
pub const CREATE2_FACTORY_ADDRESS: Address = H160([
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x00, 0x01, 0x00, 0x00,
]);
```

### Key System Contracts

1. **AccountCodeStorage** (`ACCOUNT_CODE_STORAGE_ADDRESS`)
   - Stores and manages account code
   - Provides functions to get and set account code

2. **NonceHolder** (`NONCE_HOLDER_ADDRESS`)
   - Manages account nonces for transactions and deployments
   - Provides `validateNonceUsage` which the bootloader uses to check whether a nonce has been used (declared in `system-contracts/contracts/NonceHolder.sol` of the era-contracts fork)
   - via-core mirrors the nonce packing scheme in `core/lib/types/src/system_contracts.rs`:

   ```rust
   // core/lib/types/src/system_contracts.rs
   // Note, that in the `NONCE_HOLDER_ADDRESS` storage the nonces of accounts
   // are stored in the following form:
   // `2^128 * deployment_nonce + tx_nonce`,
   // where `tx_nonce` should be number of transactions, the account has processed
   // and the `deployment_nonce` should be the number of contracts.
   pub const TX_NONCE_INCREMENT: U256 = U256([1, 0, 0, 0]); // 1
   pub const DEPLOYMENT_NONCE_INCREMENT: U256 = U256([0, 0, 1, 0]); // 2^128
   ```

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
   - Implementation of the L2 native token, which on Via represents BTC
   - Deposits mint the base token to the recipient; withdrawals burn it and emit an L2 to L1 message that the verifier network parses (`via_verifier/lib/via_withdrawal_client/src/withdraw.rs`)

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

The system also includes a `DefaultAccount` contract that implements the default account behavior. This contract is not explicitly listed in `SYSTEM_CONTRACT_LIST` but is loaded as part of the base system contracts. Its Solidity source is `system-contracts/contracts/DefaultAccount.sol` in the era-contracts fork, and its code hash is what the `SystemBootstrapping` inscription carries as `abstract_account_hash` (see the struct above).

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

On Via, the L1 is Bitcoin; there are no Ethereum L1 contracts.

- The L1Messenger contract facilitates communication from L2 to L1 (for example, withdrawal messages that the verifier network parses off-chain)
- Deposits arrive as `L1ToL2Message` inscriptions on Bitcoin and are converted by `via_btc_watch` into priority operations (`L1Tx`, see `core/node/via_btc_watch/src/message_processors/l1_to_l2.rs`), which the bootloader then processes
- The bootloader maintains a rolling hash of priority operations, which is part of the batch commitment

## Upgrade Mechanisms

Via L2 includes mechanisms for upgrading system contracts and the bootloader through Bitcoin inscriptions, providing a secure and transparent upgrade process.

### Upgrade Inscriptions: Proposal and Execution

Upgrades are a two-phase inscription flow, defined in `core/lib/via_btc_client/src/types.rs`. First, a `SystemContractUpgradeProposal` inscription carries the actual upgrade payload (protocol version, bootloader hash, default account hash, EVM emulator hash, verifier key hash, and the list of system contracts to force-deploy):

```rust
// core/lib/via_btc_client/src/types.rs
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
pub struct SystemContractUpgradeProposalInput {
    /// New protocol version ID.
    pub version: ProtocolSemanticVersion,
    /// New bootloader code hash.
    pub bootloader_code_hash: H256,
    /// New default account code hash.
    pub default_account_code_hash: H256,
    /// New EVM emulator code hash
    pub evm_emulator_code_hash: Option<H256>,
    /// Verifier key hash.
    pub recursion_scheduler_level_vk_hash: H256,
    /// The L2 transaction calldata.
    pub system_contracts: Vec<(EVMAddress, H256)>,
}

#[derive(Clone, Debug, PartialEq)]
pub struct SystemContractUpgradeProposal {
    pub common: CommonFields,
    pub input: SystemContractUpgradeProposalInput,
}
```

Second, a separate `SystemContractUpgrade` inscription executes a previously proposed upgrade. It does not carry the upgrade data itself; it only references the proposal transaction by txid, and its inputs are used to prove it was sent by governance:

```rust
// core/lib/via_btc_client/src/types.rs
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
pub struct SystemContractUpgradeInput {
    /// The input utxos.
    pub inputs: Vec<OutPoint>,
    /// The proposal tx_id.
    pub proposal_tx_id: Txid,
}

#[derive(Clone, Debug, PartialEq)]
pub struct SystemContractUpgrade {
    pub common: CommonFields,
    pub input: SystemContractUpgradeInput,
}
```

### Governance Authorization

The indexer only accepts a `SystemContractUpgrade` message if its first input UTXO is spent from the governance address that was set in the `SystemBootstrapping` inscription:

```rust
// core/lib/via_btc_client/src/indexer/mod.rs
async fn is_valid_gov_message(&self, message: &FullInscriptionMessage) -> bool {
    let maybe_input = match message {
        FullInscriptionMessage::SystemContractUpgrade(m) => m.input.inputs.first(),
        FullInscriptionMessage::UpdateBridge(m) => m.input.inputs.first(),
        FullInscriptionMessage::UpdateSequencer(m) => m.input.inputs.first(),
        FullInscriptionMessage::UpdateGovernance(m) => m.input.inputs.first(),
        _ => return false,
    };

    self.is_valid_gov_upgrade(maybe_input)
        .await
        .unwrap_or(false)
}
```

```rust
// core/lib/via_btc_client/src/indexer/mod.rs
#[instrument(skip(self, outpoint_opt), target = "bitcoin_indexer")]
async fn is_valid_gov_upgrade(&self, outpoint_opt: Option<&OutPoint>) -> anyhow::Result<bool> {
    if let Some(outpoint) = outpoint_opt {
        let tx = self.client.get_transaction(&outpoint.txid).await?;
        if let Some(txout) = tx.output.get(outpoint.vout as usize) {
            return Ok(txout.script_pubkey == self.wallets.governance.script_pubkey());
        }
    }
    Ok(false)
}
```

### Applying the Upgrade

The btc_watch component's `GovernanceUpgradesEventProcessor` (`core/node/via_btc_watch/src/message_processors/governance_upgrade.rs`) handles `SystemContractUpgrade` messages: it fetches the referenced proposal transaction from Bitcoin, re-parses it into a `SystemContractUpgradeProposal`, skips versions at or below the last seen protocol version, and turns the payload into a `ProtocolUpgrade` that is saved to the database:

```rust
// core/node/via_btc_watch/src/message_processors/governance_upgrade.rs
let tx = self.upgrade.create_protocol_upgrade_tx(
    system_contract_upgrade_proposal_msg.input.version,
    system_contract_upgrade_proposal_msg.input.system_contracts,
)?;

let upgrade = ProtocolUpgrade {
    version: system_contract_upgrade_proposal_msg.input.version,
    bootloader_code_hash: Some(
        system_contract_upgrade_proposal_msg
            .input
            .bootloader_code_hash,
    ),
    default_account_code_hash: Some(
        system_contract_upgrade_proposal_msg
            .input
            .default_account_code_hash,
    ),
    evm_emulator_code_hash: system_contract_upgrade_proposal_msg
        .input
        .evm_emulator_code_hash,
    tx: Some(tx),
    timestamp: 0,
    verifier_address: None,
    verifier_params: None,
};
```

If the version is newer than the latest one in the database, the processor applies it and persists the new protocol version (`apply_upgrade(...)` followed by `save_protocol_version_with_tx(...)`).

The individual system contracts are replaced by an L2 protocol upgrade transaction built in `core/lib/types/src/via_protocol_upgrade.rs`. It is a `PROTOCOL_UPGRADE_TX_TYPE` transaction sent from the force deployer to the `ContractDeployer`, whose calldata calls `forceDeployOnAddresses` with the (address, bytecode hash) pairs from the proposal:

```rust
// core/lib/types/src/via_protocol_upgrade.rs
pub fn create_protocol_upgrade_tx(
    &self,
    version: ProtocolSemanticVersion,
    system_contracts: Vec<(Address, H256)>,
) -> anyhow::Result<ProtocolUpgradeTx> {
    let canonical_tx_hash = self.get_canonical_tx_hash(version, system_contracts.clone())?;

    let tx = ProtocolUpgradeTx {
        execute: Execute {
            contract_address: Some(CONTRACT_DEPLOYER_ADDRESS),
            calldata: self.get_calldata(system_contracts)?,
            value: U256::zero(),
            factory_deps: vec![],
        },
        common_data: ProtocolUpgradeTxCommonData {
            sender: CONTRACT_FORCE_DEPLOYER_ADDRESS,
            upgrade_id: version.minor,
            max_fee_per_gas: U256::zero(),
            gas_limit: U256::from(GAS_LIMIT),
            gas_per_pubdata_limit: U256::from(GAS_PER_PUB_DATA_BYTE_LIMIT),
            eth_block: 0,
            canonical_tx_hash,
            to_mint: U256::zero(),
            refund_recipient: Address::zero(),
        },
        received_timestamp_ms: unix_timestamp_ms(),
    };

    Ok(tx)
}
```

```rust
// core/lib/types/src/via_protocol_upgrade.rs
fn get_calldata(&self, system_contracts: Vec<(Address, H256)>) -> anyhow::Result<Vec<u8>> {
    let encoded_deployments: Vec<_> = system_contracts
        .into_iter()
        .map(|(address, bytecode_hash)| {
            Token::Tuple(vec![
                Token::FixedBytes(bytecode_hash.as_bytes().to_vec()),
                Token::Address(address),
                Token::Bool(false),
                Token::Uint(U256::zero()),
                Token::Bytes(vec![]),
            ])
        })
        .collect();

    let args = encode(&[Token::Array(encoded_deployments)]);

    // Function selector
    let selector =
        &keccak256(b"forceDeployOnAddresses((bytes32,address,bool,uint256,bytes)[])")[0..4];

    // Concatenate selector + encoded args
    let mut calldata = selector.to_vec();
    calldata.extend_from_slice(&args);

    Ok(calldata)
}
```

The base system contracts (bootloader and default account) are not force-deployed this way; their new code hashes are recorded on the `ProtocolUpgrade` (`bootloader_code_hash`, `default_account_code_hash`) and take effect through the protocol version stored in the database.

### Protocol Version Management

The system supports multiple protocol versions through the multiVM architecture:
- Each version has its own set of base system contracts
- The appropriate VM version is selected based on the protocol version
- Protocol versions follow semantic versioning (major.minor.patch)
- The protocol version is stored in the database and tracked for each batch
- This allows for backward compatibility and smooth upgrades

### Upgrade Verification and Security

The upgrade process includes several security measures, each grounded in code cited above:
1. Governance authorization: the indexer rejects a `SystemContractUpgrade` inscription unless its first input UTXO is spent from the governance address (`is_valid_gov_upgrade` in `core/lib/via_btc_client/src/indexer/mod.rs`)
2. Version monotonicity: `GovernanceUpgradesEventProcessor` skips proposals whose version is at or below the last seen protocol version, and only persists an upgrade if its version is greater than the latest semantic version in the database
3. Transparency: both the proposal and the execution inscription are on the Bitcoin blockchain and can be audited by anyone
4. History: protocol versions are stored via `protocol_versions_dal` (`save_protocol_version_with_tx`), so the system maintains a queryable history of versions