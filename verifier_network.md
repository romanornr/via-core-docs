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

The Verifier Network is configured through environment variables, with each node having a specific role defined by the `VerifierMode` configuration parameter:

```rust
// via_verifier/bin/verifier_server/src/node_builder.rs
pub fn new(via_general_config: ViaGeneralConfig, secrets: Secrets) -> anyhow::Result<Self> {
    let via_verifier_config = try_load_config!(via_general_config.via_verifier_config);
    let is_coordinator = via_verifier_config.verifier_mode == VerifierMode::COORDINATOR;
    // ...
}
```

The Verifier node is built with different layers depending on its role:

```rust
// via_verifier/bin/verifier_server/src/node_builder.rs
pub fn build(mut self) -> anyhow::Result<ZkStackService> {
    self = self
        .add_sigint_handler_layer()?
        .add_healthcheck_layer()?
        .add_circuit_breaker_checker_layer()?
        .add_pools_layer()?
        .add_btc_sender_layer()?
        .add_verifier_btc_watcher_layer()?
        .add_via_celestia_da_client_layer()?
        .add_zkp_verification_layer()?;

    if self.is_coordinator {
        self = self.add_verifier_coordinator_api_layer()?
    }

    self = self.add_withdrawal_verifier_task_layer()?;

    Ok(self.node.build())
}
```

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
    session_manager: SessionManager,
    btc_client: Arc<dyn BitcoinOps>,
    config: ViaVerifierConfig,
    client: Client,
    signer: Signer,
    final_sig: Option<CompactSignature>,
}
```

The Verifier node periodically polls the Coordinator for new signing sessions:

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
    indexer: BitcoinInscriptionIndexer,
    poll_interval: Duration,
    confirmations_for_btc_msg: u64,
    last_processed_bitcoin_block: u32,
    pool: ConnectionPool<Verifier>,
    message_processors: Vec<Box<dyn MessageProcessor>>,
    btc_blocks_lag: u32,
}
```

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
        rand::random::<[u8; 32]>(),
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
    // Get the l1 batches finilized but withdrawals not yet processed
    let l1_batches = self
        .master_connection_pool
        .connection_tagged("withdrawal session")
        .await?
        .via_votes_dal()
        .list_finalized_blocks_and_non_processed_withdrawals()
        .await?;

    if l1_batches.is_empty() {
        return Ok(None);
    }

    // ... process withdrawals ...

    Ok(Some(SessionOperation::Withdrawal(
        l1_batch_number,
        unsigned_tx,
        sighash.to_byte_array().to_vec(),
        raw_proof_tx_id,
    )))
}
```

2. **Transaction Building**:
   - The Coordinator builds an unsigned transaction for the withdrawals

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

            tracing::info!(
                "Brodcast {} signed transaction with txid {}",
                &session_op.get_session_type(),
                &txid.to_string()
            );

            // ... update state ...

            return Ok(true);
        }
    }
    Ok(false)
}
```

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
    let to_block = self
        .indexer
        .fetch_block_height()
        .await
        .map_err(|e| MessageProcessorError::Internal(anyhow::anyhow!(e.to_string())))?
        .saturating_sub(self.confirmations_for_btc_msg) as u32;

    let messages = self
        .indexer
        .process_blocks(self.last_processed_bitcoin_block + 1, to_block)
        .await
        .map_err(|e| MessageProcessorError::Internal(e.into()))?;

    for processor in self.message_processors.iter_mut() {
        processor
            .process_messages(storage, messages.clone(), &mut self.indexer)
            .await
            .map_err(|e| MessageProcessorError::Internal(e.into()))?;
    }
    
    self.last_processed_bitcoin_block = to_block;
    Ok(())
}
```

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
pub fn new(via_general_config: ViaGeneralConfig, secrets: Secrets) -> anyhow::Result<Self> {
    let via_verifier_config = try_load_config!(via_general_config.via_verifier_config);
    let is_coordinator = via_verifier_config.verifier_mode == VerifierMode::COORDINATOR;
    // ...
}
```

2. **N-of-N Signature Scheme**:
   - All Verifiers must participate in the signing process
   - If any single Verifier becomes unresponsive, the entire network may face difficulties processing withdrawals

3. **Static Verifier Set**:
   - The current architecture does not support dynamic addition or removal of Verifiers
   - Verifier public keys are configured statically

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