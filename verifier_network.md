# Via L2 Bitcoin ZK-Rollup: Verifier Network Documentation

## 1. Introduction

The Verifier Network is a critical component of the Via L2 Bitcoin ZK-Rollup protocol, responsible for validating Zero-Knowledge (ZK) proofs and ensuring the integrity of off-chain execution. This document focuses specifically on the network architecture, participant roles, and communication mechanisms between Verifier nodes.

## 2. Network Architecture

The Verifier Network follows a hub-and-spoke architecture with a designated Coordinator node:

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

### 2.1 Key Architectural Characteristics

1. **Hub-and-Spoke Model**: All regular Verifier nodes communicate with a central Coordinator node rather than directly with each other.
2. **Centralized Coordination**: The Coordinator node manages signing sessions and orchestrates the multi-signature process.
3. **REST API Communication**: Verifier nodes interact with the Coordinator through a REST API.
4. **Shared Bitcoin Monitoring**: All nodes independently monitor the Bitcoin blockchain for relevant inscriptions.
5. **Independent Proof Verification**: Each Verifier independently verifies ZK proofs and submits attestations.

### 2.2 Configuration and Deployment

The Verifier Network is configured through environment variables, with each node's role defined by `ViaNodeRole` (the old `VerifierMode` enum was replaced):

```rust
// via_verifier/bin/verifier_server/src/node_builder.rs
is_coordinator: configs.via_verifier_config.role == ViaNodeRole::Coordinator,
```

The Verifier node is assembled from layers; the coordinator additionally gets the REST API layer. The current layer set (`node_builder.rs`) includes:

```
add_sigint_handler_layer
add_healthcheck_layer
add_circuit_breaker_checker_layer
add_prometheus_exporter_layer
add_pools_layer
add_btc_client_layer
add_via_da_client_layer
add_btc_sender_layer
add_btc_watcher_layer
add_zkp_verification_layer
add_verifier_coordinator_api_layer   (coordinator only)
add_withdrawal_verifier_task_layer
add_block_reverter_layer
add_storage_initialization_layer
```

The block reverter layer is worth noting: verifier nodes can automatically roll back their state after a hard Bitcoin reorg (see `via_verifier/node/via_block_reverter/`).

## 3. Participant Roles and Responsibilities

### 3.1 Regular Verifier Node

Regular Verifier nodes are responsible for:

1. **ZK Proof Verification**: Validating cryptographic proofs that certify the correctness of L2 state transitions.
2. **Data Availability Verification**: Ensuring that batch data is available on the Celestia network.
3. **Bitcoin Attestation**: Providing attestations for valid proofs by inscribing them on the Bitcoin network.
4. **Withdrawal Signature Contribution**: Participating in the MuSig2 multi-signature process for withdrawal transactions.

Implementation:

```rust
// via_verifier/node/via_verifier_coordinator/src/verifier/mod.rs
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

Signing state is kept **per UTXO input**: each input of the bridge transaction has its own `Signer` (its own MuSig2 nonce/partial-signature rounds) and its own aggregated `CompactSignature`, because Taproot key-path spends sign a distinct sighash per input.

The Verifier node periodically polls the Coordinator for new signing sessions. The current loop gates on reorg/sync state and re-validates the verifier set before touching a session:

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
        // ... skip empty or already-processed sessions ...
    }

    if session_info.session_op.is_empty() {
        return Ok(());
    }
    let session_op = SessionOperation::from_bytes(&session_info.session_op);
    // ... per-input nonce submission, partial signature submission, final broadcast ...
}
```

### 3.2 Coordinator Node

The Coordinator node is a special Verifier node with additional responsibilities:

1. **Signing Session Management**: Initiating and managing MuSig2 signing sessions for withdrawals.
2. **Nonce Collection**: Collecting and distributing public nonces from all Verifiers.
3. **Partial Signature Collection**: Collecting partial signatures from all Verifiers.
4. **Signature Aggregation**: Combining partial signatures into a complete signature.
5. **Transaction Broadcasting**: Sending the signed withdrawal transaction to the Bitcoin network.

Implementation:

```rust
// via_verifier/node/via_verifier_coordinator/src/coordinator/api_decl.rs
pub struct RestApi {
    pub state: ViaWithdrawalState,
    pub session_manager: SessionManager,
}

// via_verifier/node/via_verifier_coordinator/src/coordinator/api_decl.rs
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

## 4. Communication Protocols

### 4.1 REST API

Session timeout gating:
- New config [rust ViaVerifierConfig::session_timeout](/core/lib/config/src/configs/via_verifier.rs:26) controls the maximum duration of an active signing session (seconds). Default is 30; see [etc/env/base/via_verifier.toml](/etc/env/base/via_verifier.toml:48).
- The Coordinator defers creating a new session if one is in progress and the timeout has not elapsed:
  - Timeout predicate: [rust RestApi::is_session_timeout](/via_verifier/node/via_verifier_coordinator/src/coordinator/api_decl.rs:137)
  - Enforcement when starting a new session: [rust RestApi::new_session](/via_verifier/node/via_verifier_coordinator/src/coordinator/api_impl.rs:151)

The Coordinator exposes a REST API that Verifier nodes use to participate in signing sessions:

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/session/new` | POST | Create a new signing session |
| `/session` | GET | Get the current signing session |
| `/session/nonce` | POST | Submit a public nonce |
| `/session/nonce` | GET | Get all submitted nonces |
| `/session/signature` | POST | Submit a partial signature |
| `/session/signature` | GET | Get all submitted signatures |

Authentication is implemented using a timestamp and signature-based scheme:

```rust
// via_verifier/node/via_verifier_coordinator/src/verifier/mod.rs
fn create_request_headers(&self) -> anyhow::Result<header::HeaderMap> {
    let mut headers = header::HeaderMap::new();
    let timestamp = chrono::Utc::now().timestamp().to_string();
    let verifier_index = self.signer.signer_index().to_string();

    let private_key = bitcoin::PrivateKey::from_wif(&self.config.private_key)?;
    let secret_key = private_key.inner;

    // Sign timestamp + verifier_index as a JSON object
    let payload = serde_json::json!({
        "timestamp": timestamp,
        "verifier_index": verifier_index,
    });
    let signature = crate::auth::sign_request(&payload, &secret_key)?;

    headers.insert("X-Timestamp", header::HeaderValue::from_str(&timestamp)?);
    headers.insert("X-Verifier-Index", header::HeaderValue::from_str(&verifier_index)?);
    headers.insert("X-Signature", header::HeaderValue::from_str(&signature)?);

    Ok(headers)
}
```

### 4.2 Bitcoin Blockchain Monitoring

All Verifier nodes independently monitor the Bitcoin blockchain for relevant inscriptions using the `VerifierBtcWatch` component:

```rust
// via_verifier/node/via_btc_watch/src/lib.rs
pub struct VerifierBtcWatch {
    config: ViaBtcWatchConfig,
    indexer: BitcoinInscriptionIndexer,
    pool: ConnectionPool<Verifier>,
    system_wallet_processor: Box<dyn MessageProcessor>,
    message_processors: Vec<Box<dyn MessageProcessor>>,
}
```

Polling interval and confirmation depth now live in `ViaBtcWatchConfig`, and the last-processed block is persisted in the database (`via_indexer_dal().get_last_processed_l1_block(...)`) instead of held in memory. A dedicated `system_wallet_processor` runs before the regular processors to apply governance-driven wallet rotations.

The `VerifierMessageProcessor` handles two types of messages:

1. **ProofDAReference**: References to proof data stored on Celestia
2. **ValidatorAttestation**: Attestations from other verifiers

```rust
// via_verifier/node/via_btc_watch/src/message_processors/verifier.rs
async fn process_messages(
    &mut self,
    storage: &mut Connection<'_, Verifier>,
    msgs: Vec<FullInscriptionMessage>,
    indexer: &mut BitcoinInscriptionIndexer,
) -> Result<(), MessageProcessorError> {
    for msg in msgs {
        match msg {
            ref f @ FullInscriptionMessage::ProofDAReference(ref proof_msg) => {
                // Process proof reference
                // ...
            }
            ref f @ FullInscriptionMessage::ValidatorAttestation(ref attestation_msg) => {
                // Process attestation
                // ...
                
                // Check finalization
                if votes_dal
                    .finalize_transaction_if_needed(
                        votable_transaction_id,
                        self.zk_agreement_threshold,
                        indexer.get_number_of_verifiers(),
                    )
                    .await
                    .map_err(|e| MessageProcessorError::DatabaseError(e.to_string()))?
                {
                    // Transaction finalized
                }
            }
            // Other message types...
        }
    }
    Ok(())
}
```

### 4.3 Celestia Data Availability Interaction

Verifier nodes interact with the Celestia network to retrieve batch and proof data:

```rust
// via_verifier/node/via_zk_verifier/src/lib.rs
async fn process_proof_da_reference(
    &mut self,
    proof_msg: &ProofDAReference,
) -> anyhow::Result<(InclusionData, BitcoinTxid)> {
    let blob = self
        .da_client
        .get_inclusion_data(&proof_msg.input.blob_id)
        .await
        .with_context(|| "Failed to fetch the blob")?
        .ok_or_else(|| anyhow::anyhow!("Blob not found"))?;
    let batch_tx_id = proof_msg.input.l1_batch_reveal_txid;
    
    Ok((blob, batch_tx_id))
}
```

## 5. MuSig2 Multi-Signature Implementation

The Verifier Network uses the MuSig2 protocol for multi-signature transactions, implemented in the `via_musig2` library:

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

The MuSig2 protocol consists of two rounds:

1. **Round 1 (Nonce Generation)**:
   - Each signer generates a nonce pair and shares the public nonce
   - The Coordinator collects all public nonces

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
```

2. **Round 2 (Partial Signature Generation)**:
   - Each signer creates a partial signature using all public nonces
   - The Coordinator collects all partial signatures and aggregates them

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
```

3. **Signature Aggregation**:
   - The Coordinator combines all partial signatures into a complete signature

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

## 6. Withdrawal Processing Flow

The withdrawal processing flow involves multiple components working together:

1. **Session Management**:
   - The Coordinator identifies finalized L1 batches with pending withdrawals
   - It creates a new signing session for these withdrawals

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

2. **Transaction Building**:
   - Withdrawals are read from the withdrawal DAL (minimum 660 sats each, up to `WITHDRAWAL_LIMIT` per session), each carrying its withdrawal id as per-output OP_RETURN data
   - The `TransactionBuilder` splits the set into multiple transactions when weight (`MAX_STANDARD_TX_WEIGHT`) or output-count limits are hit; the session processes one transaction at a time
   - The session payload contains one sighash per input (`get_tr_sighashes`); see `musig2_implementation.md` for the builder internals

3. **MuSig2 Signing Process**:
   - Verifiers participate in the MuSig2 protocol to sign the transaction
   - The Coordinator collects and aggregates signatures

4. **Transaction Broadcasting**:
   - The Coordinator broadcasts the signed transaction to the Bitcoin network
   - The verifier attempts to build and broadcast the final withdrawal transaction earlier once quorum is met; see [`via_verifier_coordinator/src/verifier/mod.rs`](https://github.com/vianetwork/via-core/blob/main/via_verifier/node/via_verifier_coordinator/src/verifier/mod.rs).
   - The DAL sets updated_at alongside bridge_tx_id and provides helpers to fetch by proof_reveal_tx_id and to update by l1_batch_number; see [via_verifier/lib/verifier_dal/src/via_votes_dal.rs](https://github.com/vianetwork/via-core/blob/main/via_verifier/lib/verifier_dal/src/lib.rs).

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
        // ... broadcast_signed_transaction, after_broadcast_final_transaction ...
    }
    // ...
}
```

Aggregation produces one `CompactSignature` per input (`final_sig_per_utxo_input`); `sign_transaction` attaches each input's signature to its witness before broadcast. The session hooks (`before_process_session`, `before_broadcast_final_transaction`, `after_broadcast_final_transaction`) guard against double-processing across restarts.

## 7. Interactions with L1 (Bitcoin) and Data Availability (Celestia)

### 7.1 Bitcoin Interaction

The Verifier Network interacts with the Bitcoin blockchain in several ways:

1. **Monitoring for Inscriptions**:
   - Watching for new inscriptions related to batches and proofs
   - Processing attestations from other verifiers

2. **Submitting Attestations**:
   - Inscribing attestations for valid proofs

```rust
// via_verifier/node/via_btc_watch/src/lib.rs
async fn loop_iteration(
    &mut self,
    storage: &mut Connection<'_, Verifier>,
) -> Result<(), MessageProcessorError> {
    if storage
        .via_l1_block_dal()
        .has_reorg_in_progress()
        .await?
        .is_some()
    {
        return Ok(());
    }

    let last_processed_bitcoin_block = storage
        .via_indexer_dal()
        .get_last_processed_l1_block(VerifierBtcWatch::module_name())
        .await? as u32;

    if last_processed_bitcoin_block == 0 {
        return Err(MessageProcessorError::Internal(anyhow::anyhow!(
            "The indexer was not initialized".to_string()
        )));
    }
    // ... fetch protocol version, process blocks through system_wallet_processor
    //     and message_processors, persist the new last-processed block ...
}
```

Two changes from earlier revisions: the loop pauses entirely while a reorg marker is present (`via_l1_block_dal().has_reorg_in_progress()`), and indexer progress is persisted in the database per module rather than held in memory, so a restarted verifier resumes exactly where it left off.

3. **Broadcasting Withdrawals**:
   - Sending signed withdrawal transactions to the Bitcoin network

### 7.2 Celestia Interaction

The Verifier Network interacts with Celestia for data availability:

1. **Retrieving Batch Data**:
   - Fetching batch data using blob IDs from inscriptions

2. **Retrieving Proof Data**:
   - Fetching proof data for verification

```rust
// via_verifier/node/via_zk_verifier/src/lib.rs
async fn process_batch_da_reference(
    &mut self,
    batch_msg: &L1BatchDAReference,
) -> anyhow::Result<(InclusionData, H256)> {
    let blob = self
        .da_client
        .get_inclusion_data(&batch_msg.input.blob_id)
        .await
        .with_context(|| "Failed to fetch the blob")?
        .ok_or_else(|| anyhow::anyhow!("Blob not found"))?;
    let hash = batch_msg.input.l1_batch_hash;
    
    Ok((blob, hash))
}
```

## 8. Current Limitations and Future Improvements

### 8.1 Current Limitations

1. **Fixed Coordinator Role**:
   - One of the Verifiers holds the Coordinator role, creating a potential single point of failure
   - The Coordinator is determined by configuration rather than dynamic selection

```rust
// via_verifier/bin/verifier_server/src/node_builder.rs
is_coordinator: configs.via_verifier_config.role == ViaNodeRole::Coordinator,
```

2. **N-of-N Signature Scheme**:
   - All Verifiers must participate in the signing process
   - If any single Verifier becomes unresponsive, the entire network may face difficulties processing withdrawals

3. **Static Verifier Set**:
   - The current architecture does not support dynamic addition or removal of Verifiers
   - Verifier public keys are configured statically

```rust
// via_verifier/node/via_verifier_coordinator/src/types.rs
pub struct ViaWithdrawalState {
    pub signing_session: Arc<RwLock<SigningSession>>,
    pub verifiers_pub_keys: Vec<bitcoin::secp256k1::PublicKey>,
    pub verifier_request_timeout: u8,
    pub session_timeout: u64,
}
```

`required_signers` is no longer a config field; the coordinator derives it from the verifier set, hardcoding n-of-n:

```rust
// via_verifier/node/via_verifier_coordinator/src/coordinator/api_impl.rs
required_signers: self_.state.verifiers_pub_keys.len(),
```

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

The Verifier Network is a critical component of the Via L2 Bitcoin ZK-Rollup system, ensuring the integrity and security of the L2 state transitions. It follows a hub-and-spoke architecture with a designated Coordinator node that manages signing sessions and orchestrates the multi-signature process for withdrawals.

While the current implementation has some limitations, such as the fixed Coordinator role and n-of-n signature scheme, it provides a solid foundation for secure verification of ZK proofs and processing of withdrawals. Future improvements will focus on making the network more decentralized, fault-tolerant, and open to dynamic participation.