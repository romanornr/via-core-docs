# Via L2 Bitcoin ZK-Rollup: System Upgrade Process

This document details the end-to-end process for upgrading the Via L2 Bitcoin ZK-Rollup system, including protocol versions, system contracts, the bootloader, and the default account. All code below is copied verbatim from the via-core repository.

## Table of Contents

1. [Introduction](#introduction)
2. [Two-Phase Upgrade Design](#two-phase-upgrade-design)
3. [Upgrade Message Types](#upgrade-message-types)
4. [Phase 1: The Upgrade Proposal Inscription](#phase-1-the-upgrade-proposal-inscription)
5. [Phase 2: Governance Execution via OP_RETURN](#phase-2-governance-execution-via-op_return)
6. [Indexer Detection and Governance Validation](#indexer-detection-and-governance-validation)
7. [Sequencer Processing](#sequencer-processing)
8. [Activation at the Batch Boundary (State Keeper)](#activation-at-the-batch-boundary-state-keeper)
9. [Sequential Batch Commitment During Upgrades](#sequential-batch-commitment-during-upgrades)
10. [Verifier Processing](#verifier-processing)
11. [Protocol Version Management](#protocol-version-management)
12. [Security Considerations](#security-considerations)
13. [Upgrade Tooling and Example Workflow](#upgrade-tooling-and-example-workflow)

## Introduction

The Via L2 Bitcoin ZK-Rollup system supports upgrades to its protocol version, system contracts, bootloader, and default account through a secure and transparent process using Bitcoin transactions. Every upgrade is recorded on the Bitcoin blockchain, so all network participants can verify and validate upgrades before they are applied.

System upgrades are critical for:
- Fixing bugs and security vulnerabilities
- Implementing new features and optimizations
- Ensuring compatibility with Bitcoin protocol changes
- Improving the overall performance and security of the system

## Two-Phase Upgrade Design

An upgrade is not a single inscription. The flow is split into two on-chain steps, as documented in the operator guide `docs/via_guides/upgrade.md` in via-core:

> The VIA upgrade flow is split in 2 main parts:
>
> 1. The upgrade proposal inscription.
> 2. The Governance proposal execution inscription.

1. **Upgrade proposal (any wallet)**: a taproot inscription carrying a `SystemContractUpgradeProposalInput` payload (new version, bootloader hash, default account hash, verifier key hash, system contract address/hash pairs). Creating a proposal does not require any privileged key. By itself the proposal has no effect.
2. **Governance execution (governance multisig)**: a separate Bitcoin transaction that spends a governance-wallet UTXO and carries an `OP_RETURN` output with the prefix `VIA_PROTOCOL:UPGRADE` followed by the txid of the proposal transaction. Only this step authorizes the upgrade, because the indexer verifies that the spent input belongs to the governance wallet.

Nodes that see the execution transaction fetch the referenced proposal transaction from Bitcoin, re-parse its payload, and apply the upgrade.

## Upgrade Message Types

The payload structs are defined in `core/lib/via_btc_client/src/types.rs`:

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

Note the split: `SystemContractUpgradeProposalInput` carries the actual upgrade content, while `SystemContractUpgradeInput` (the execution message) carries only the spent UTXOs and a pointer to the proposal transaction.

Both appear in the message enums. Governance execution messages are sorted last so the indexer always applies the latest wallet state first:

```rust
// core/lib/via_btc_client/src/types.rs
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
pub enum InscriptionMessage {
    L1BatchDAReference(L1BatchDAReferenceInput),
    ProofDAReference(ProofDAReferenceInput),
    ValidatorAttestation(ValidatorAttestationInput),
    SystemBootstrapping(SystemBootstrappingInput),
    L1ToL2Message(L1ToL2MessageInput),
    SystemContractUpgradeProposal(SystemContractUpgradeProposalInput),
    UpdateBridgeProposal(UpdateBridgeProposalInput),
}
```

```rust
// core/lib/via_btc_client/src/types.rs
impl FullInscriptionMessage {
    /// Return a numeric sort key that reflects the enum declaration order
    fn order_key(&self) -> usize {
        match self {
            FullInscriptionMessage::L1BatchDAReference(_) => 0,
            FullInscriptionMessage::ProofDAReference(_) => 1,
            FullInscriptionMessage::ValidatorAttestation(_) => 2,
            FullInscriptionMessage::SystemBootstrapping(_) => 3,
            FullInscriptionMessage::L1ToL2Message(_) => 4,
            FullInscriptionMessage::SystemContractUpgradeProposal(_) => 5,
            FullInscriptionMessage::BridgeWithdrawal(_) => 6,
            FullInscriptionMessage::UpdateBridgeProposal(_) => 7,

            // System inscriptions must be ordered to ensure the indexer always uses the latest wallet state.
            FullInscriptionMessage::UpdateGovernance(_) => 8,
            FullInscriptionMessage::UpdateSequencer(_) => 9,
            FullInscriptionMessage::SystemContractUpgrade(_) => 10,
            FullInscriptionMessage::UpdateBridge(_) => 11,
        }
    }
    ...
}
```

The wire tag for the proposal inscription is `SystemContractUpgradeProposal` (not `SystemContractUpgrade`):

```rust
// core/lib/via_btc_client/src/types.rs
lazy_static! {
    pub static ref SYSTEM_BOOTSTRAPPING_MSG: PushBytesBuf =
        PushBytesBuf::from(b"SystemBootstrappingMessage");
    pub static ref PROPOSE_SEQUENCER_MSG: PushBytesBuf =
        PushBytesBuf::from(b"ProposeSequencerMessage");
    pub static ref VALIDATOR_ATTESTATION_MSG: PushBytesBuf =
        PushBytesBuf::from(b"ValidatorAttestationMessage");
    pub static ref L1_BATCH_DA_REFERENCE_MSG: PushBytesBuf =
        PushBytesBuf::from(b"L1BatchDAReferenceMessage");
    pub static ref PROOF_DA_REFERENCE_MSG: PushBytesBuf =
        PushBytesBuf::from(b"ProofDAReferenceMessage");
    pub static ref L1_TO_L2_MSG: PushBytesBuf = PushBytesBuf::from(b"L1ToL2Message");
    pub static ref SYSTEM_CONTRACT_UPGRADE_MSG: PushBytesBuf =
        PushBytesBuf::from(b"SystemContractUpgradeProposal");
    pub static ref UPGRADE_BRIDGE_MSG: PushBytesBuf = PushBytesBuf::from(b"UpgradeBridgeProposal");
}
pub(crate) const VIA_INSCRIPTION_PROTOCOL: &str = "via_inscription_protocol";
```

## Phase 1: The Upgrade Proposal Inscription

The proposal uses the standard Via taproot inscription envelope, built by the script builder:

```rust
// core/lib/via_btc_client/src/inscriber/script_builder.rs
fn build_basic_inscription_script(encoded_pubkey: &PushBytesBuf) -> Result<ScriptBuilder> {
    debug!("Building basic inscription script");
    let mut via_prefix_encoded =
        PushBytesBuf::with_capacity(types::VIA_INSCRIPTION_PROTOCOL.len());
    via_prefix_encoded
        .extend_from_slice(types::VIA_INSCRIPTION_PROTOCOL.as_bytes())
        .ok();

    let script = ScriptBuilder::new()
        .push_slice(encoded_pubkey.as_push_bytes())
        .push_opcode(all::OP_CHECKSIG)
        .push_opcode(OP_FALSE)
        .push_opcode(all::OP_IF)
        .push_slice(via_prefix_encoded);

    debug!("Basic inscription script built");
    Ok(script)
}
```

The upgrade proposal payload is appended field by field after the `SystemContractUpgradeProposal` tag:

```rust
// core/lib/via_btc_client/src/inscriber/script_builder.rs
fn build_system_contract_upgrade_message_script(
    basic_script: ScriptBuilder,
    input: &types::SystemContractUpgradeProposalInput,
) -> ScriptBuilder {
    debug!("Building SystemContract script");

    let version_encoded =
        Self::encode_push_bytes(H256::from_uint(&input.version.pack()).as_bytes());
    let bootloader_code_hash_encoded =
        Self::encode_push_bytes(input.bootloader_code_hash.as_bytes());
    let default_account_code_hash_encoded =
        Self::encode_push_bytes(input.default_account_code_hash.as_bytes());
    let recursion_scheduler_level_vk_hash_encoded =
        Self::encode_push_bytes(input.recursion_scheduler_level_vk_hash.as_bytes());

    let mut basic_script = basic_script;
    basic_script = basic_script
        .push_slice(&*types::SYSTEM_CONTRACT_UPGRADE_MSG)
        .push_slice(version_encoded)
        .push_slice(bootloader_code_hash_encoded)
        .push_slice(default_account_code_hash_encoded)
        .push_slice(recursion_scheduler_level_vk_hash_encoded);

    for (address, hash) in &input.system_contracts {
        basic_script = basic_script.push_slice(Self::encode_push_bytes(address.as_bytes()));
        basic_script = basic_script.push_slice(Self::encode_push_bytes(hash.as_bytes()));
    }
    basic_script
}
```

On the read side, the parser reconstructs `SystemContractUpgradeProposalInput` from the inscription instructions:

```rust
// core/lib/via_btc_client/src/indexer/parser.rs
fn parse_system_contract_upgrade_message(
    &self,
    instructions: &[Instruction],
    common_fields: &CommonFields,
) -> Option<FullInscriptionMessage> {
    if instructions.len() < MIN_SYSTEM_CONTRACT_UPGRADE_PROPOSAL {
        return None;
    }

    let version = ProtocolSemanticVersion::try_from_packed(U256::from_big_endian(
        instructions.get(2)?.push_bytes()?.as_bytes(),
    ))
    .ok()?;
    debug!("Parsed protocol version");

    let bootloader_code_hash = H256::from_slice(instructions.get(3)?.push_bytes()?.as_bytes());
    debug!("Parsed bootloader code hash");

    let default_account_code_hash =
        H256::from_slice(instructions.get(4)?.push_bytes()?.as_bytes());
    debug!("Parsed default account code hash");

    let recursion_scheduler_level_vk_hash =
        H256::from_slice(instructions.get(5)?.push_bytes()?.as_bytes());
    debug!("Parsed recursion scheduler level vk hash");

    let len = instructions.len() - 7;
    let mut system_contracts = Vec::with_capacity(len / 2);

    for i in (6..len).step_by(2) {
        let address = EVMAddress::from_slice(instructions.get(i)?.push_bytes()?.as_bytes());
        let hash = H256::from_slice(instructions.get(i + 1)?.push_bytes()?.as_bytes());
        system_contracts.push((address, hash))
    }
    debug!("Parsed system contracts");

    Some(FullInscriptionMessage::SystemContractUpgradeProposal(
        SystemContractUpgradeProposal {
            common: common_fields.clone(),
            input: SystemContractUpgradeProposalInput {
                version,
                bootloader_code_hash,
                default_account_code_hash,
                evm_emulator_code_hash: None,
                recursion_scheduler_level_vk_hash,
                system_contracts,
            },
        },
    ))
}
```

## Phase 2: Governance Execution via OP_RETURN

The execution transaction is not an inscription. It is a plain Bitcoin transaction with an `OP_RETURN` output. The parser recognizes a family of governance prefixes:

```rust
// core/lib/via_btc_client/src/indexer/parser.rs
const OP_RETURN_WITHDRAW_PREFIX: &[u8] = b"VIA_WI";
const OP_RETURN_UPGRADE_PROTOCOL_PREFIX: &[u8] = b"VIA_PROTOCOL:UPGRADE";
const OP_RETURN_UPDATE_SEQUENCER_PREFIX: &[u8] = b"VIA_PROTOCOL:SEQ";
const OP_RETURN_UPDATE_BRIDGE_PREFIX: &[u8] = b"VIA_PROTOCOL:BRI";
const OP_RETURN_UPDATE_GOVERNANCE_PREFIX: &[u8] = b"VIA_PROTOCOL:GOV";
```

`VIA_PROTOCOL:UPGRADE` executes a system contract upgrade proposal. The sibling prefixes execute wallet rotations (sequencer, bridge, governance) which follow the same proposal/execution pattern but are out of scope for this document. The upgrade execution message is parsed as follows:

```rust
// core/lib/via_btc_client/src/indexer/parser.rs
fn parse_op_return_protocol_upgrade(
    &self,
    tx: &Transaction,
    block_height: u32,
) -> Option<FullInscriptionMessage> {
    // Find OP_RETURN output
    let op_return_output = tx
        .output
        .iter()
        .find(|output| output.script_pubkey.is_op_return())?;

    // Parse OP_RETURN data
    if let Some(op_return_data) = op_return_output.script_pubkey.as_bytes().get(2..) {
        if !op_return_data.starts_with(OP_RETURN_UPGRADE_PROTOCOL_PREFIX) {
            return None;
        }

        let start = OP_RETURN_UPGRADE_PROTOCOL_PREFIX.len() + 1;
        if op_return_data.len() < start + 32 {
            return None;
        }

        // Parse proposal_tx_id from OP_RETURN data
        let proposal_tx_id = match Txid::from_slice(&op_return_data[start..start + 32]) {
            Ok(tx_id) => tx_id,
            Err(_) => return None,
        };

        let input = SystemContractUpgradeInput {
            inputs: tx.input.iter().map(|input| input.previous_output).collect(),
            proposal_tx_id,
        };

        // Create common fields with empty signature for OP_RETURN
        let common_fields = CommonFields {
            schnorr_signature: TaprootSignature::from_slice(&[0; 64]).ok()?,
            encoded_public_key: PushBytesBuf::new(),
            block_height,
            tx_id: tx.compute_ntxid().into(),
            p2wpkh_address: None,
            tx_index: None,
            output_vout: None,
        };

        return Some(FullInscriptionMessage::SystemContractUpgrade(
            SystemContractUpgrade {
                common: common_fields,
                input,
            },
        ));
    }
    None
}
```

## Indexer Detection and Governance Validation

`BitcoinInscriptionIndexer` (in `core/lib/via_btc_client/src/indexer/mod.rs`) scans each Bitcoin block, extracts governance transactions, parses them, and keeps only messages that pass governance validation:

```rust
// core/lib/via_btc_client/src/indexer/mod.rs (inside process_block)
// Parse protocol upgrade messages (Upgrade system contracts, bridge addresses, sequencer address)
if !system_txs.governance_txs.is_empty() {
    let parsed_messages: Vec<_> = system_txs
        .governance_txs
        .iter()
        .flat_map(|tx| {
            self.parser
                .parse_protocol_upgrade_transactions(tx, block_height)
        })
        .collect();

    let mut messages = vec![];
    for message in parsed_messages {
        if self.is_valid_gov_message(&message).await {
            messages.push(message);
        }
    }

    valid_messages.extend(messages);
}
```

The parser entry point tries all governance OP_RETURN message kinds:

```rust
// core/lib/via_btc_client/src/indexer/parser.rs
#[instrument(skip(self, tx), target = "bitcoin_indexer::parser")]
pub fn parse_protocol_upgrade_transactions(
    &mut self,
    tx: &TransactionWithMetadata,
    block_height: u32,
) -> Vec<FullInscriptionMessage> {
    let mut messages = Vec::new();

    if let Some(update_sequencer) = self.parse_op_return_update_governance(&tx.tx, block_height)
    {
        messages.push(update_sequencer);
    }

    if let Some(upgrade_protocol) = self.parse_op_return_protocol_upgrade(&tx.tx, block_height)
    {
        messages.push(upgrade_protocol);
    }

    if let Some(update_bridge) = self.parse_op_return_update_bridge(&tx.tx, block_height) {
        messages.push(update_bridge);
    }

    if let Some(update_sequencer) = self.parse_op_return_update_sequencer(&tx.tx, block_height)
    {
        messages.push(update_sequencer);
    }

    messages
}
```

Authorization is enforced by checking that the first spent input of the execution transaction is a UTXO owned by the governance wallet:

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

Anyone can broadcast a `VIA_PROTOCOL:UPGRADE` OP_RETURN, but only a transaction that spends a governance-wallet UTXO passes this check. The governance wallet itself is a multisig (see the operator guide below).

## Sequencer Processing

On the sequencer, `GovernanceUpgradesEventProcessor` (registered in the via_btc_watch component) consumes validated `SystemContractUpgrade` messages, fetches the referenced proposal transaction from Bitcoin, re-parses the proposal payload, builds a `ProtocolUpgrade` with an L2 upgrade transaction, and saves the new protocol version:

```rust
// core/node/via_btc_watch/src/message_processors/governance_upgrade.rs
/// Listens to operation events coming from the governance contract and saves new protocol upgrade proposals to the database.
#[derive(Debug)]
pub struct GovernanceUpgradesEventProcessor {
    /// Last protocol version seen. Used to skip events for already known upgrade proposals.
    last_seen_protocol_version: ProtocolSemanticVersion,
    /// BTC client
    btc_client: Arc<BitcoinClient>,
    /// Message parser
    message_parser: MessageParser,
    /// upgrade proposal
    upgrade: ViaProtocolUpgrade,
}
```

```rust
// core/node/via_btc_watch/src/message_processors/governance_upgrade.rs
#[async_trait::async_trait]
impl MessageProcessor for GovernanceUpgradesEventProcessor {
    async fn process_messages(
        &mut self,
        storage: &mut Connection<'_, Core>,
        msgs: Vec<FullInscriptionMessage>,
        _: &mut BitcoinInscriptionIndexer,
    ) -> Result<Option<u32>, MessageProcessorError> {
        let mut upgrades = Vec::new();
        for msg in msgs {
            if let FullInscriptionMessage::SystemContractUpgrade(system_contract_upgrade_msg) = &msg
            {
                let proposal_tx = self
                    .btc_client
                    .get_transaction(&system_contract_upgrade_msg.input.proposal_tx_id)
                    .await
                    .map_err(|err| {
                        MessageProcessorError::Internal(anyhow::anyhow!(
                            "Failed to fetch protocol upgrade transaction: {}, error {}",
                            system_contract_upgrade_msg.input.proposal_tx_id,
                            err
                        ))
                    })?;

                let messages = self.message_parser.parse_system_transaction(
                    &proposal_tx,
                    system_contract_upgrade_msg.common.block_height,
                    None,
                );

                for message in messages {
                    match message {
                        FullInscriptionMessage::SystemContractUpgradeProposal(
                            system_contract_upgrade_proposal_msg,
                        ) => {
                            // Ignore if old version
                            if system_contract_upgrade_proposal_msg.input.version
                                <= self.last_seen_protocol_version
                            {
                                tracing::info!(
                                    "Upgrade transaction with version {} already processed, skipping",
                                    system_contract_upgrade_proposal_msg.input.version
                                );
                                continue;
                            }

                            tracing::info!(
                                "Received upgrades with versions: {:?}",
                                system_contract_upgrade_proposal_msg.input.version
                            );
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
                            upgrades.push((
                                upgrade,
                                system_contract_upgrade_proposal_msg
                                    .input
                                    .recursion_scheduler_level_vk_hash,
                            ));
                        }
                        _ => (),
                    }
                }
            }
        }

        let Some(last_upgrade) = upgrades.last() else {
            return Ok(None);
        };

        let last_version = last_upgrade.0.version;
        for (upgrade, recursion_scheduler_level_vk_hash) in upgrades {
            let latest_semantic_version = storage
                .protocol_versions_dal()
                .latest_semantic_version()
                .await
                .map_err(DalError::generalize)?
                .context("expected some version to be present in DB")?;

            if upgrade.version > latest_semantic_version {
                let latest_version = storage
                    .protocol_versions_dal()
                    .get_protocol_version_with_latest_patch(latest_semantic_version.minor)
                    .await
                    .map_err(DalError::generalize)?
                    .with_context(|| {
                        format!(
                            "expected minor version {} to be present in DB",
                            latest_semantic_version.minor as u16
                        )
                    })?;

                let new_version =
                    latest_version.apply_upgrade(upgrade, Some(recursion_scheduler_level_vk_hash));

                storage
                    .protocol_versions_dal()
                    .save_protocol_version_with_tx(&new_version)
                    .await
                    .map_err(DalError::generalize)?;

                METRICS.inscriptions_processed[&InscriptionStage::Upgrade]
                    .set(new_version.version.minor as usize);
            }
        }
        self.last_seen_protocol_version = last_version;

        Ok(None)
    }
}
```

### Building the L2 Upgrade Transaction

The system contract redeployment is executed on L2 as a protocol upgrade transaction that calls `forceDeployOnAddresses` on the contract deployer, sent from the force deployer address:

```rust
// core/lib/types/src/via_protocol_upgrade.rs
const GAS_LIMIT: u64 = 72_000_000;
const GAS_PER_PUB_DATA_BYTE_LIMIT: u64 = 800;

#[derive(Debug, Clone, Default)]
pub struct ViaProtocolUpgrade {}

impl ViaProtocolUpgrade {
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

    pub fn get_canonical_tx_hash(
        &self,
        version: ProtocolSemanticVersion,
        system_contracts: Vec<(Address, H256)>,
    ) -> anyhow::Result<H256> {
        let l2_transaction = L2CanonicalTransaction {
            tx_type: PROTOCOL_UPGRADE_TX_TYPE.into(),
            from: U256::from_big_endian(&CONTRACT_FORCE_DEPLOYER_ADDRESS.0),
            to: U256::from_big_endian(&CONTRACT_DEPLOYER_ADDRESS.0),
            gas_limit: U256::from(GAS_LIMIT),
            gas_per_pubdata_byte_limit: U256::from(GAS_PER_PUB_DATA_BYTE_LIMIT),
            max_fee_per_gas: U256::zero(),
            max_priority_fee_per_gas: U256::zero(),
            paymaster: U256::zero(),
            nonce: U256::from(version.minor as u64),
            value: U256::zero(),
            reserved: [U256::zero(), U256::zero(), U256::zero(), U256::zero()],
            data: self.get_calldata(system_contracts.clone())?,
            signature: vec![],
            factory_deps: vec![],
            paymaster_input: vec![],
            reserved_dynamic: vec![],
        };

        Ok(l2_transaction.hash())
    }

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
}
```

Because both the sequencer and the verifier derive `canonical_tx_hash` deterministically from the same proposal data, the verifier can later recognize the executed upgrade transaction in the batch pubdata without trusting the sequencer.

## Activation at the Batch Boundary (State Keeper)

There is no separately scheduled "activation block". The upgrade activates on the first L1 batch whose protocol version differs from the previous batch's version. At that boundary the state keeper loads the pending upgrade transaction and executes it as the first transaction of the batch:

```rust
// core/node/state_keeper/src/keeper.rs
pub(super) async fn load_protocol_upgrade_tx(
    &mut self,
    pending_l2_blocks: &[L2BlockExecutionData],
    protocol_version: ProtocolVersionId,
    l1_batch_number: L1BatchNumber,
) -> Result<Option<ProtocolUpgradeTx>, Error> {
    // After the Shared Bridge is integrated,
    // there has to be a setChainId upgrade transaction after the chain genesis.
    // It has to be the first transaction of the first batch.
    // The setChainId upgrade does not bump the protocol version, but attaches an upgrade
    // transaction to the genesis protocol version.
    let first_batch_in_shared_bridge =
        l1_batch_number == L1BatchNumber(1) && !protocol_version.is_pre_shared_bridge();
    let previous_batch_protocol_version =
        self.io.load_batch_version_id(l1_batch_number - 1).await?;

    let version_changed = protocol_version != previous_batch_protocol_version;
    let mut protocol_upgrade_tx = if version_changed || first_batch_in_shared_bridge {
        self.io.load_upgrade_tx(protocol_version).await?
    } else {
        None
    };

    // Sanity check: if `txs_to_reexecute` is not empty and upgrade tx is present for this block
    // then it must be the first one in `txs_to_reexecute`.
    if !pending_l2_blocks.is_empty() && protocol_upgrade_tx.is_some() {
        // We already processed the upgrade tx but did not seal the batch it was in.
        let first_tx_to_reexecute = &pending_l2_blocks[0].txs[0];
        assert_eq!(
            first_tx_to_reexecute.tx_format(),
            TransactionType::ProtocolUpgradeTransaction,
            "Expected an upgrade transaction to be the first one in pending L2 blocks, but found {:?}",
            first_tx_to_reexecute.hash()
        );
        tracing::info!(
            "There is a protocol upgrade in batch #{l1_batch_number}, upgrade tx already processed"
        );
        protocol_upgrade_tx = None; // The protocol upgrade was already executed
    }

    if protocol_upgrade_tx.is_some() {
        tracing::info!("There is a new upgrade tx to be executed in batch #{l1_batch_number}");
    }
    Ok(protocol_upgrade_tx)
}
```

## Sequential Batch Commitment During Upgrades

The BTC sender aggregator guarantees that batch commit inscriptions to Bitcoin remain strictly sequential across an upgrade. When a protocol upgrade occurs, all batches created with the previous protocol version are committed first, then the aggregator switches to the new version:

```rust
// core/node/via_btc_sender/src/aggregator.rs
async fn get_ready_for_commit_l1_batches(
    &self,
    storage: &mut Connection<'_, Core>,
) -> anyhow::Result<Vec<ViaBtcL1BlockDetails>> {
    let protocol_version_id = self.get_last_protocol_version_id(storage).await?;
    let prev_protocol_version_id = self.get_prev_used_protocol_version(storage).await?;

    let base_system_contracts_hashes = self
        .load_base_system_contracts(storage, protocol_version_id)
        .await?;

    // In case of a protocol upgrade, we first process the l1 batches created with the previous protocol version
    // then switch to the new one.
    if prev_protocol_version_id != protocol_version_id {
        let prev_base_system_contracts_hashes = self
            .load_base_system_contracts(storage, prev_protocol_version_id)
            .await?;
        let ready_for_commit_l1_batches = storage
            .via_blocks_dal()
            .get_ready_for_commit_l1_batches(
                self.config.max_aggregated_blocks_to_commit as usize,
                &prev_base_system_contracts_hashes.bootloader,
                &prev_base_system_contracts_hashes.default_aa,
                prev_protocol_version_id,
            )
            .await?;

        if !ready_for_commit_l1_batches.is_empty() {
            return Ok(ready_for_commit_l1_batches);
        }
    }

    let ready_for_commit_l1_batches = storage
        .via_blocks_dal()
        .get_ready_for_commit_l1_batches(
            self.config.max_aggregated_blocks_to_commit as usize,
            &base_system_contracts_hashes.bootloader,
            &base_system_contracts_hashes.default_aa,
            protocol_version_id,
        )
        .await?;
    Ok(ready_for_commit_l1_batches)
}
```

Batch sequence integrity is enforced by number and hash chaining:

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
    } else if let Some(batch) = ready_for_commit_l1_batches.first() {
        if batch.number.0 != 1 {
            anyhow::bail!("Invalid batch after genesis, not sequential")
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

## Verifier Processing

The verifier network runs its own `GovernanceUpgradesEventProcessor` in `via_verifier/node/via_btc_watch`. It follows the same fetch-and-reparse pattern as the sequencer, but instead of building an executable L2 transaction it computes the canonical upgrade transaction hash and stores the pending version:

```rust
// via_verifier/node/via_btc_watch/src/message_processors/governance_upgrade.rs
#[async_trait::async_trait]
impl MessageProcessor for GovernanceUpgradesEventProcessor {
    async fn process_messages(
        &mut self,
        storage: &mut Connection<'_, Verifier>,
        msgs: Vec<FullInscriptionMessage>,
        _: &mut BitcoinInscriptionIndexer,
    ) -> Result<Option<u32>, MessageProcessorError> {
        let mut upgrades = Vec::new();
        for msg in msgs {
            if let FullInscriptionMessage::SystemContractUpgrade(system_contract_upgrade_msg) = &msg
            {
                let proposal_tx = self
                    .btc_client
                    .get_transaction(&system_contract_upgrade_msg.input.proposal_tx_id)
                    .await
                    .map_err(|err| {
                        MessageProcessorError::Internal(anyhow::anyhow!(
                            "Failed to fetch protocol upgrade transaction: {}, error {}",
                            system_contract_upgrade_msg.input.proposal_tx_id,
                            err
                        ))
                    })?;

                let messages = self.message_parser.parse_system_transaction(
                    &proposal_tx,
                    system_contract_upgrade_msg.common.block_height,
                    None,
                );

                for message in messages {
                    match message {
                        FullInscriptionMessage::SystemContractUpgradeProposal(
                            system_contract_upgrade_proposal_msg,
                        ) => {
                            if system_contract_upgrade_proposal_msg.input.version
                                < get_sequencer_version()
                            {
                                tracing::info!(
                                    "Upgrade transaction with version {} already processed, skipping",
                                    system_contract_upgrade_proposal_msg.input.version
                                );
                                continue;
                            }

                            tracing::info!(
                                "Received upgrades with versions: {:?}",
                                system_contract_upgrade_proposal_msg.input.version
                            );

                            let hash = self.upgrade.get_canonical_tx_hash(
                                system_contract_upgrade_proposal_msg.input.version,
                                system_contract_upgrade_proposal_msg.input.system_contracts,
                            )?;

                            let upgrade = (
                                system_contract_upgrade_proposal_msg.input.version,
                                system_contract_upgrade_proposal_msg
                                    .input
                                    .bootloader_code_hash,
                                system_contract_upgrade_proposal_msg
                                    .input
                                    .default_account_code_hash,
                                hash,
                                system_contract_upgrade_proposal_msg
                                    .input
                                    .recursion_scheduler_level_vk_hash,
                            );

                            upgrades.push(upgrade);
                        }
                        _ => (),
                    }
                }
            }
        }

        for (
            version,
            bootloader_code_hash,
            default_account_code_hash,
            canonical_tx_hash,
            recursion_scheduler_level_vk_hash,
        ) in upgrades
        {
            METRICS.inscriptions_processed[&InscriptionStage::Upgrade].set(version.minor as usize);

            storage
                .via_protocol_versions_dal()
                .save_protocol_version(
                    version,
                    bootloader_code_hash.as_bytes(),
                    default_account_code_hash.as_bytes(),
                    canonical_tx_hash.as_bytes(),
                    recursion_scheduler_level_vk_hash.as_bytes(),
                )
                .await
                .map_err(DalError::generalize)?;
        }
        Ok(None)
    }
}
```

### The "executed" Flag

`save_protocol_version` inserts the new version into the verifier's `protocol_versions` table with `executed = FALSE`:

```sql
-- via_verifier/lib/verifier_dal/src/via_protocol_versions_dal.rs (save_protocol_version)
INSERT INTO
protocol_versions (
    id,
    bootloader_code_hash,
    default_account_code_hash,
    upgrade_tx_hash,
    recursion_scheduler_level_vk_hash,
    executed,
    created_at
)
VALUES
($1, $2, $3, $4, $5, FALSE, NOW())
ON CONFLICT (id) DO
UPDATE
SET
upgrade_tx_hash = excluded.upgrade_tx_hash,
executed = FALSE;
```

During ZK verification, the verifier checks whether the first user log of the batch pubdata is the pending upgrade transaction (sent by the bootloader with the pre-computed canonical hash as its key):

```rust
// via_verifier/node/via_zk_verifier/src/lib.rs
/// Check whether the first user_log corresponds to an upgrade transaction.
pub async fn verify_upgrade_tx_hash(
    &mut self,
    storage: &mut Connection<'_, Verifier>,
    pubdata: &Pubdata,
) -> anyhow::Result<Option<H256>> {
    if let Some(upgrade_tx_hash) = storage
        .via_protocol_versions_dal()
        .get_in_progress_upgrade_tx_hash()
        .await?
    {
        if let Some(log) = pubdata.user_logs.first() {
            if log.sender == H160::from_str(L2_BOOTLOADER_CONTRACT_ADDR)?
                && log.key == upgrade_tx_hash
            {
                tracing::info!("Found upgrade transaction in pubdata: {}", upgrade_tx_hash);
                return Ok(Some(upgrade_tx_hash));
            }
        }
        return Ok(None);
    }
    Ok(None)
}
```

When the batch proof verifies successfully, the upgrade is marked as executed:

```rust
// via_verifier/node/via_zk_verifier/src/lib.rs (inside the batch verification loop)
if let Some(upgrade_tx_hash) = upgrade_tx_hash_opt {
    transaction
        .via_protocol_versions_dal()
        .mark_upgrade_as_executed(upgrade_tx_hash.as_bytes())
        .await?;
}
```

```rust
// via_verifier/lib/verifier_dal/src/via_protocol_versions_dal.rs
pub async fn mark_upgrade_as_executed(&mut self, upgrade_tx_hash: &[u8]) -> DalResult<()> {
    sqlx::query!(
        r#"
        UPDATE protocol_versions
        SET
            executed = TRUE
        WHERE
            upgrade_tx_hash = $1
        "#,
        upgrade_tx_hash
    )
    .instrument("mark_upgrade_as_executed")
    .fetch_optional(self.storage)
    .await?;
    Ok(())
}
```

### Verifier Node Version Gating

The verifier binary hardcodes the highest sequencer protocol version it supports. If the network upgrades beyond it, the node refuses to run until the operator upgrades the software:

```rust
// via_verifier/lib/via_verifier_types/src/protocol_version.rs
const SEQUENCER_MINOR: ProtocolVersionId = ProtocolVersionId::Version28;
const SEQUENCER_PATCH: u32 = 0;

/// Get the supported sequencer version by the verifier.
pub fn get_sequencer_version() -> ProtocolSemanticVersion {
    ProtocolSemanticVersion {
        minor: SEQUENCER_MINOR,
        patch: VersionPatch(SEQUENCER_PATCH),
    }
}

/// TODO: Once the protocol stabilizes, update the logic to check only for changes in the minor version,
/// preventing node upgrades for patch-level changes.
pub fn check_if_supported_sequencer_version(
    last_protocol_version: ProtocolSemanticVersion,
) -> anyhow::Result<()> {
    let supported_sequencer_version = get_sequencer_version();

    if supported_sequencer_version < last_protocol_version {
        anyhow::bail!(
            "Verifier node version must be upgraded from {} to {}",
            supported_sequencer_version.to_string(),
            last_protocol_version.to_string()
        );
    }
    Ok(())
}
```

Proof verification itself is versioned. The `via_verification` library ships per-protocol-version modules (`version_27`, `version_28`) and dispatches on the batch's protocol version:

```rust
// via_verifier/lib/via_verification/src/lib.rs
/// Decodes the proof data into the appropriate version. It's possible that the data serialization format changes even within the same version.
pub fn decode_prove_batch_data(
    protocol_version_id: ProtocolVersionId,
    proof_data: &[u8],
) -> anyhow::Result<ProveBatchData> {
    if protocol_version_id <= ProtocolVersionId::Version27 {
        if let Ok(prove_batch) = bincode::deserialize::<ProveBatchesV27>(proof_data) {
            tracing::info!("Decode proof data with V27");
            return Ok(ProveBatchData::V27(prove_batch));
        }
        tracing::warn!("Failed to decode proof data as V27");
    }

    if let Ok(prove_batch) = bincode::deserialize::<ProveBatchesV28>(proof_data) {
        tracing::info!("Decode proof data with V28");
        return Ok(ProveBatchData::V28(prove_batch));
    }

    // If both fail, return an error
    anyhow::bail!("Failed to decode proof data as either V27 or V28");
}

pub async fn verify_proof(prover_batch_data: ProveBatchData) -> anyhow::Result<bool> {
    match prover_batch_data {
        ProveBatchData::V27(data) => verify_proof_v27(data).await,
        ProveBatchData::V28(data) => verify_proof_v28(data).await,
    }
}
```

## Protocol Version Management

- **Semantic versions**: versions are `ProtocolSemanticVersion` values (a `minor` `ProtocolVersionId` plus a `patch`). On the wire, the version is packed into a `U256` (`input.version.pack()` in the script builder, `ProtocolSemanticVersion::try_from_packed` in the parser).
- **Sequencer storage**: new versions are stored via `protocol_versions_dal().save_protocol_version_with_tx(...)` after `apply_upgrade(upgrade, Some(recursion_scheduler_level_vk_hash))`.
- **Verifier storage**: new versions are stored in the verifier's own `protocol_versions` and `protocol_patches` tables via `via_protocol_versions_dal().save_protocol_version(...)`, with the `executed` flag tracking whether the upgrade transaction has been observed in a verified batch.
- **Per-batch versioning**: each L1 batch records its protocol version; the state keeper switches versions at a batch boundary, and the BTC sender commits old-version batches before new-version ones.
- **Verifier key rotation**: the `recursion_scheduler_level_vk_hash` carried in the proposal updates the verification key hash used to check SNARK proofs for batches produced under the new version.

## Security Considerations

The upgrade process includes several security measures:

1. **Governance authorization**: an upgrade only takes effect through a `VIA_PROTOCOL:UPGRADE` transaction that spends a governance-wallet UTXO (`is_valid_gov_message` / `is_valid_gov_upgrade`). The governance wallet is a multisig; proposals themselves are permissionless and inert.
2. **Transparency**: both the proposal payload and the execution decision are recorded on the Bitcoin blockchain, providing a public audit trail.
3. **Independent re-derivation**: sequencer and verifier each fetch the proposal transaction from Bitcoin and re-parse it; the verifier recomputes the canonical L2 upgrade transaction hash and later matches it against the bootloader's user log in the proven batch pubdata (`verify_upgrade_tx_hash`).
4. **Monotonic versions**: the sequencer skips proposals whose version is not strictly greater than the latest known semantic version; the verifier skips versions below its supported sequencer version.
5. **Sequential commitment**: `validate_l1_batch_sequence` and `get_ready_for_commit_l1_batches` guarantee that no batch is skipped or reordered across the version transition.
6. **Node gating**: `check_if_supported_sequencer_version` halts verifier nodes that have not been upgraded to software supporting the new protocol version.

## Upgrade Tooling and Example Workflow

### Creating a proposal inscription (example binary)

`core/lib/via_btc_client/examples/upgrade_system_contracts.rs` inscribes a `SystemContractUpgradeProposalInput`. Its usage, from the source:

```rust
// core/lib/via_btc_client/examples/upgrade_system_contracts.rs
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
//     0x010000758176955a4cb7af4c768cdad127fa6b1d0329ce3e8b37d1382e9c21e6,... \
//     0x14f97b81e54b35fe673d8708cc1a19e1ea5b5e348e12d31e39824ed4f42bbca2
```

The positional arguments are:
1. Bitcoin network (mainnet, testnet, regtest)
2. Bitcoin RPC URL
3. Bitcoin RPC username
4. Bitcoin RPC password
5. Private key for signing the inscription
6. New protocol version (e.g. `0.26.0`)
7. New bootloader code hash
8. New default account code hash
9. System contract addresses (comma-separated)
10. System contract hashes (comma-separated)
11. Recursion scheduler level (SNARK) verification key hash

The core of the example:

```rust
// core/lib/via_btc_client/examples/upgrade_system_contracts.rs
// System contracts Upgrade message
let input = SystemContractUpgradeProposalInput {
    version,
    bootloader_code_hash,
    default_account_code_hash,
    recursion_scheduler_level_vk_hash,
    system_contracts,
    evm_emulator_code_hash: None,
};
let system_contract_upgrade_info = inscriber
    .inscribe(InscriptionMessage::SystemContractUpgradeProposal(input))
    .await?;
```

### End-to-end operator flow

The operator guide `docs/via_guides/upgrade.md` in via-core describes the full production workflow:

1. **Build the new system contracts**: `cd contracts && yarn sc build`.
2. **Create an upgrade config** with the `infrastructure/via-protocol-upgrade` tool: `yarn start upgrades create via-network --protocol-version <new-version>`.
3. **Publish the new contracts** and generate the upgrade file (`yarn start system-contracts publish ... --new-protocol-version <version> --recursion-scheduler-level-vk-hash <hash> --bootloader --default-aa --system-contracts`).
4. **Create the upgrade proposal inscription**: `yarn start l2-transaction upgrade-system-contracts --environment <env> --private-key <l1-private-key>`. Any P2PKH wallet can do this.
5. **Execute the proposal via the governance multisig** using the `via` CLI:
   - `via multisig compute-multisig --pubkeys <pk1,pk2,pk3> --minimumSigners 2`
   - `via multisig create-upgrade-tx --inputTxId <tx_id> --inputVout <vout> --inputAmount <amount> --upgradeProposalTxId <upgradeProposalTxId> --fee 500` (the selected input must be a governance-wallet UTXO)
   - Each signer runs `via multisig sign-tx --privateKey <key>` and passes the file along until the threshold is met, then the final transaction (with its `VIA_PROTOCOL:UPGRADE` OP_RETURN) is broadcast.
6. **Monitor processing**: the sequencer and verifier btc_watch components detect the execution transaction, validate the governance UTXO, and store the new protocol version.
7. **Activation**: the state keeper executes the `forceDeployOnAddresses` upgrade transaction as the first transaction of the first batch with the new version; the verifier matches its hash in the proven pubdata and marks the upgrade as executed.

---

In summary, a Via upgrade is a two-phase Bitcoin-anchored process: a permissionless proposal inscription carrying the new version, code hashes, and verifier key hash, followed by a governance-multisig execution transaction that references the proposal by txid in a `VIA_PROTOCOL:UPGRADE` OP_RETURN. Sequencer and verifier independently re-derive the upgrade from Bitcoin data, the state keeper applies it deterministically at a batch boundary, the BTC sender preserves strict batch ordering across the transition, and the verifier network confirms execution inside a validity-proven batch before marking the version as live.
