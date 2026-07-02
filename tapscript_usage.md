# Tapscript Usage in Via L2 Bitcoin ZK-Rollup

This document details how Tapscript is utilized within the Via L2 Bitcoin ZK-Rollup system. Tapscript, a scripting language introduced with Bitcoin's Taproot upgrade (BIP 341), is leveraged by Via L2 for various critical functions.

## Table of Contents

1. [Overview](#overview)
2. [Inscription Protocol](#inscription-protocol)
3. [Commit-Reveal Pattern](#commit-reveal-pattern)
4. [Inscription Types](#inscription-types)
5. [Tapscript Construction](#tapscript-construction)
6. [Tapscript Witness Structure](#tapscript-witness-structure)
7. [MuSig2 Integration](#musig2-integration)
8. [Bridge Functionality](#bridge-functionality)
9. [Code References](#code-references)

## Overview

Via L2 uses Tapscript extensively to commit state and operational data to Bitcoin's L1. This provides security, transparency, and verifiability for the L2 system by anchoring critical information in Bitcoin's immutable ledger.

The primary implementation is found in the `core/lib/via_btc_client` module, which contains code for creating, signing, and parsing Tapscript-based inscriptions.

## Inscription Protocol

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

## Commit-Reveal Pattern

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

Via L2 supports several types of inscriptions, each serving a specific purpose in the system:

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

4. **SystemBootstrapping**: Genesis configuration; establishes every trust anchor
   ```rust
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

5. **L1ToL2Message**: Deposit / cross-layer message data
   ```rust
   pub struct L1ToL2MessageInput {
       pub receiver_l2_address: EVMAddress,
       pub l2_contract_address: EVMAddress,
       pub call_data: Vec<u8>,
   }
   ```

6. **SystemContractUpgradeProposal**: Witness-based protocol upgrade proposal
   ```rust
   pub struct SystemContractUpgradeProposalInput {
       pub version: ProtocolSemanticVersion,
       pub bootloader_code_hash: H256,
       pub default_account_code_hash: H256,
       pub evm_emulator_code_hash: Option<H256>,
       pub recursion_scheduler_level_vk_hash: H256,
       pub system_contracts: Vec<(EVMAddress, H256)>,
   }
   ```

7. **UpdateBridgeProposal**: Witness-based bridge migration proposal (on-wire tag: `UpgradeBridgeProposal`)
   ```rust
   pub struct UpdateBridgeProposalInput {
       pub bridge_musig2_address: BitcoinAddress<NetworkUnchecked>,
       pub verifier_p2wpkh_addresses: Vec<BitcoinAddress<NetworkUnchecked>>,
   }
   ```

See `inscription_interaction.md` for the full type catalog, including the OP_RETURN-based execution messages.

> There is no `ProposeSequencer` inscription type anymore; sequencer rotation happens via the `VIA_PROTOCOL:SEQ` OP_RETURN flow.

Each inscription type has a corresponding message identifier defined as a static `PushBytesBuf` value:

```rust
pub static ref SYSTEM_BOOTSTRAPPING_MSG: PushBytesBuf =
    PushBytesBuf::from(b"SystemBootstrappingMessage");
pub static ref PROPOSE_SEQUENCER_MSG: PushBytesBuf =
    PushBytesBuf::from(b"ProposeSequencerMessage"); // legacy, parse-only
pub static ref VALIDATOR_ATTESTATION_MSG: PushBytesBuf =
    PushBytesBuf::from(b"ValidatorAttestationMessage");
pub static ref L1_BATCH_DA_REFERENCE_MSG: PushBytesBuf =
    PushBytesBuf::from(b"L1BatchDAReferenceMessage");
pub static ref PROOF_DA_REFERENCE_MSG: PushBytesBuf =
    PushBytesBuf::from(b"ProofDAReferenceMessage");
pub static ref L1_TO_L2_MSG: PushBytesBuf = PushBytesBuf::from(b"L1ToL2Message");
pub static ref SYSTEM_CONTRACT_UPGRADE_MSG: PushBytesBuf =
    PushBytesBuf::from(b"SystemContractUpgradeProposal");
pub static ref UPGRADE_BRIDGE_MSG: PushBytesBuf =
    PushBytesBuf::from(b"UpgradeBridgeProposal");
```

## Tapscript Construction

The `InscriptionData` struct in `script_builder.rs` encapsulates the Tapscript construction:

```rust
pub struct InscriptionData {
    pub inscription_script: ScriptBuf,
    pub script_size: usize,
    pub script_pubkey: ScriptBuf,
    pub taproot_spend_info: TaprootSpendInfo,
}
```

The `construct_inscription_commitment_data` method shows how the Taproot commitment is created:

```rust
fn construct_inscription_commitment_data<C: Signing + Verification>(
    secp: &Secp256k1<C>,
    inscription_script: &ScriptBuf,
    internal_key: UntweakedPublicKey,
    network: Network,
) -> Result<(ScriptBuf, TaprootSpendInfo)> {
    let mut builder = TaprootBuilder::new();
    builder = builder
        .add_leaf(0, inscription_script.clone())
        .context("adding leaf should work")?;

    let taproot_spend_info = builder
        .finalize(secp, internal_key)
        .map_err(|e| anyhow::anyhow!("Failed to finalize taproot spend info: {:?}", e))?;

    let taproot_address = Address::p2tr_tweaked(taproot_spend_info.output_key(), network);

    let script_pubkey = taproot_address.script_pubkey();

    Ok((script_pubkey, taproot_spend_info))
}
```

This method:
1. Creates a `TaprootBuilder`
2. Adds the inscription script as a leaf
3. Finalizes the Taproot spend info with the internal key
4. Creates a P2TR address and script pubkey

## Tapscript Witness Structure

The Tapscript witness for spending a Taproot output consists of three components:

1. The signature
2. The script (inscription_script)
3. The control block (which proves the script is part of the Taproot commitment)

This is shown in the `sign_reveal_tx` method in `inscriber/mod.rs`:

```rust
let mut witness_data: Witness = Witness::new();

witness_data.push(reveal_input_signature.to_vec());
witness_data.push(inscription_data.inscription_script.to_bytes());

// add control block to witness
let control_block = input.control_block.clone();
witness_data.push(control_block.serialize());
```

The `parse_system_input` method in `indexer/parser.rs` shows how these witnesses are parsed:

```rust
let signature = match TaprootSignature::from_slice(&witness[0]) {
    Ok(sig) => sig,
    Err(e) => {
        warn!("Failed to parse Taproot signature: {}", e);
        return None;
    }
};
let script = ScriptBuf::from_bytes(witness[1].to_vec());
let control_block = match ControlBlock::decode(&witness[2]) {
    Ok(cb) => cb,
    Err(e) => {
        warn!("Failed to decode control block: {}", e);
        return None;
    }
};
```

## MuSig2 Integration

Via L2 uses MuSig2 for creating Taproot signatures, with the Taproot tweak applied to the aggregated public key. This is implemented in the `via_musig2` module.

The `Signer::new` method in `via_musig2/src/lib.rs` shows how the Taproot tweak is calculated and applied. The tweak takes an optional script-tree `merkle_root` (used when the bridge address commits to a governance script path); passing `None` yields a key-path-only output:

```rust
// Convert to bitcoin XOnlyPublicKey first
let internal_key = bitcoin::XOnlyPublicKey::from_slice(&xonly_agg_key.serialize())?;

// Calculate taproot tweak
let tap_tweak = TapTweakHash::from_key_and_tweak(internal_key, merkle_root);
let tweak = tap_tweak.to_scalar();
let tweak_bytes = tweak.to_be_bytes();
let musig2_compatible_tweak = secp256k1_musig2::Scalar::from_be_bytes(tweak_bytes)?;
// Apply tweak to the key aggregation context before signing
musig_key_agg_cache = musig_key_agg_cache
    .with_xonly_tweak(musig2_compatible_tweak)?;
```

This ensures that the MuSig2 signatures are compatible with Taproot's key path spending, and that the same `merkle_root` used to derive the bridge address is used at signing time (a mismatch makes signatures invalid for the address).

## Bridge Functionality

The bridge component uses Taproot/Tapscript for secure multi-signature operations for deposits and withdrawals. The `via_withdrawal_client` module contains code for parsing withdrawal messages and creating withdrawal transactions.

The `TransactionBuilder` in `via_musig2/src/transaction_builder.rs` computes the Taproot key-spend sighashes for signing. The current API is plural: `get_tr_sighashes()` returns one sighash per transaction input (each input must be signed over its own sighash), replacing the older single-input `get_tr_sighash()`:

```rust
pub fn get_tr_sighashes(&self, unsigned_tx: &UnsignedBridgeTx) -> Result<Vec<Vec<u8>>> {
    let mut sighash_cache = SighashCache::new(&unsigned_tx.tx);
    let sighash_type = TapSighashType::All;

    let txout_list: Vec<TxOut> = unsigned_tx
        .utxos
        .iter()
        .map(|(_, txout)| txout.clone())
        .collect();

    let mut sighashes = Vec::new();
    for (i, _) in txout_list.iter().enumerate() {
        let sighash = sighash_cache
            .taproot_key_spend_signature_hash(i, &Prevouts::All(&txout_list), sighash_type)
            .context("Error taproot_key_spend_signature_hash")?;
        sighashes.push(sighash.to_raw_hash().to_byte_array().to_vec());
    }

    Ok(sighashes)
}
```

Each input's sighash is signed in its own MuSig2 session: the withdrawal verifier keeps one `Signer` per UTXO input (`signer_per_utxo_input: BTreeMap<usize, Signer>` in `via_verifier_coordinator/src/verifier/mod.rs`).

## Code References

The primary code modules related to Tapscript usage are:

1. **core/lib/via_btc_client/src/inscriber/script_builder.rs**: Contains code for building Tapscript inscriptions
2. **core/lib/via_btc_client/src/inscriber/mod.rs**: Implements the commit-reveal pattern for inscriptions
3. **core/lib/via_btc_client/src/types.rs**: Defines the different types of inscriptions
4. **core/lib/via_btc_client/src/indexer/parser.rs**: Contains code for parsing Tapscript inscriptions
5. **via_verifier/lib/via_musig2/src/lib.rs**: Implements MuSig2 for Taproot signatures
6. **via_verifier/lib/via_musig2/src/transaction_builder.rs**: Contains code for building transactions with Taproot signatures
7. **via_verifier/lib/via_withdrawal_client/src/withdraw.rs**: Contains code for parsing withdrawal messages

These modules work together to provide a comprehensive implementation of Tapscript usage in the Via L2 system, enabling secure and verifiable state commitments, validator attestations, system bootstrapping, sequencer proposals, and L1-to-L2 messaging.