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
│   ├── compute_musig2.rs
│   ├── coordinator.rs
│   ├── key_generation_setup.rs
│   ├── musig2_with_script_path.rs
│   ├── transfer_utxos_from_bridge.rs
│   ├── verify_partial_sig.rs
│   └── withdrawal.rs
└── src/
    ├── constants.rs
    ├── fee.rs
    ├── lib.rs
    ├── test/
    ├── transaction_builder.rs
    ├── types.rs
    ├── utils.rs
    └── utxo_manager.rs
```

## 3. Where and Why MuSig2 is Used

### 3.1 Primary Use Case: Withdrawal Transactions

MuSig2 is primarily used for signing withdrawal transactions that transfer funds from the Via L2 system back to the Bitcoin network. This process requires multiple Verifier nodes to collectively sign a Bitcoin transaction.

The coordinator's withdrawal session pulls unprocessed withdrawals straight from the withdrawal DAL, attaches each withdrawal's id as per-output OP_RETURN data, and hands everything to the `TransactionBuilder` through a `TransactionBuilderConfig`:

```rust
// via_verifier/node/via_verifier_coordinator/src/sessions/withdrawal.rs
async fn session(&self) -> anyhow::Result<Option<SessionOperation>> {
    let mut storage = self
        .master_connection_pool
        .connection_tagged("verifier task")
        .await?;

    // Set the minimum amount to withdraw + fee = 660 sats.
    let min_value = 660;

    let no_processed_withdrawals = storage
        .via_withdrawal_dal()
        .list_no_processed_withdrawals(min_value, WITHDRAWAL_LIMIT)
        .await?;

    if no_processed_withdrawals.is_empty() {
        return Ok(None);
    }

    let mut outputs = vec![];
    for w in no_processed_withdrawals {
        let mut op_return_data = Vec::new();
        op_return_data.extend_from_slice(&hex::decode(w.id)?);

        outputs.push(TransactionOutput {
            output: TxOut {
                value: w.amount,
                script_pubkey: w.receiver.script_pubkey(),
            },
            op_return_data: Some(op_return_data),
        });
    }

    let mut op_return_prefix = Vec::new();
    op_return_prefix.extend_from_slice(OP_RETURN_WITHDRAW_PREFIX);
    op_return_prefix.push(WITHDRAWAL_VERSION as u8);

    let config = TransactionBuilderConfig {
        fee_strategy: Arc::new(WithdrawalFeeStrategy::new()),
        max_tx_weight: MAX_STANDARD_TX_WEIGHT as u64,
        max_output_per_tx: WITHDRAWAL_LIMIT as usize,
        op_return_prefix,
        bridge_address: self.get_system_wallets().await?.bridge,
        default_fee_rate_opt: None,
        default_available_utxos_opt: None,
        op_return_data_input_opt: None,
    };

    let unsigned_txs = self
        .transaction_builder
        .build_transaction_with_op_return(outputs, config)
        .await?;

    if unsigned_txs.is_empty() {
        return Ok(None);
    }

    let sig_hashes = self
        .transaction_builder
        .get_tr_sighashes(&unsigned_txs[0])?;

    Ok(Some(SessionOperation::Withdrawal(
        unsigned_txs[0].clone(),
        sig_hashes,
    )))
}
```

Notable properties of the current flow:

- Withdrawals below `min_value` (660 sats, covering dust plus fee) are filtered out at the DAL level
- The `VIA_WI` OP_RETURN prefix is followed by a 1-byte `WITHDRAWAL_VERSION`
- Large withdrawal sets are split across multiple bridge transactions by weight (`max_tx_weight`, based on `bitcoin::policy::MAX_STANDARD_TX_WEIGHT`) and by output count (`max_output_per_tx`); the session signs one transaction at a time
- The `SessionOperation::Withdrawal` payload carries one sighash **per input** (`get_tr_sighashes`), because each input is signed in its own MuSig2 session
- Weight math constants (INPUT_BASE_SIZE, INPUT_WITNESS_SIZE, OUTPUT_SIZE, OP_RETURN_SIZE, TX_OVERHEAD, WITNESS_OVERHEAD) live in `via_verifier/lib/via_musig2/src/constants.rs`; the witness-aware fee model is in `via_verifier/lib/via_musig2/src/fee.rs`; chunking and weight enforcement are covered by tests in `via_verifier/lib/via_musig2/src/test/`

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
// merkle_root is the optional script-tree root: Some(..) when the bridge
// address commits to a governance script path, None for key-path-only
let tap_tweak = TapTweakHash::from_key_and_tweak(internal_key, merkle_root);
let tweak = tap_tweak.to_scalar();
let tweak_bytes = tweak.to_be_bytes();
let musig2_compatible_tweak = secp256k1_musig2::Scalar::from_be_bytes(tweak_bytes)
    .with_context(|| TAPROOT_TWEAK_SCALAR_RANGE_ERR)
    .map_err(|e| MusigError::Musig2Error(e.to_string()))?;
// Apply tweak to the key aggregation context before signing
musig_key_agg_cache = musig_key_agg_cache
    .with_xonly_tweak(musig2_compatible_tweak)
    .map_err(|e| MusigError::Musig2Error(format!("Failed to apply tweak: {}", e)))?;
```

The `merkle_root` passed here must be the same one used when deriving the bridge address; a mismatch produces signatures that are invalid for the address.

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

The `transaction_builder.rs` module handles the construction of Bitcoin transactions that will be signed using MuSig2. The entry point takes a `TransactionBuilderConfig` and can return **multiple** unsigned transactions when the withdrawal set exceeds the weight or output limits:

```rust
// via_verifier/lib/via_musig2/src/types.rs
#[derive(Clone)]
pub struct TransactionBuilderConfig {
    /// The fee strategy
    pub fee_strategy: Arc<dyn FeeStrategy>,
    /// The max tx weight
    pub max_tx_weight: u64,
    /// The max number of output to include in each transaction
    pub max_output_per_tx: usize,
    /// The OP_RETURN prefix
    pub op_return_prefix: Vec<u8>,
    /// Bridge address
    pub bridge_address: Address,
    /// The fee rate.
    pub default_fee_rate_opt: Option<u64>,
    /// The transaction fee rate
    pub default_available_utxos_opt: Option<Vec<(OutPoint, TxOut)>>,
    // ...
}

// via_verifier/lib/via_musig2/src/transaction_builder.rs
pub async fn build_transaction_with_op_return(
    &self,
    outputs: Vec<TransactionOutput>,
    config: TransactionBuilderConfig,
) -> Result<Vec<UnsignedBridgeTx>> {
    self.utxo_manager.sync_context_with_blockchain().await?;

    let available_utxos = self.get_available_utxos(&config).await?;
    let fee_rate = self.get_fee_rate(&config).await?;

    self.build_bridge_txs(available_utxos, outputs, config, fee_rate)
        .await
}

pub async fn build_bridge_txs(
    &self,
    available_utxos: Vec<(OutPoint, TxOut)>,
    outputs: Vec<TransactionOutput>,
    config: TransactionBuilderConfig,
    fee_rate: u64,
) -> Result<Vec<UnsignedBridgeTx>> {
    let output_chunks = self.chunk_outputs(&outputs, config.max_output_per_tx);
    let mut utxos_pool = available_utxos;
    let mut bridge_txs = Vec::new();
    // ... per-chunk selection, fee calculation, weight enforcement ...
}
```

The result type carries fee metadata alongside the transaction (`via_verifier/lib/via_verifier_types/src/transaction.rs`):

```rust
pub struct UnsignedBridgeTx {
    pub tx: Transaction,
    pub txid: Txid,
    pub utxos: Vec<(OutPoint, TxOut)>,
    pub change_amount: Amount,
    pub fee: Amount,
    pub fee_rate: u64,
}
```

Sighash computation is per input; each input gets its own MuSig2 signing session:

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

Correspondingly, the withdrawal verifier maintains per-input signing state (`via_verifier_coordinator/src/verifier/mod.rs`):

```rust
pub struct ViaWithdrawalVerifier {
    verifier_config: ViaVerifierConfig,
    wallet: ViaWallet,
    session_manager: SessionManager,
    btc_client: Arc<dyn BitcoinOps>,
    master_connection_pool: ConnectionPool<Verifier>,
    client: Client,
    signer_per_utxo_input: BTreeMap<usize, Signer>,
    final_sig_per_utxo_input: BTreeMap<usize, CompactSignature>,
    // ...
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
    self.session_manager.prepare_session().await?;

    if self.state.is_reorg_in_progress().await? {
        return Ok(());
    }

    if self.state.is_sync_in_progress().await? {
        return Ok(());
    }

    self.validate_verifier_addresses().await?;

    let mut session_info = self.get_session().await?;

    if self.is_coordinator() {
        self.create_new_session().await?;
        session_info = self.get_session().await?;

        if session_info.session_op.is_empty() {
            tracing::debug!("Empty session, nothing to process");
            return Ok(());
        }

        let session_op = SessionOperation::from_bytes(&session_info.session_op);

        if !self
            .session_manager
            .before_process_session(&session_op)
            .await?
        {
            tracing::debug!("Session already processed");
            return Ok(());
        }
    }

    if session_info.session_op.is_empty() {
        tracing::debug!("Empty session, nothing to process");
        return Ok(());
    }
    let session_op = SessionOperation::from_bytes(&session_info.session_op);
    // ... nonce submission, partial signature submission, final broadcast ...
}
```

The loop refuses to participate while a Bitcoin reorg or state sync is in progress, and re-validates the verifier address set every iteration. Nonce and partial-signature submission then proceed per input, mirroring the per-input signer map.

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
    let input_index = 0;
    let received_partial_signatures = session_info
        .received_partial_signatures
        .get(&input_index)
        .map_or(0, |len| *len);

    if received_partial_signatures < session_info.required_signers {
        return Ok(false);
    }

    let unsigned_tx = session_op.get_unsigned_bridge_tx();
    let messages = session_op.get_message_to_sign();

    self.create_final_signature(&messages)
        .await
        .map_err(|e| anyhow::format_err!("Error create final signature: {e}"))?;

    if !self.final_sig_per_utxo_input.is_empty() {
        if !self
            .session_manager
            .before_broadcast_final_transaction(session_op)
            .await?
        {
            return Ok(false);
        }

        let signed_tx = self.sign_transaction(unsigned_tx.clone());
        // ... broadcast_signed_transaction, update state ...
    }
    // ...
}
```

Note the per-input bookkeeping: partial-signature counts are tracked per input index, the final aggregation fills `final_sig_per_utxo_input` (one `CompactSignature` per input), and `sign_transaction` attaches each input's aggregated Schnorr signature to its witness.

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
// via_verifier/node/via_verifier_coordinator/src/types.rs
pub struct ViaWithdrawalState {
    pub signing_session: Arc<RwLock<SigningSession>>,
    pub verifiers_pub_keys: Vec<bitcoin::secp256k1::PublicKey>,
    pub verifier_request_timeout: u8,
    pub session_timeout: u64,
}
```

There is no separate `required_signers` configuration: the coordinator derives it as the full verifier set, making n-of-n structural rather than configurable:

```rust
// via_verifier/node/via_verifier_coordinator/src/coordinator/api_impl.rs
required_signers: self_.state.verifiers_pub_keys.len(),
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