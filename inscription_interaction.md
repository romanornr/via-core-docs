# Inscription Standard Interaction in Via L2 Bitcoin ZK-Rollup

This document details how the Via L2 Bitcoin ZK-Rollup system interacts with Bitcoin inscriptions, focusing on both Via's custom inscription protocol and its handling of standard Bitcoin inscriptions.

## Table of Contents

1. [Overview](#overview)
2. [Primary Code Modules](#primary-code-modules)
3. [Via's Custom Inscription Protocol](#vias-custom-inscription-protocol)
4. [Inscription Types](#inscription-types)
5. [Inscription Creation Process](#inscription-creation-process)
6. [Inscription Indexing and Parsing](#inscription-indexing-and-parsing)
7. [Interaction with Standard Bitcoin Inscriptions](#interaction-with-standard-bitcoin-inscriptions)
8. [Integration with Other Components](#integration-with-other-components)

## Overview

Via L2 uses Bitcoin inscriptions as a fundamental mechanism for committing state and operational data to Bitcoin's L1 blockchain. This provides security, transparency, and verifiability for the L2 system by anchoring critical information in Bitcoin's immutable ledger.

The system defines a custom inscription protocol that follows a specific format and uses a commit-reveal pattern for creating inscriptions. These inscriptions serve various purposes in the system, from committing batch data to handling L1-to-L2 messages.

## Primary Code Modules

The inscription-related functionality is primarily implemented in the following modules:

1. **`core/lib/via_btc_client/`**: The main module for Bitcoin client functionality, including:
   - `inscriber/`: Creates and broadcasts inscriptions
   - `indexer/`: Indexes and parses inscriptions
   - `client/`: Interacts with the Bitcoin network
   - `signer/`: Signs transactions
   - `types.rs`: Defines inscription data structures

2. **`core/bin/via_server/`**: The server implementation that uses the Bitcoin client:
   - `node_builder.rs`: Configures and builds the node with Bitcoin-related components
   - `config.rs`: Handles Bitcoin-related configuration

3. **`core/lib/via_da_clients/`**: Data availability clients, including Celestia integration

## Via's Custom Inscription Protocol

Via L2 defines a custom inscription protocol for embedding data in Tapscript outputs. All inscriptions follow a standard format:

```
<pubkey> OP_CHECKSIG OP_FALSE OP_IF "via_inscription_protocol" <message_type> <data...> OP_ENDIF
```

This structure allows for:
- Verification of the inscription creator via the pubkey and signature
- Clear identification of Via L2 inscriptions via the protocol marker
- Type-specific data encoding

The protocol marker is defined in `types.rs`:

```rust
pub(crate) const VIA_INSCRIPTION_PROTOCOL: &str = "via_inscription_protocol";
```

### Commit-Reveal Pattern

Via L2 uses a two-transaction pattern for inscriptions:

1. **Commit Transaction**: Creates a Taproot output that commits to the inscription script
2. **Reveal Transaction**: Spends the Taproot output, revealing the inscription data

This pattern is implemented in the `inscriber/mod.rs` file:

```rust
// Commit transaction creates a Taproot output
let final_commit_tx = self.sign_commit_tx(&commit_tx_input_info, &commit_tx_output_info)?;

// Reveal transaction spends the Taproot output
let final_reveal_tx = self.sign_reveal_tx(
    &reveal_tx_input_info,
    &reveal_tx_output_info,
    &inscription_data,
)?;
```

The commit transaction creates a P2TR (Pay-to-Taproot) output that commits to the inscription script, while the reveal transaction spends this output using the Tapscript path, revealing the inscription data.

## Inscription Types

Via L2 supports six types of inscriptions, each serving a specific purpose in the system:

1. **L1BatchDAReference**: Commits L1 batch data availability information
   ```rust
   pub struct L1BatchDAReferenceInput {
       pub l1_batch_hash: H256,
       pub l1_batch_index: L1BatchNumber,
       pub da_identifier: String,
       pub blob_id: String,
       pub prev_l1_batch_hash: H256,
   }
   ```

2. **ProofDAReference**: Commits proof data availability information
   ```rust
   pub struct ProofDAReferenceInput {
       pub l1_batch_reveal_txid: Txid,
       pub da_identifier: String,
       pub blob_id: String,
   }
   ```

3. **ValidatorAttestation**: Contains validator attestations for batch validation
   ```rust
   pub struct ValidatorAttestationInput {
       pub reference_txid: Txid,
       pub attestation: Vote,
   }
   ```

4. **SystemBootstrapping**: Contains system bootstrapping data
   ```rust
   pub struct SystemBootstrappingInput {
       pub start_block_height: u32,
       pub verifier_p2wpkh_addresses: Vec<BitcoinAddress<NetworkUnchecked>>,
       pub bridge_musig2_address: BitcoinAddress<NetworkUnchecked>,
       pub bootloader_hash: H256,
       pub abstract_account_hash: H256,
   }
   ```

5. **ProposeSequencer**: Contains sequencer proposal data
   ```rust
   pub struct ProposeSequencerInput {
       pub sequencer_new_p2wpkh_address: BitcoinAddress<NetworkUnchecked>,
   }
   ```

6. **L1ToL2Message**: Contains L1 to L2 message data
   ```rust
   pub struct L1ToL2MessageInput {
       pub receiver_l2_address: EVMAddress,
       pub l2_contract_address: EVMAddress,
       pub call_data: Vec<u8>,
   }
   ```

7. **SystemContractUpgrade**: Contains system contract upgrade information
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

Each inscription type has a corresponding message identifier defined as a static `PushBytesBuf` value:

```rust
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
    PushBytesBuf::from(b"SystemContractUpgrade");
```

## Inscription Creation Process

The inscription creation process is handled by the `Inscriber` class in `inscriber/mod.rs`. The main steps are:

1. **Prepare Inscription Data**: Create the inscription script and Taproot commitment
   ```rust
   let inscription_data = InscriptionData::new(input, secp_ref, internal_key, network)?;
   ```

2. **Prepare Commit Transaction**: Create a transaction that commits to the inscription script
   ```rust
   let commit_tx_input_info = self.prepare_commit_tx_input().await?;
   let commit_tx_output_info = self.prepare_commit_tx_output(&commit_tx_input_info, inscription_data.script_pubkey.clone()).await?;
   let final_commit_tx = self.sign_commit_tx(&commit_tx_input_info, &commit_tx_output_info)?;
   ```

3. **Prepare Reveal Transaction**: Create a transaction that reveals the inscription data
   ```rust
   let reveal_tx_input_info = self.prepare_reveal_tx_input(&commit_tx_output_info, &final_commit_tx, &inscription_data)?;
   let reveal_tx_output_info = self.prepare_reveal_tx_output(&reveal_tx_input_info, &inscription_data, recipient, commit_tx_output_info.commit_tx_fee).await?;
   let final_reveal_tx = self.sign_reveal_tx(&reveal_tx_input_info, &reveal_tx_output_info, &inscription_data)?;
   ```

4. **Broadcast Transactions**: Send both transactions to the Bitcoin network
   ```rust
   self.broadcast_inscription(&final_commit_tx, &final_reveal_tx).await?;
   ```

The `InscriptionData` struct in `script_builder.rs` encapsulates the Tapscript construction:

```rust
pub struct InscriptionData {
    pub inscription_script: ScriptBuf,
    pub script_size: usize,
    pub script_pubkey: ScriptBuf,
    pub taproot_spend_info: TaprootSpendInfo,
}
```

The `script_builder.rs` file contains methods for building different types of inscription scripts, such as:

```rust
fn build_l1_batch_da_reference_script(
    basic_script: ScriptBuilder,
    input: &types::L1BatchDAReferenceInput,
) -> ScriptBuilder {
    // ...
}
```

## Inscription Indexing and Parsing

The inscription indexing and parsing process is handled by the `BitcoinInscriptionIndexer` class in `indexer/mod.rs` and the `MessageParser` class in `indexer/parser.rs`.

The `BitcoinInscriptionIndexer` is responsible for:
1. Monitoring the Bitcoin blockchain for new blocks
2. Extracting relevant transactions
3. Parsing inscriptions from these transactions
4. Validating the inscriptions
5. Processing the inscriptions based on their type

```rust
pub async fn process_block(
    &mut self,
    block_height: u32,
) -> BitcoinIndexerResult<Vec<FullInscriptionMessage>> {
    // ...
    let (system_tx, bridge_tx) = self.extract_important_transactions(&block.txdata);
    
    if let Some(system_tx) = system_tx {
        let parsed_messages: Vec<_> = system_tx
            .iter()
            .flat_map(|tx| self.parser.parse_system_transaction(tx, block_height))
            .collect();

        let messages: Vec<_> = parsed_messages
            .into_iter()
            .filter(|message| self.is_valid_system_message(message))
            .collect();

        valid_messages.extend(messages);
    }
    // ...
}
```

The `MessageParser` is responsible for parsing the inscriptions from the transaction witnesses:

```rust
fn parse_system_input(
    &mut self,
    input: &bitcoin::TxIn,
    tx: &Transaction,
    block_height: u32,
    address: Address,
) -> Option<FullInscriptionMessage> {
    // Parse signature, script, and control block from witness
    let signature = TaprootSignature::from_slice(&witness[0])?;
    let script = ScriptBuf::from_bytes(witness[1].to_vec());
    let control_block = ControlBlock::decode(&witness[2])?;

    // Extract instructions and find Via inscription protocol
    let instructions: Vec<_> = script.instructions().filter_map(Result::ok).collect();
    let via_index = find_via_inscription_protocol(&instructions)?;

    // Create common fields
    let common_fields = CommonFields {
        schnorr_signature: signature,
        encoded_public_key: PushBytesBuf::from(public_key.serialize()),
        block_height,
        tx_id: tx.compute_ntxid().into(),
        p2wpkh_address: Some(address),
    };

    // Parse the message based on its type
    self.parse_system_message(tx, &instructions[via_index..], &common_fields)
}
```

The parser includes methods for parsing each type of inscription, such as:

```rust
fn parse_l1_batch_da_reference(
    &self,
    instructions: &[Instruction],
    common_fields: &CommonFields,
) -> Option<FullInscriptionMessage> {
    // ...
}
```

## Interaction with Standard Bitcoin Inscriptions

Via L2 also supports interaction with standard Bitcoin inscriptions, particularly for L1-to-L2 messaging. This is handled by the `parse_bridge_transaction` method in the `MessageParser` class:

```rust
pub fn parse_bridge_transaction(
    &mut self,
    tx: &Transaction,
    block_height: u32,
) -> Vec<FullInscriptionMessage> {
    // Find bridge output
    let bridge_output = match tx.output.iter().find(|output| {
        if let Some(bridge_addr) = &self.bridge_address {
            output.script_pubkey == bridge_addr.script_pubkey()
        } else {
            false
        }
    }) {
        Some(output) => output,
        None => return vec![], // Not a bridge transaction
    };

    // Try to parse as inscription-based deposit first
    if let Some(inscription_message) = self.parse_inscription_deposit(tx, block_height) {
        return vec![inscription_message];
    }

    // If not an inscription, try to parse as OP_RETURN based deposit
    if let Some(op_return_message) = self.parse_op_return_deposit(tx, block_height, bridge_output) {
        return vec![op_return_message];
    }

    vec![]
}
```

The system supports two types of deposits:

1. **Inscription-based deposits**: Using the Via inscription protocol
   ```rust
   fn parse_inscription_deposit(
       &self,
       tx: &Transaction,
       block_height: u32,
   ) -> Option<FullInscriptionMessage> {
       // ...
   }
   ```

2. **OP_RETURN-based deposits**: Using standard Bitcoin OP_RETURN outputs
   ```rust
   fn parse_op_return_deposit(
       &self,
       tx: &Transaction,
       block_height: u32,
       bridge_output: &TxOut,
   ) -> Option<FullInscriptionMessage> {
       // ...
   }
   ```

This dual approach allows Via L2 to support both its custom inscription protocol and standard Bitcoin transactions with OP_RETURN outputs.

## Integration with Other Components

The inscription handling is integrated with other components of the Via L2 system through several layers in the node architecture:

1. **BtcWatchLayer**: Monitors the Bitcoin blockchain for inscriptions
   ```rust
   fn add_btc_watcher_layer(mut self) -> anyhow::Result<Self> {
       let btc_watch_config = try_load_config!(self.configs.via_btc_watch_config);
       self.node.add_layer(BtcWatchLayer::new(btc_watch_config));
       Ok(self)
   }
   ```

2. **ViaBtcInscriptionAggregatorLayer** and **ViaInscriptionManagerLayer**: Manage the creation and broadcasting of inscriptions
   ```rust
   fn add_btc_sender_layer(mut self) -> anyhow::Result<Self> {
       let btc_sender_config = try_load_config!(self.configs.via_btc_sender_config);
       self.node.add_layer(ViaBtcInscriptionAggregatorLayer::new(btc_sender_config.clone()));
       self.node.add_layer(ViaInscriptionManagerLayer::new(btc_sender_config));
       Ok(self)
   }
   ```

3. **DataAvailabilityDispatcherLayer**: Handles data availability for batch data and proofs
   ```rust
   fn add_da_dispatcher_layer(mut self) -> anyhow::Result<Self> {
       let state_keeper_config = try_load_config!(self.configs.state_keeper_config);
       let da_config = try_load_config!(self.configs.da_dispatcher_config);
       let btc_sender_config = try_load_config!(self.configs.via_btc_sender_config);

       let dispatch_real_proof = btc_sender_config.proof_sending_mode != ProofSendingMode::SkipEveryProof;
       self.node.add_layer(DataAvailabilityDispatcherLayer::new(
           state_keeper_config,
           da_config,
           dispatch_real_proof,
       ));

       Ok(self)
   }
   ```

These components work together to provide a complete system for creating, broadcasting, indexing, and processing Bitcoin inscriptions in the Via L2 system.

---

In summary, Via L2 uses a custom inscription protocol on Bitcoin L1 to commit state and operational data, supporting six types of inscriptions for different purposes. The system also supports standard Bitcoin inscriptions for L1-to-L2 messaging, providing a comprehensive approach to leveraging Bitcoin's security and immutability for the L2 rollup.