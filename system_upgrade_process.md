# Via L2 Bitcoin ZK-Rollup: System Upgrade Process

This document details the end-to-end process for upgrading the Via L2 Bitcoin ZK-Rollup system, including protocol versions, system contracts, and the bootloader.

## Table of Contents

1. [Introduction](#introduction)
2. [Upgrade Components](#upgrade-components)
3. [Upgrade Inscription](#upgrade-inscription)
4. [Upgrade Process Flow](#upgrade-process-flow)
5. [Verifier Processing](#verifier-processing)
6. [Sequencer Processing](#sequencer-processing)
7. [Protocol Version Management](#protocol-version-management)
8. [Security Considerations](#security-considerations)
9. [Upgrade CLI Tool](#upgrade-cli-tool)
10. [Example Upgrade Workflow](#example-upgrade-workflow)

## Introduction

The Via L2 Bitcoin ZK-Rollup system supports upgrades to its protocol version, system contracts, and bootloader through a secure and transparent process using Bitcoin inscriptions. This process ensures that all network participants can verify and validate upgrades before they are applied.

System upgrades are critical for:
- Fixing bugs and security vulnerabilities
- Implementing new features and optimizations
- Ensuring compatibility with Bitcoin protocol changes
- Improving the overall performance and security of the system

## Upgrade Components

A system upgrade can include changes to the following components:

1. **Protocol Version**: The semantic version (major.minor.patch) that identifies the protocol implementation
2. **Bootloader**: The special program that processes transactions and enforces the rules of the rollup
3. **Default Account**: The implementation of the default account abstraction
4. **System Contracts**: The core contracts that define the rules and functionality of the rollup

Each of these components has a corresponding code hash that uniquely identifies its implementation. When an upgrade is proposed, the new hashes are specified in the upgrade inscription.

## Upgrade Inscription

The upgrade process begins with the creation of a `SystemContractUpgrade` inscription, which contains all the necessary information for the upgrade:

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

This inscription follows the standard Via inscription format:

```
<pubkey> OP_CHECKSIG OP_FALSE OP_IF "via_inscription_protocol" "SystemContractUpgrade" <data...> OP_ENDIF
```

The inscription is created and submitted to the Bitcoin blockchain using the `upgrade_system_contracts` tool provided in the Via L2 codebase.

## Upgrade Process Flow

The upgrade process follows these steps:

1. **Inscription Creation**: An authorized entity creates a `SystemContractUpgrade` inscription with the new protocol version and contract hashes
2. **Bitcoin Submission**: The inscription is submitted to the Bitcoin blockchain through a commit-reveal transaction pair
3. **Inscription Detection**: The Verifier Network and Sequencer detect the upgrade inscription by monitoring the Bitcoin blockchain
4. **Validation**: The inscription is validated to ensure it comes from an authorized source and contains valid data
5. **Database Update**: The upgrade information is stored in the database with an "executed" flag set to false
6. **Activation Planning**: The upgrade is scheduled for activation at a specific block
7. **Activation**: When the activation block is reached, the new contracts are deployed and the protocol version is updated
8. **Verification**: The system verifies that the upgrade was applied correctly

## Verifier Processing

The Verifier Network plays a crucial role in the upgrade process:

1. **Inscription Monitoring**: The `BtcWatchLayer` monitors the Bitcoin blockchain for new inscriptions
2. **Upgrade Detection**: When a `SystemContractUpgrade` inscription is detected, it is passed to the `GovernanceUpgradeProcessor`
3. **Validation**: The processor validates the inscription and extracts the upgrade information
4. **Database Storage**: The upgrade information is stored in the `protocol_versions` table with an "executed" flag set to false
5. **Activation**: When processing a batch with a number greater than or equal to the activation block, the verifier applies the upgrade
6. **Verification**: The verifier ensures that the upgrade was applied correctly by verifying the contract hashes

The `GovernanceUpgradeProcessor` is implemented in the `core/node/via_btc_watch/src/message_processors/governance_upgrade.rs` file and handles the processing of upgrade inscriptions.

## Sequencer Processing

The Sequencer also processes upgrade inscriptions:

1. **Inscription Detection**: The Sequencer monitors the Bitcoin blockchain for upgrade inscriptions
2. **Validation**: The Sequencer validates the inscription and extracts the upgrade information
3. **Database Storage**: The upgrade information is stored in the database
4. **Activation**: When creating a batch with a number greater than or equal to the activation block, the Sequencer applies the upgrade
5. **State Update**: The Sequencer updates the state with the new contract implementations
6. **Protocol Version Update**: The Sequencer updates the protocol version for new batches

The Sequencer ensures that all transactions in a batch are processed according to the correct protocol version.

### Sequential Batch Processing During Upgrades

During protocol upgrades, the system ensures that batches are processed sequentially to maintain the integrity of the state transitions:

1. **Batch Sequence Validation**: The system validates that batches are processed in sequential order by checking both batch numbers and batch hashes:

```rust
// core/node/via_btc_sender/src/aggregator.rs
fn validate_l1_batch_sequence(
    last_committed_l1_batch_opt: Option<ViaBtcL1BlockDetails>,
    ready_for_commit_l1_batches: &[ViaBtcL1BlockDetails],
) -> anyhow::Result<()> {
    let mut all_batches = vec![];
    // The last_committed_l1_batch should be empty only in case of genesis.
    if let Some(last_committed_l1_batch) = last_committed_l1_batch_opt {
        all_batches.extend_from_slice(&[last_committed_l1_batch.clone()]);
    } else {
        if let Some(batch) = ready_for_commit_l1_batches.get(0) {
            if batch.number.0 != 1 {
                anyhow::bail!("Invalid batch after genesis, not sequential")
            }
        }
    }

    all_batches.extend_from_slice(ready_for_commit_l1_batches);

    for i in 1..all_batches.len() {
        let last_batch = &all_batches[i - 1];
        let next_batch = &all_batches[i];

        if last_batch.number + 1 != next_batch.number {
            anyhow::bail!(
                "L1 batches prepared for commit or proof batch numbers are not sequential"
            );
        }
        if last_batch.hash != next_batch.prev_l1_batch_hash {
            anyhow::bail!(
                "L1 batches prepared for commit or proof batch hashes are not sequential"
            );
        }
    }

    Ok(())
}
```

2. **Protocol Version Transition**: When a protocol upgrade occurs, the system first processes all batches with the previous protocol version before switching to the new one:

```rust
// core/node/via_btc_sender/src/aggregator.rs
async fn get_ready_for_commit_l1_batches(
    &self,
    storage: &mut Connection<'_, Core>,
) -> anyhow::Result<Vec<ViaBtcL1BlockDetails>> {
    let protocol_version_id = self.get_last_protocol_version_id(storage).await?;
    let prev_protocol_version_id = self.get_prev_used_protocol_version(storage).await?;

    // In case of a protocol upgrade, we first process the l1 batches created with the previous protocol version
    // then switch to the new one.
    if prev_protocol_version_id != protocol_version_id {
        let prev_base_system_contracts_hashes = self
            .load_base_system_contracts(storage, prev_protocol_version_id)
            .await?;
        let ready_for_commit_l1_batches = storage
            .via_blocks_dal()
            .get_ready_for_commit_l1_batches(
                self.config.max_aggregated_blocks_to_commit() as usize,
                &prev_base_system_contracts_hashes.bootloader,
                &prev_base_system_contracts_hashes.default_aa,
                prev_protocol_version_id,
            )
            .await?;

        if !ready_for_commit_l1_batches.is_empty() {
            return Ok(ready_for_commit_l1_batches);
        }
    }

    // If no batches with the previous protocol version are ready, or if we've processed all of them,
    // then process batches with the current protocol version
    let ready_for_commit_l1_batches = storage
        .via_blocks_dal()
        .get_ready_for_commit_l1_batches(
            self.config.max_aggregated_blocks_to_commit() as usize,
            &base_system_contracts_hashes.bootloader,
            &base_system_contracts_hashes.default_aa,
            protocol_version_id,
        )
        .await?;
    Ok(ready_for_commit_l1_batches)
}
```

This sequential processing ensures that:
- All batches are processed in the correct order
- The state transitions are consistent
- Protocol upgrades are applied cleanly without skipping any batches
- The system maintains a continuous chain of state transitions during upgrades

## Protocol Version Management

The system maintains a history of protocol versions in the database:

1. **Version Storage**: Protocol versions are stored in the `protocol_versions` table
2. **Version Tracking**: Each batch is associated with a specific protocol version
3. **Version Selection**: The appropriate VM version is selected based on the protocol version
4. **Version Compatibility**: The system ensures backward compatibility for older protocol versions
5. **Version Transition**: The system handles transitions between protocol versions by processing batches sequentially

Protocol versions follow semantic versioning (major.minor.patch):
- **Major**: Incompatible API changes
- **Minor**: Backward-compatible functionality additions
- **Patch**: Backward-compatible bug fixes

The protocol version is packed into a `U256` value for efficient storage and transmission.

## Security Considerations

The upgrade process includes several security measures:

1. **Authorization**: Only inscriptions from authorized signers are considered valid
2. **Transparency**: All upgrades are recorded on the Bitcoin blockchain, providing a transparent audit trail
3. **Validation**: The system validates all upgrade information before applying it
4. **Gradual Activation**: Upgrades are activated at a specific block, allowing time for preparation
5. **Verification**: The system verifies that the upgrade was applied correctly
6. **Rollback Prevention**: Once an upgrade is applied, it cannot be rolled back without a new upgrade inscription

## Upgrade CLI Tool

The Via L2 codebase includes a CLI tool for creating upgrade inscriptions:

```rust
// Example:
// cargo run --example upgrade_system_contracts \
//     regtest \
//     http://0.0.0.0:18443 \
//     rpcuser \
//     rpcpassword \
//     cVZduZu265sWeAqFYygoDEE1FZ7wV9rpW5qdqjRkUehjaUMWLT1R \
//     0.26.0 \
//     0x010008e79c154523aa30981e598b73c4a33c304bef9c82bae7d2ca4d21daedc7 \
//     0x010005630848b5537f934eea6bd8c61c50648a162b90c82f316454f4109462b1 \
//     0x0000000000000000000000000000000000008002,... \
//     0x010000758176955a4cb7af4c768cdad127fa6b1d0329ce3e8b37d1382e9c21e6,...
```

This tool takes the following parameters:
1. Bitcoin network (mainnet, testnet, regtest)
2. Bitcoin RPC URL
3. Bitcoin RPC username
4. Bitcoin RPC password
5. Private key for signing the inscription
6. New protocol version
7. New bootloader code hash
8. New default account code hash
9. System contract addresses (comma-separated)
10. System contract hashes (comma-separated)

## Example Upgrade Workflow

Here's an example workflow for upgrading the system:

1. **Prepare New Contracts**: Develop and test the new system contracts, bootloader, and default account
2. **Calculate Hashes**: Calculate the code hashes for the new contracts
3. **Create Upgrade Inscription**: Use the upgrade CLI tool to create an upgrade inscription
4. **Submit to Bitcoin**: Submit the inscription to the Bitcoin blockchain
5. **Monitor Processing**: Monitor the Verifier Network and Sequencer to ensure they detect and process the upgrade
6. **Verify Activation**: Verify that the upgrade is activated at the specified block
7. **Confirm Functionality**: Confirm that the system functions correctly with the new contracts

Example command for creating an upgrade inscription:

```bash
cargo run --example upgrade_system_contracts \
    mainnet \
    https://bitcoin-rpc.example.com \
    username \
    password \
    <private_key> \
    1.2.0 \
    0x<bootloader_hash> \
    0x<default_account_hash> \
    0x8001,0x8002,0x8003,... \
    0x<hash1>,0x<hash2>,0x<hash3>,...
```

---

In summary, the Via L2 Bitcoin ZK-Rollup system provides a secure and transparent process for upgrading its components through Bitcoin inscriptions. This process ensures that all network participants can verify and validate upgrades before they are applied, maintaining the security and integrity of the system. The sequential batch processing during protocol upgrades ensures a smooth transition between protocol versions without compromising the integrity of the state transitions.