# Via L2 Bitcoin ZK-Rollup: MuSig2 Implementation Documentation

## 1. Introduction

This document details the implementation and usage of the MuSig2 multi-signature scheme within the Via L2 Bitcoin ZK-Rollup protocol. MuSig2 is a crucial component that enables secure, multi-party signing of Bitcoin transactions, particularly for processing withdrawals from the L2 to the Bitcoin network.

> **Security Update**: The MuSig2 implementation has been updated to use `OsRng` from the `rand` crate for nonce generation instead of `rand::random()`. This change improves the security of the random number generation process used in the signing protocol, which is critical for the security of the multi-signature scheme.

## 2. MuSig2 Library and Dependencies

### 2.1 Core Libraries

The Via L2 system uses the following libraries for its MuSig2 implementation:

```toml
# via_verifier/lib/via_musig2/Cargo.toml
musig2 = "0.2.0"
secp256k1_musig2 = { package = "secp256k1", version = "0.30.0", features = [
    "rand",
    "hashes",
] }
use rand::{rngs::OsRng, Rng};
```

The implementation is built on:
- `musig2` crate (v0.2.0) - Provides the core MuSig2 protocol implementation
- `secp256k1` with MuSig2 features - Handles the underlying elliptic curve cryptography

### 2.2 Custom Wrapper Library

Via L2 implements a custom wrapper library (`via_musig2`) that encapsulates the MuSig2 functionality and integrates it with the rest of the system:

```
via_verifier/lib/via_musig2/
├── Cargo.toml
├── examples/
│   ├── coordinator.rs
│   ├── key_generation_setup.rs
│   └── withdrawal.rs
└── src/
    ├── lib.rs
    ├── transaction_builder.rs
    └── utxo_manager.rs
```

## 3. Where and Why MuSig2 is Used

### 3.1 Primary Use Case: Withdrawal Transactions

MuSig2 is primarily used for signing withdrawal transactions that transfer funds from the Via L2 system back to the Bitcoin network. This process requires multiple Verifier nodes to collectively sign a Bitcoin transaction.

```rust
// via_verifier/node/via_verifier_coordinator/src/sessions/withdrawal.rs
pub async fn create_unsigned_tx(
    &self,
    withdrawals: Vec<WithdrawalRequest>,
    proof_txid: Txid,
) -> anyhow::Result<UnsignedBridgeTx> {
    // Group withdrawals by address and sum amounts
    let mut grouped_withdrawals: HashMap<Address, Amount> = HashMap::new();
    for w in withdrawals {
        *grouped_withdrawals.entry(w.address).or_insert(Amount::ZERO) = grouped_withdrawals
            .get(&w.address)
            .unwrap_or(&Amount::ZERO)
            .checked_add(w.amount)
            .ok_or_else(|| anyhow::anyhow!("Withdrawal amount overflow when grouping"))?;
    }

    // Create outputs for grouped withdrawals
    let outputs: Vec<TxOut> = grouped_withdrawals
        .into_iter()
        .map(|(address, amount)| TxOut {
            value: amount,
            script_pubkey: address.script_pubkey(),
        })
        .collect();

    self.transaction_builder
        .build_transaction_with_op_return(
            outputs,
            OP_RETURN_WITHDRAW_PREFIX,
            vec![proof_txid.as_raw_hash().to_byte_array()],
        )
        .await
}
```
> Update Transaction weight-based splitting and API changes
>
> - Large withdrawals are now split across multiple bridge transactions when the estimated transaction weight approaches the standard policy limit.
> - Constants and weight math live in [via_verifier/lib/via_musig2/src/constants.rs](https://github.com/vianetwork/via-core/blob/main/via_verifier/lib/via_musig2/src/constants.rs#L1) and include INPUT_BASE_SIZE, INPUT_WITNESS_SIZE, OUTPUT_SIZE, OP_RETURN_SIZE, TX_OVERHEAD, WITNESS_OVERHEAD, FIXED_OVERHEAD_WEIGHT, and AVAILABLE_WEIGHT (based on bitcoin::policy::MAX_STANDARD_TX_WEIGHT).
> - Fee calculation was updated to be witness-aware and align with the new sizing model: see [via_verifier/lib/via_musig2/src/fee.rs](https://github.com/vianetwork/via-core/blob/main/via_verifier/lib/via_musig2/src/fee.rs).
> - TransactionBuilder APIs:
>   - [rust TransactionBuilder::estimate_transaction_weight()](https://github.com/vianetwork/via-core/blob/main/via_verifier/lib/via_musig2/src/transaction_builder.rs) estimates total weight from input/output counts.
>   - [rust TransactionBuilder::get_transaction_metadata()](https://github.com/vianetwork/via-core/blob/main/via_verifier/lib/via_musig2/src/transaction_builder.rs) partitions inputs/outputs into chunks that fit within a configurable weight limit.
>   - [rust TransactionBuilder::build_bridge_txs()](https://github.com/vianetwork/via-core/blob/main/via_verifier/lib/via_musig2/src/transaction_builder.rs) materializes signed-ready transaction skeletons from the metadata chunks.
>   - [rust TransactionBuilder::build_transaction_with_op_return()](https://github.com/vianetwork/via-core/blob/main/via_verifier/lib/via_musig2/src/transaction_builder.rs) now returns Vec&lt;UnsignedBridgeTx&gt; and accepts a weight_limit parameter (e.g., MAX_STANDARD_TX_WEIGHT) and op_return_data as Vec&lt;&amp;Vec&lt;u8&gt;&gt;.
>
> Updated signature (simplified):
>
> ```rust
> // via_verifier/lib/via_musig2/src/transaction_builder.rs
> pub async fn build_transaction_with_op_return(
>     &self,
>     outputs: Vec<TxOut>,
>     op_return_prefix: &[u8],
>     op_return_data: Vec<&Vec<u8>>,
>     fee_strategy: Arc<dyn FeeStrategy>,
>     change_output: Option<TxOut>,
>     max_outputs_per_tx: Option<usize>,
>     weight_limit: u64, // e.g. bitcoin::policy::MAX_STANDARD_TX_WEIGHT
> ) -> anyhow::Result<Vec<UnsignedBridgeTx>> { /* ... */ }
> ```
>
> Example usage (examples updated to pick the first tx when only a single transaction is needed):
>
> - [via_verifier/lib/via_musig2/examples/coordinator.rs](https://github.com/vianetwork/via-core/blob/main/via_verifier/lib/via_musig2/examples/coordinator.rs#L447)
> - [via_verifier/lib/via_musig2/examples/withdrawal.rs](https://github.com/vianetwork/via-core/blob/main/via_verifier/lib/via_musig2/examples/withdrawal.rs#L129)
>
> Test coverage for chunking and weight enforcement:
>
> - [via_verifier/lib/via_musig2/src/test/bridge_tx.rs](https://github.com/vianetwork/via-core/blob/main/via_verifier/lib/via_musig2/src/test/bridge_tx.rs#L334)
> - [via_verifier/lib/via_musig2/src/test/chunk_outputs.rs](https://github.com/vianetwork/via-core/blob/main/via_verifier/lib/via_musig2/src/test/chunk_outputs.rs#L1)
> - [via_verifier/lib/via_musig2/src/test/mod.rs](https://github.com/vianetwork/via-core/blob/main/via_verifier/lib/via_musig2/src/test/mod.rs#L1)

### 3.2 Security Benefits

MuSig2 is used for several security reasons:

1. **Distributed Trust**: No single entity can unilaterally withdraw funds from the bridge address
2. **Taproot Integration**: Leverages Bitcoin's Taproot functionality for enhanced privacy and efficiency
3. **Threshold Security**: Requires multiple Verifiers to agree on the validity of withdrawals

## 4. Implementation Details

### 4.1 Core Signer Structure

The core of the MuSig2 implementation is the `Signer` struct, which represents a participant in the MuSig2 protocol:

```rust
// via_verifier/lib/via_musig2/src/lib.rs
pub struct Signer {
    secret_key: SecretKey,
    public_key: PublicKey,
    signer_index: usize,
    key_agg_ctx: KeyAggContext,
    first_round: Option<FirstRound>,
    second_round: Option<SecondRound<Vec<u8>>>,
    message: Vec<u8>,
    nonce_submitted: bool,
    partial_sig_submitted: bool,
}
```

### 4.2 Key Aggregation and Taproot Integration

The implementation integrates with Bitcoin's Taproot functionality by applying a taproot tweak to the aggregated public key:

```rust
// via_verifier/lib/via_musig2/src/lib.rs
let mut musig_key_agg_cache = KeyAggContext::new(all_pubkeys).map_err(|e| MusigError::Musig2Error(e.to_string()))?;

let agg_pubkey = musig_key_agg_cache.aggregated_pubkey::<secp256k1_musig2::PublicKey>();
let (xonly_agg_key, _) = agg_pubkey.x_only_public_key();

// Convert to bitcoin XOnlyPublicKey first
let internal_key = bitcoin::XOnlyPublicKey::from_slice(&xonly_agg_key.serialize())?;

// Calculate taproot tweak
let tap_tweak = TapTweakHash::from_key_and_tweak(internal_key, None);
let tweak = tap_tweak.to_scalar();
let tweak_bytes = tweak.to_be_bytes();
let musig2_compatible_tweak = secp256k1_musig2::Scalar::from_be_bytes(tweak_bytes).unwrap();
// Apply tweak to the key aggregation context before signing
musig_key_agg_cache = musig_key_agg_cache
    .with_xonly_tweak(musig2_compatible_tweak)
    .map_err(|e| MusigError::Musig2Error(format!("Failed to apply tweak: {}", e)))?;
```

### 4.3 Multi-Round Signing Protocol

The MuSig2 protocol is implemented as a multi-round process:

#### 4.3.1 Round 1: Nonce Generation and Exchange

```rust
// via_verifier/lib/via_musig2/src/lib.rs
pub fn start_signing_session(&mut self, message: Vec<u8>) -> Result<PubNonce, MusigError> {
    self.message = message.clone();

    let msg_array = message.as_slice();

    let first_round = FirstRound::new(
        self.key_agg_ctx.clone(),
        OsRng.gen::<[u8; 32]>(),
        self.signer_index,
        SecNonceSpices::new()
            .with_seckey(self.secret_key)
            .with_message(&msg_array),
    )
    .map_err(|e| MusigError::Musig2Error(e.to_string()))?;

    let nonce = first_round.our_public_nonce();
    self.first_round = Some(first_round);
    Ok(nonce)
}

pub fn receive_nonce(
    &mut self,
    signer_index: usize,
    nonce: PubNonce,
) -> Result<(), MusigError> {
    let first_round = self
        .first_round
        .as_mut()
        .ok_or_else(|| MusigError::InvalidState("First round not initialized".into()))?;

    first_round
        .receive_nonce(signer_index, nonce)
        .map_err(|e| MusigError::Musig2Error(e.to_string()))?;
    Ok(())
}
```

#### 4.3.2 Round 2: Partial Signature Generation and Exchange

```rust
// via_verifier/lib/via_musig2/src/lib.rs
pub fn create_partial_signature(&mut self) -> Result<PartialSignature, MusigError> {
    let msg_array = self.message.clone();

    let first_round = self
        .first_round
        .take()
        .ok_or_else(|| MusigError::InvalidState("First round not initialized".into()))?;

    let second_round = first_round
        .finalize(self.secret_key, msg_array)
        .map_err(|e| MusigError::Musig2Error(e.to_string()))?;

    let partial_sig = second_round.our_signature();
    self.second_round = Some(second_round);
    Ok(partial_sig)
}

pub fn receive_partial_signature(
    &mut self,
    signer_index: usize,
    partial_sig: PartialSignature,
) -> Result<(), MusigError> {
    let second_round = self
        .second_round
        .as_mut()
        .ok_or_else(|| MusigError::InvalidState("Second round not initialized".into()))?;

    second_round
        .receive_signature(signer_index, partial_sig)
        .map_err(|e| MusigError::Musig2Error(e.to_string()))?;
    Ok(())
}
```

#### 4.3.3 Final Signature Aggregation

```rust
// via_verifier/lib/via_musig2/src/lib.rs
pub fn create_final_signature(&mut self) -> Result<CompactSignature, MusigError> {
    let second_round = self
        .second_round
        .take()
        .ok_or_else(|| MusigError::InvalidState("Second round not initialized".into()))?;

    second_round
        .finalize()
        .map_err(|e| MusigError::Musig2Error(e.to_string()))
}
```

### 4.4 Transaction Building and Signing

Withdrawal fee accounting:
- The fee strategy evenly distributes the total fee across withdrawal outputs and rounds up to the next multiple of the output count to guarantee the transaction is not underfunded with respect to its virtual size (let `r = base_fee % output_count`; if `r == 0` then `total_fee = base_fee`; else `total_fee = base_fee + (output_count - r)`); see [rust FeeStrategy::estimate_fee_sats()](via_verifier/lib/via_musig2/src/fee.rs:22).
- Per-user fee equals `total_fee / outputs_count`. If there are no withdrawal outputs (e.g., all requests are too small after fee filtering), [rust UnsignedBridgeTx::get_fee_per_user()](via_verifier/lib/via_verifier_types/src/transaction.rs:96) returns the total fee to avoid division by zero and keep accounting consistent.
- If the constructed unsigned transaction ends up with no withdrawal outputs, the builder treats it as a no-op and does not insert it into the UTXO manager; see [rust TransactionBuilder::build_transaction_with_op_return()](via_verifier/lib/via_musig2/src/transaction_builder.rs:58).

The `transaction_builder.rs` module handles the construction of Bitcoin transactions that will be signed using MuSig2:

```rust
// via_verifier/lib/via_musig2/src/transaction_builder.rs
pub async fn build_transaction_with_op_return(
    &self,
    mut outputs: Vec<TxOut>,
    op_return_prefix: &[u8],
    op_return_data: Vec<[u8; 32]>,
) -> Result<UnsignedBridgeTx> {
    // ... transaction building logic ...

    // Create unsigned transaction
    let unsigned_tx = Transaction {
        version: transaction::Version::TWO,
        lock_time: absolute::LockTime::ZERO,
        input: inputs,
        output: outputs.clone(),
    };

    let txid = unsigned_tx.compute_txid();

    // ... more logic ...

    Ok(UnsignedBridgeTx {
        tx: unsigned_tx,
        txid,
        utxos: selected_utxos,
        change_amount,
    })
}

pub fn get_tr_sighash(&self, unsigned_tx: &UnsignedBridgeTx) -> anyhow::Result<TapSighash> {
    let mut sighash_cache = SighashCache::new(&unsigned_tx.tx);
    let sighash_type = TapSighashType::All;
    let mut txout_list = Vec::with_capacity(unsigned_tx.utxos.len());

    for (_, txout) in unsigned_tx.utxos.clone() {
        txout_list.push(txout);
    }
    let sighash = sighash_cache
        .taproot_key_spend_signature_hash(0, &Prevouts::All(&txout_list), sighash_type)
        .with_context(|| "Error taproot_key_spend_signature_hash")?;

    Ok(sighash)
}
```

## 5. Verifier Network Integration

### 5.1 Network Architecture

The MuSig2 implementation is integrated into the Verifier Network, which follows a hub-and-spoke architecture:

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Verifier 1    │     │   Verifier 2    │     │   Verifier n-1  │
└────────┬────────┘     └────────┬────────┘     └────────┬────────┘
         │                       │                       │
         │                       │                       │
         │                       ▼                       │
         │             ┌─────────────────────┐           │
         └────────────►│  Coordinator Node   │◄──────────┘
                       │   (Verifier n)      │
                       └─────────────────────┘
```

### 5.2 Coordinator Role

The Coordinator node manages signing sessions and orchestrates the MuSig2 protocol:

```rust
// via_verifier/node/via_verifier_coordinator/src/coordinator/api_decl.rs
pub struct RestApi {
    pub state: ViaWithdrawalState,
    pub session_manager: SessionManager,
}

pub fn into_router(self) -> axum::Router<()> {
    // ...
    let router = axum::Router::new()
        .route("/new", axum::routing::post(Self::new_session))
        .route("/", axum::routing::get(Self::get_session))
        .route("/signature", axum::routing::post(Self::submit_partial_signature))
        .route("/signature", axum::routing::get(Self::get_submitted_signatures))
        .route("/nonce", axum::routing::post(Self::submit_nonce))
        .route("/nonce", axum::routing::get(Self::get_nonces))
        // ...
}
```

### 5.3 Verifier Node Role

Regular Verifier nodes participate in the MuSig2 protocol by submitting nonces and partial signatures:

```rust
// via_verifier/node/via_verifier_coordinator/src/verifier/mod.rs
async fn loop_iteration(&mut self) -> Result<(), anyhow::Error> {
    let mut session_info = self.get_session().await?;
    
    // ... process session ...
    
    if session_info.received_nonces < session_info.required_signers {
        if !session_nonces.contains_key(&verifier_index) {
            self.submit_nonce().await?;
        }
    } else if session_info.received_nonces >= session_info.required_signers {
        if self.signer.has_created_partial_sig() {
            return Ok(());
        }
        self.submit_partial_signature(session_nonces).await?;
    }
    
    Ok(())
}
```

## 6. Withdrawal Processing Flow

The complete withdrawal processing flow involves multiple components working together:

1. **Session Initiation**:
   - The Coordinator identifies finalized L1 batches with pending withdrawals
   - It creates a new signing session for these withdrawals

2. **Transaction Building**:
   - The Coordinator builds an unsigned transaction for the withdrawals
   - The transaction includes an OP_RETURN output with a reference to the proof transaction

3. **MuSig2 Signing Process**:
   - Verifiers participate in the MuSig2 protocol to sign the transaction
   - The Coordinator collects and aggregates signatures

4. **Transaction Broadcasting**:
   - The Coordinator broadcasts the signed transaction to the Bitcoin network

```rust
// via_verifier/node/via_verifier_coordinator/src/verifier/mod.rs
async fn build_and_broadcast_final_transaction(
    &mut self,
    session_info: &SigningSessionResponse,
    session_op: &SessionOperation,
) -> anyhow::Result<bool> {
    if session_info.received_partial_signatures < session_info.required_signers {
        return Ok(false);
    }

    if let Some((unsigned_tx, message)) = session_op.session() {
        self.create_final_signature(message)
            .await
            .map_err(|e| anyhow::format_err!("Error create final signature: {e}"))?;

        if let Some(musig2_signature) = self.final_sig {
            // ... verify and prepare transaction ...

            let signed_tx = self.sign_transaction(unsigned_tx.clone(), musig2_signature);

            let txid = self
                .btc_client
                .broadcast_signed_transaction(&signed_tx)
                .await?;

            // ... update state ...

            return Ok(true);
        }
    }
    Ok(false)
}
```

## 7. Key Management

### 7.1 Key Generation

Keys are generated for individual participants and then aggregated to create a shared Taproot address:

```rust
// via_verifier/lib/via_musig2/examples/key_generation_setup.rs
fn generate_keypair() -> (SecretKey, PublicKey) {
    let mut rng = OsRng;
    let secp = Secp256k1::new();
    let secret_key = SecretKey::new(&mut rng);
    let public_key = PublicKey::from_secret_key(&secp, &secret_key);
    (secret_key, public_key)
}

fn create_bridge_address(
    pubkeys: Vec<PublicKey>,
) -> Result<BitcoinAddress, Box<dyn std::error::Error>> {
    let secp = bitcoin::secp256k1::Secp256k1::new();

    let musig_key_agg_cache = KeyAggContext::new(pubkeys)?;

    let agg_pubkey = musig_key_agg_cache.aggregated_pubkey::<secp256k1_musig2::PublicKey>();
    let (xonly_agg_key, _) = agg_pubkey.x_only_public_key();

    // Convert to bitcoin XOnlyPublicKey first
    let internal_key = bitcoin::XOnlyPublicKey::from_slice(&xonly_agg_key.serialize())?;

    // Use internal_key for address creation
    let address = BitcoinAddress::p2tr(&secp, internal_key, None, Network::Regtest);

    Ok(address)
}
```

### 7.2 Key Storage and Security

The Verifier nodes store their private keys securely and use them to participate in the MuSig2 protocol. The current implementation uses an n-of-n signature scheme, requiring all Verifiers to participate in the signing process.

```rust
// via_verifier/node/via_verifier_coordinator/src/coordinator/api_decl.rs
let state = ViaWithdrawalState {
    signing_session: Arc::new(RwLock::new(SigningSession::default())),
    required_signers: config.required_signers,
    verifiers_pub_keys: config
        .verifiers_pub_keys_str
        .iter()
        .map(|s| bitcoin::secp256k1::PublicKey::from_str(s).unwrap())
        .collect(),
    verifier_request_timeout: config.verifier_request_timeout,
};
```

## 8. Current Limitations and Future Improvements

### 8.1 Current Limitations

1. **Fixed Coordinator Role**:
   - One of the Verifiers holds the Coordinator role, creating a potential single point of failure
   - The Coordinator is determined by configuration rather than dynamic selection

2. **N-of-N Signature Scheme**:
   - All Verifiers must participate in the signing process
   - If any single Verifier becomes unresponsive, the entire network may face difficulties processing withdrawals

3. **Static Verifier Set**:
   - The current architecture does not support dynamic addition or removal of Verifiers
   - Verifier public keys are configured statically

### 8.2 Future Improvements

1. **More Trust-Minimized Approach**:
   - Potentially incorporating solutions like BitVM-based bridges
   - Reducing reliance on the Verifier Network for processing bridge transactions

2. **Open Verifier Network**:
   - Allowing for dynamic participation of Verifiers
   - Implementing a mechanism for Verifier rotation or election

3. **Improved Fault Tolerance**:
   - Moving away from the n-of-n signature scheme to a more robust approach (e.g., m-of-n)
   - Implementing a mechanism to handle unresponsive Verifiers

## 9. Conclusion

The MuSig2 implementation in the Via L2 Bitcoin ZK-Rollup system provides a secure and efficient way for multiple Verifier nodes to collectively sign withdrawal transactions. By leveraging the MuSig2 protocol and integrating with Bitcoin's Taproot functionality, the system ensures that no single entity can unilaterally withdraw funds from the bridge address.

While the current implementation has some limitations, such as the fixed Coordinator role and n-of-n signature scheme, it provides a solid foundation for secure multi-party signing of Bitcoin transactions. Future improvements will focus on making the system more decentralized, fault-tolerant, and open to dynamic participation.