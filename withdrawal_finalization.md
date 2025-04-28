# Via L2 Bitcoin ZK-Rollup: Withdrawal Finalization Logic

## 1. Overview

The Withdrawal Finalization Logic in the Via L2 Bitcoin ZK-Rollup system is responsible for securely transferring assets from the L2 rollup back to the Bitcoin L1 blockchain. This document details the end-to-end process of finalizing withdrawals, from their initiation on L2 to the final Bitcoin transaction broadcast.

The withdrawal finalization process is a critical security component of the Via L2 system, ensuring that:
1. Only valid withdrawals that have been included in finalized L2 batches can be processed
2. Withdrawals require cryptographic proofs of L2 state inclusion
3. Multiple verifiers must collectively sign the withdrawal transaction using MuSig2
4. The withdrawal transaction includes references to proof data for verification

## 2. Primary Components

The withdrawal finalization logic involves several key components:

### 2.1 Core Components

1. **WithdrawalClient** (`via_verifier/lib/via_withdrawal_client/src/client.rs`)
   - Retrieves withdrawal requests from the data availability layer
   - Parses and validates withdrawal messages

2. **WithdrawalSession** (`via_verifier/node/via_verifier_coordinator/src/sessions/withdrawal.rs`)
   - Identifies finalized L1 batches with pending withdrawals
   - Creates and verifies unsigned withdrawal transactions
   - Manages the withdrawal session lifecycle

3. **TransactionBuilder** (`via_verifier/lib/via_musig2/src/transaction_builder.rs`)
   - Constructs Bitcoin transactions for withdrawals
   - Manages UTXOs for the bridge address
   - Creates OP_RETURN outputs with proof references

4. **ViaWithdrawalVerifier** (`via_verifier/node/via_verifier_coordinator/src/verifier/mod.rs`)
   - Coordinates the MuSig2 signing process among verifiers
   - Signs and broadcasts the final withdrawal transaction

5. **RestApi** (`via_verifier/node/via_verifier_coordinator/src/coordinator/api_decl.rs` and `api_impl.rs`)
   - Provides API endpoints for verifiers to participate in signing sessions
   - Manages nonce and signature collection

### 2.2 Supporting Components

1. **MuSig2 Implementation** (`via_verifier/lib/via_musig2/src/lib.rs`)
   - Implements the MuSig2 multi-signature protocol
   - Integrates with Bitcoin's Taproot functionality

2. **Data Availability Client** (`via_da_client`)
   - Interfaces with Celestia to retrieve batch and withdrawal data

3. **Bitcoin Client** (`via_btc_client`)
   - Interfaces with the Bitcoin network to broadcast transactions and manage UTXOs

## 3. Withdrawal Finalization Process

The withdrawal finalization process consists of several sequential steps:

### 3.1 Withdrawal Detection

1. The `WithdrawalSession` identifies finalized L1 batches with pending withdrawals:

```rust
// via_verifier/node/via_verifier_coordinator/src/sessions/withdrawal.rs
async fn session(&self) -> anyhow::Result<Option<SessionOperation>> {
    // Get the l1 batches finalized but withdrawals not yet processed
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
    
    // Process withdrawals...
}
```

2. The `WithdrawalClient` retrieves withdrawal requests from the data availability layer:

```rust
// via_verifier/lib/via_withdrawal_client/src/client.rs
pub async fn get_withdrawals(&self, blob_id: &str) -> anyhow::Result<Vec<WithdrawalRequest>> {
    let pubdata_bytes = self.fetch_pubdata(blob_id).await?;
    let pubdata = Pubdata::decode_pubdata(pubdata_bytes)?;
    let l2_bridge_metadata = WithdrawalClient::list_l2_bridge_metadata(&pubdata);
    let withdrawals = WithdrawalClient::get_valid_withdrawals(self.network, l2_bridge_metadata);
    Ok(withdrawals)
}
```

3. Withdrawal messages are parsed from L2 bridge logs:

```rust
// via_verifier/lib/via_withdrawal_client/src/withdraw.rs
pub fn parse_l2_withdrawal_message(
    l2_to_l1_message: Vec<u8>,
    network: Network,
) -> anyhow::Result<WithdrawalRequest> {
    // Extract function selector, Bitcoin address, and amount
    let func_selector_bytes = &l2_to_l1_message[0..4];
    let address_bytes = &l2_to_l1_message[4..4 + address_size];
    let amount_bytes = &l2_to_l1_message[address_size + 4..];
    
    // Parse Bitcoin address and amount
    let address_str = String::from_utf8(address_bytes.to_vec())?;
    let address = BitcoinAddress::from_str(&address_str)?.require_network(network)?;
    let amount = Amount::from_sat(U256::from_big_endian(amount_bytes).as_u64());
    
    Ok(WithdrawalRequest { address, amount })
}
```

### 3.2 Transaction Construction

1. The `WithdrawalSession` creates an unsigned transaction for the withdrawals:

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
            .checked_add(w.amount)?;
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

2. The `TransactionBuilder` constructs the Bitcoin transaction with:
   - Outputs for each withdrawal recipient
   - An OP_RETURN output containing the proof transaction ID
   - A change output back to the bridge address (if needed)

```rust
// via_verifier/lib/via_musig2/src/transaction_builder.rs
pub async fn build_transaction_with_op_return(
    &self,
    mut outputs: Vec<TxOut>,
    op_return_prefix: &[u8],
    op_return_data: Vec<[u8; 32]>,
) -> Result<UnsignedBridgeTx> {
    // Sync UTXOs with blockchain
    self.utxo_manager.sync_context_with_blockchain().await?;
    
    // Calculate total required amount
    let mut total_required_amount: Amount = Amount::ZERO;
    for output in &outputs {
        total_required_amount = total_required_amount.checked_add(output.value)?;
    }
    
    // Select UTXOs and calculate fee
    let selected_utxos = self.utxo_manager.select_utxos_by_target_value(&available_utxos, total_needed).await?;
    
    // Create OP_RETURN output with proof txid
    let op_return_data = TransactionBuilder::create_op_return_script(op_return_prefix, op_return_data)?;
    let op_return_output = TxOut {
        value: Amount::ZERO,
        script_pubkey: op_return_data,
    };
    
    // Add outputs and create transaction
    outputs.push(op_return_output);
    
    // Add change output if needed
    if change_amount.to_sat() > 0 {
        outputs.push(TxOut {
            value: change_amount,
            script_pubkey: self.bridge_address.script_pubkey(),
        });
    }
    
    // Create unsigned transaction
    let unsigned_tx = Transaction {
        version: transaction::Version::TWO,
        lock_time: absolute::LockTime::ZERO,
        input: inputs,
        output: outputs.clone(),
    };
    
    Ok(UnsignedBridgeTx {
        tx: unsigned_tx,
        txid,
        utxos: selected_utxos,
        change_amount,
    })
}
```

3. The transaction is verified to ensure it correctly includes all withdrawals:

```rust
// via_verifier/node/via_verifier_coordinator/src/sessions/withdrawal.rs
async fn _verify_withdrawals(
    &self,
    l1_batch_number: i64,
    unsigned_tx: &UnsignedBridgeTx,
    blob_id: &str,
    proof_tx_id: Vec<u8>,
) -> anyhow::Result<bool> {
    let withdrawals = self.withdrawal_client.get_withdrawals(blob_id).await?;
    
    // Group withdrawals and verify amounts
    // ...
    
    // Verify the OP_RETURN output contains the correct proof txid
    let tx_id = h256_to_txid(&proof_tx_id)?;
    let op_return_data = TransactionBuilder::create_op_return_script(
        OP_RETURN_WITHDRAW_PREFIX,
        vec![*tx_id.as_raw_hash().as_byte_array()],
    )?;
    
    // Verify OP_RETURN output matches expected data
    // ...
    
    Ok(true)
}
```

### 3.3 MuSig2 Signing Process

1. The Coordinator initiates a signing session:

```rust
// via_verifier/node/via_verifier_coordinator/src/coordinator/api_impl.rs
pub async fn new_session(
    State(self_): State<Arc<Self>>,
) -> anyhow::Result<Response<String>, ApiError> {
    // Check if there's an existing session in progress
    
    if let Some(session_op) = self_.session_manager.get_next_session().await? {
        // Create a new signing session
        let new_session = SigningSession {
            session_op: Some(session_op),
            received_nonces: HashMap::new(),
            received_sigs: HashMap::new(),
        };
        
        let mut signing_session = self_.state.signing_session.write().await;
        *signing_session = new_session;
    }
    
    // ...
}
```

2. Verifiers participate in the MuSig2 protocol by:
   - Retrieving the current session
   - Submitting their public nonces
   - Receiving nonces from other verifiers
   - Creating and submitting partial signatures

```rust
// via_verifier/node/via_verifier_coordinator/src/verifier/mod.rs
async fn loop_iteration(&mut self) -> Result<(), anyhow::Error> {
    let mut session_info = self.get_session().await?;
    
    // ...
    
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

3. The Coordinator collects nonces and partial signatures:

```rust
// via_verifier/node/via_verifier_coordinator/src/coordinator/api_impl.rs
pub async fn submit_nonce(
    State(self_): State<Arc<Self>>,
    Json(nonce_pair): Json<NoncePair>,
) -> anyhow::Result<Response<String>, ApiError> {
    // Decode and validate nonce
    
    let mut session = self_.state.signing_session.write().await;
    session.received_nonces.insert(nonce_pair.signer_index, pub_nonce);
    
    Ok(ok_json("Success"))
}

pub async fn submit_partial_signature(
    State(self_): State<Arc<Self>>,
    Json(sig_pair): Json<PartialSignaturePair>,
) -> anyhow::Result<Response<String>, ApiError> {
    // Decode and validate signature
    
    let mut session = self_.state.signing_session.write().await;
    session.received_sigs.insert(sig_pair.signer_index, partial_sig);
    
    Ok(ok_json("Success"))
}
```

### 3.4 Final Signature Aggregation and Transaction Broadcasting

1. The Coordinator aggregates the partial signatures into a final signature:

```rust
// via_verifier/node/via_verifier_coordinator/src/verifier/mod.rs
async fn create_final_signature(&mut self, message: &[u8]) -> anyhow::Result<()> {
    if self.final_sig.is_some() {
        return Ok(());
    }

    let signatures = self.get_session_signatures().await?;
    for (&i, sig) in &signatures {
        if self.signer.signer_index() != i {
            self.signer.receive_partial_signature(i, *sig)?;
        }
    }

    let final_sig = self.signer.create_final_signature()?;
    let agg_pub = self.signer.aggregated_pubkey();
    verify_signature(agg_pub, final_sig, message)?;
    self.final_sig = Some(final_sig);

    Ok(())
}
```

2. The final signature is applied to the transaction:

```rust
// via_verifier/node/via_verifier_coordinator/src/verifier/mod.rs
fn sign_transaction(
    &self,
    unsigned_tx: UnsignedBridgeTx,
    musig2_signature: CompactSignature,
) -> String {
    let mut unsigned_tx = unsigned_tx;
    let mut final_sig_with_hashtype = musig2_signature.serialize().to_vec();
    let sighash_type = TapSighashType::All;
    final_sig_with_hashtype.push(sighash_type as u8);
    for tx in &mut unsigned_tx.tx.input {
        tx.witness = Witness::from(vec![final_sig_with_hashtype.clone()]);
    }
    bitcoin::consensus::encode::serialize_hex(&unsigned_tx.tx)
}
```

3. The signed transaction is broadcast to the Bitcoin network:

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
        self.create_final_signature(message).await?;

        if let Some(musig2_signature) = self.final_sig {
            // Sign transaction
            let signed_tx = self.sign_transaction(unsigned_tx.clone(), musig2_signature);

            // Broadcast to Bitcoin network
            let txid = self.btc_client.broadcast_signed_transaction(&signed_tx).await?;

            tracing::info!(
                "Broadcast {} signed transaction with txid {}",
                &session_op.get_session_type(),
                &txid.to_string()
            );

            // Update state to mark withdrawal as processed
            self.session_manager.after_broadcast_final_transaction(txid, session_op).await?;
            
            self.reinit_signer()?;
            return Ok(true);
        }
    }
    Ok(false)
}
```

4. The withdrawal is marked as processed in the database:

```rust
// via_verifier/node/via_verifier_coordinator/src/sessions/withdrawal.rs
async fn after_broadcast_final_transaction(
    &self,
    txid: Txid,
    session_op: &SessionOperation,
) -> anyhow::Result<bool> {
    self.master_connection_pool
        .connection_tagged("verifier task")
        .await?
        .via_votes_dal()
        .mark_vote_transaction_as_processed(
            H256::from_slice(&txid.as_raw_hash().to_byte_array()),
            &session_op.get_proof_tx_id(),
            session_op.get_l1_batche_number(),
        )
        .await?;

    tracing::info!(
        "New withdrawal transaction processed, l1 batch {} musig2 tx_id {}",
        session_op.get_l1_batche_number(),
        txid
    );

    Ok(true)
}
```

## 4. Bitcoin Transaction Structure

The final Bitcoin withdrawal transaction has the following structure:

### 4.1 Inputs

- One or more UTXOs from the bridge address (MuSig2 multi-signature address)
- Each input uses Taproot key path spending with the aggregated MuSig2 signature

### 4.2 Outputs

1. **Withdrawal Outputs**: One output for each unique recipient address, with the total amount for that address
2. **OP_RETURN Output**: Contains the prefix "VIA_PROTOCOL:WITHDRAWAL:" followed by the proof transaction ID
3. **Change Output** (optional): Returns any excess funds back to the bridge address

### 4.3 Witness Structure

The witness for each input consists of a single item:
- The aggregated MuSig2 signature with the SIGHASH_ALL flag appended

```rust
// via_verifier/node/via_verifier_coordinator/src/verifier/mod.rs
fn sign_transaction(
    &self,
    unsigned_tx: UnsignedBridgeTx,
    musig2_signature: CompactSignature,
) -> String {
    let mut unsigned_tx = unsigned_tx;
    let mut final_sig_with_hashtype = musig2_signature.serialize().to_vec();
    let sighash_type = TapSighashType::All;
    final_sig_with_hashtype.push(sighash_type as u8);
    for tx in &mut unsigned_tx.tx.input {
        tx.witness = Witness::from(vec![final_sig_with_hashtype.clone()]);
    }
    bitcoin::consensus::encode::serialize_hex(&unsigned_tx.tx)
}
```

## 5. Required Proofs and Verification

The withdrawal finalization process relies on several types of proofs:

### 5.1 ZK Proof of L2 State Inclusion

- Each L2 batch has a ZK proof that verifies the correctness of all state transitions
- This proof ensures that the withdrawal was correctly initiated on L2
- The proof is published to Celestia and referenced in a Bitcoin inscription

### 5.2 Data Availability Proof

- The L2 batch data is published to Celestia, ensuring data availability
- The blob ID is included in the Bitcoin inscription, allowing verifiers to retrieve the data

### 5.3 Verifier Attestations

- Verifiers submit attestations to Bitcoin confirming the validity of the ZK proof
- A threshold of attestations is required before a batch is considered finalized
- Only withdrawals from finalized batches can be processed

## 6. Verifier Network Coordination

The withdrawal finalization process is coordinated by the Verifier Network:

### 6.1 Coordinator Role

- One verifier acts as the Coordinator, managing signing sessions
- The Coordinator identifies finalized batches with pending withdrawals
- It creates and manages MuSig2 signing sessions
- It aggregates partial signatures and broadcasts the final transaction

### 6.2 Regular Verifier Role

- Regular verifiers participate in the MuSig2 signing process
- They verify the withdrawal transaction before signing
- They submit nonces and partial signatures to the Coordinator

### 6.3 Communication Protocol

- Verifiers communicate with the Coordinator through a REST API
- Authentication is implemented using a timestamp and signature-based scheme
- The API includes endpoints for session management, nonce submission, and signature submission

## 7. Security Considerations

### 7.1 Multi-Signature Security

- The bridge address is a MuSig2 multi-signature address
- All verifiers must participate in the signing process (n-of-n scheme)
- The MuSig2 protocol ensures that no single verifier can unilaterally withdraw funds

### 7.2 Proof Verification

- Withdrawals are only processed if they come from finalized L2 batches
- Batch finalization requires ZK proof verification and verifier attestations
- The OP_RETURN output in the withdrawal transaction references the proof transaction

### 7.3 Transaction Validation

- The withdrawal transaction is verified to ensure it correctly includes all withdrawals
- The OP_RETURN output is verified to contain the correct proof transaction ID
- The transaction is signed using the MuSig2 protocol with the SIGHASH_ALL flag

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

The Withdrawal Finalization Logic in the Via L2 Bitcoin ZK-Rollup system provides a secure and efficient mechanism for transferring assets from L2 back to the Bitcoin L1 blockchain. It leverages advanced cryptographic techniques like ZK proofs and MuSig2 multi-signatures to ensure the security and integrity of withdrawals.

The process involves multiple steps, from withdrawal detection and transaction construction to MuSig2 signing and transaction broadcasting. Each step includes various security measures to ensure that only valid withdrawals from finalized L2 batches can be processed.

While the current implementation has some limitations, such as the fixed Coordinator role and n-of-n signature scheme, it provides a solid foundation for secure cross-chain transfers. Future improvements will focus on making the system more decentralized, fault-tolerant, and open to dynamic participation.