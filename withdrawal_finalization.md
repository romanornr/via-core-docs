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

1. WithdrawalClient (`via_verifier/lib/via_withdrawal_client/src/client.rs`)
   - Retrieves withdrawal requests from the data availability layer
   - Parses and validates withdrawal messages

2. WithdrawalSession (`via_verifier/node/via_verifier_coordinator/src/sessions/withdrawal.rs`)
   - Identifies finalized L1 batches with pending withdrawals
   - Creates and verifies unsigned withdrawal transactions
   - Manages the withdrawal session lifecycle

3. TransactionBuilder (`via_verifier/lib/via_musig2/src/transaction_builder.rs`)
   - Constructs Bitcoin transactions for withdrawals
   - Manages UTXOs for the bridge address
   - Creates OP_RETURN outputs with proof references

4. ViaWithdrawalVerifier (`via_verifier/node/via_verifier_coordinator/src/verifier/mod.rs`)
   - Coordinates the MuSig2 signing process among verifiers
   - Signs and broadcasts the final withdrawal transaction

5. RestApi (`via_verifier/node/via_verifier_coordinator/src/coordinator/api_decl.rs` and `api_impl.rs`)
   - Provides API endpoints for verifiers to participate in signing sessions
   - Manages nonce and signature collection

### 2.2 Supporting Components

1. MuSig2 Implementation (`via_verifier/lib/via_musig2/src/lib.rs`)
   - Implements the MuSig2 multi-signature protocol
   - Integrates with Bitcoin's Taproot functionality

2. Data Availability Client (`via_da_client`)
   - Interfaces with Celestia to retrieve batch and withdrawal data

3. Bitcoin Client (`via_btc_client`)
   - Interfaces with the Bitcoin network to broadcast transactions and manage UTXOs

## 3. Withdrawal Finalization Process

The withdrawal finalization process consists of several sequential steps:

### 3.1 Withdrawal Detection

1. The `WithdrawalSession` pulls unprocessed withdrawals from the database (populated when withdrawals from finalized batches are indexed) and builds the session directly:

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

Each withdrawal is keyed by an id (hex, embedded in the OP_RETURN data of its output) and the batch is capped at `WITHDRAWAL_LIMIT` entries with a 660 sat minimum per withdrawal.

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

3. Withdrawal messages are parsed from L2→L1 messages and cross-validated against the `Withdrawal` event log:

```rust
// via_verifier/lib/via_withdrawal_client/src/withdraw.rs
pub fn parse_l2_withdrawal_message(
    l2_to_l1_message: Vec<u8>,
    log: Log,
    network: Network,
) -> anyhow::Result<WithdrawalRequest> {
    // Message layout: 4-byte selector ++ variable-length receiver ++ 32-byte amount
    let message_len = l2_to_l1_message.len();
    let address_size = message_len - 36;

    let func_selector_bytes = &l2_to_l1_message[0..4];
    // (must equal the finalizeEthWithdrawal selector, used as a format tag)

    let address_bytes = &l2_to_l1_message[4..4 + address_size];
    let address_str =
        String::from_utf8(address_bytes.to_vec()).with_context(|| "Parse address to string")?;
    let receiver = BitcoinAddress::from_str(&address_str)
        .with_context(|| "parse bitcoin address")?
        .require_network(network)?;

    // The last 32 bytes represent the amount (uint256)
    let amount_bytes = &l2_to_l1_message[address_size + 4..];
    let amount = Amount::from_sat(U256::from_big_endian(amount_bytes).as_u64());

    // Cross-check against the Withdrawal event log: same receiver, same amount
    let l2_sender = Address::from_slice(&log.topics[1].as_bytes()[12..]);
    let tokens = decode(&[ParamType::Bytes, ParamType::Uint(256)], &log.data.0)?;
    // ... decode log receiver + amount, compare with message values ...
}
```

The receiver is a **variable-length Bitcoin address** (`address_size = message_len - 36`), and the parsed result carries full provenance:

```rust
// via_verifier/lib/via_verifier_types/src/withdrawal.rs
pub struct WithdrawalRequest {
    pub id: String,
    pub receiver: BitcoinAddress,
    pub amount: Amount,
    pub l2_sender: Address,
    pub l2_tx_hash: String,
    pub l2_tx_log_index: u16,
}
```

The `id` is what later appears in the bridge transaction's OP_RETURN data, tying each Bitcoin output back to the exact L2 withdrawal it pays.

### 3.2 Transaction Construction

1. The unsigned transaction is built inside `session()` (shown above): withdrawal outputs carry their withdrawal ids as OP_RETURN data, and the `TransactionBuilder` receives everything through a `TransactionBuilderConfig`.

2. The `TransactionBuilder` constructs one or more Bitcoin transactions (splitting by weight and output count):

```rust
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
```

Each resulting transaction contains:
   - Outputs for each withdrawal recipient
   - An OP_RETURN output with the `VIA_WI` prefix, a version byte, and the withdrawal ids paid by this transaction
   - A change output back to the bridge address (if needed)

See `musig2_implementation.md` for `TransactionBuilderConfig` and `build_bridge_txs` internals.

3. Every verifier (not just the coordinator) verifies the session before signing:

```rust
// via_verifier/node/via_verifier_coordinator/src/sessions/withdrawal.rs
async fn verify_message(&self, session_op: &SessionOperation) -> anyhow::Result<bool> {
    let messages = session_op.get_message_to_sign();
    let unsigned_tx = session_op.get_unsigned_bridge_tx();

    if !self._verify_withdrawals(&session_op).await? {
        tracing::error!("Failed to verify session withdrawals");
        return Ok(false);
    }

    if !self._verify_sighashes(&unsigned_tx, &messages).await? {
        tracing::error!("Failed to verify session message");
        return Ok(false);
    }

    Ok(true)
}
```

The verification includes a **fee-rate tolerance check**: the fee rate baked into the unsigned transaction must be within ±1 sat/vbyte of the verifier's own current estimate:

```rust
// via_verifier/node/via_verifier_coordinator/src/sessions/withdrawal.rs
let used_fee_rate = session_op.get_unsigned_bridge_tx().fee_rate;

// Acceptable if difference is within ±1 sat/vbyte
if (used_fee_rate as i32 - fee_rate as i32).abs() > 1 {
    // reject the session
}
```

### 3.2.1 Fee distribution and rounding in withdrawals (MuSig2)

- Total fee estimation is based on the transaction virtual size. To avoid underpayment due to integer division when splitting fees across multiple withdrawal outputs, the fee strategy rounds up to the next multiple of the output count: let `r = base_fee % output_count`; if `r == 0` then `total_fee = base_fee`; else `total_fee = base_fee + (output_count - r)`; see [rust FeeStrategy::estimate_fee_sats()](via_verifier/lib/via_musig2/src/fee.rs:22).
- Per-user fee is computed from the total fee divided by the number of withdrawal outputs. If there are no withdrawal outputs (e.g., all withdrawals are too small after fee filtering), [rust UnsignedBridgeTx::get_fee_per_user()](via_verifier/lib/via_verifier_types/src/transaction.rs:96) returns the total fee to prevent division-by-zero and keep accounting consistent.
- Builder no-op guard: If the constructed unsigned withdrawal transaction is effectively empty (no withdrawal outputs remain), it is not inserted into the UTXO manager; see [rust TransactionBuilder::build_transaction_with_op_return()](via_verifier/lib/via_musig2/src/transaction_builder.rs:58). This prevents stale UTXO tracking for non-broadcast transactions.
- Coordinator validation uses explicit semantics for user amounts after fees: `user_should_receive = amount - fee_per_user`, and checks that each output value matches this expectation; see [rust WithdrawalSession](via_verifier/node/via_verifier_coordinator/src/sessions/withdrawal.rs:113).

- Output count consistency check: The coordinator validates that the number of adjusted withdrawals equals the number of withdrawal outputs in the unsigned transaction, accounting for OP_RETURN and change outputs (i.e., `adjusted_withdrawals.len() + 2 == unsigned_tx.tx.output.len()`); see [rust WithdrawalSession](via_verifier/node/via_verifier_coordinator/src/sessions/withdrawal.rs:43).
### 3.3 MuSig2 Signing Process

1. The Coordinator initiates a signing session:

```rust
// via_verifier/node/via_verifier_coordinator/src/coordinator/api_impl.rs
pub async fn new_session(
    State(self_): State<Arc<Self>>,
) -> anyhow::Result<Response<String>, ApiError> {
    let current_session_op_opt = {
        let current_session_op_opt =
            self_.state.signing_session.read().await.session_op.clone();
        current_session_op_opt
    };

    if let Some(current_session) = current_session_op_opt {
        if self_
            .session_manager
            .is_session_in_progress(&current_session)
            .await?
        {
            return Ok(ok_json(""));
        }

        if !self_.is_session_timeout().await {
            // wait for the session timeout before replacing it
            return Ok(ok_json(""));
        }
    }

    if let Some(session_op) = self_.session_manager.get_next_session().await? {
        // ... install the new SigningSession ...
    }
    // ...
}
```

The session state tracks nonces and partial signatures **per input** (outer key = input index, inner key = signer index):

```rust
// via_verifier/node/via_verifier_coordinator/src/types.rs
#[derive(Default, Debug, Clone)]
pub struct SigningSession {
    pub session_op: Option<SessionOperation>,
    pub received_nonces: BTreeMap<usize, BTreeMap<usize, PubNonce>>,
    pub received_sigs: BTreeMap<usize, BTreeMap<usize, PartialSignature>>,
    pub created_at: u64,
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
    // ... verify_message, then per-input nonce and partial-signature submission ...
}
```

3. The Coordinator collects nonces and partial signatures through the REST endpoints (`submit_nonce`, `submit_partial_signature`), storing them in the per-input nested maps of `SigningSession` shown above. Each verifier runs one MuSig2 round per transaction input.

### 3.4 Final Signature Aggregation and Transaction Broadcasting

1. The Coordinator aggregates the partial signatures into one final signature **per input**:

```rust
// via_verifier/node/via_verifier_coordinator/src/verifier/mod.rs
pub async fn create_final_signature(&mut self, messages: &[Vec<u8>]) -> anyhow::Result<()> {
    if !self.final_sig_per_utxo_input.is_empty() {
        return Ok(());
    }

    let signatures = self.get_session_signatures().await?;
    let input_count = self.signer_per_utxo_input.len();

    if signatures.len() != input_count {
        anyhow::bail!(
            "Mismatch: expected signatures for {} inputs, but got {}",
            input_count,
            signatures.len()
        );
    }

    if messages.len() != input_count {
        anyhow::bail!(
            "Mismatch: expected messages for {} inputs, but got {}",
            input_count,
            messages.len()
        );
    }

    let mut final_sig_per_utxo_input = BTreeMap::new();
    // ... per input: feed partial signatures into that input's Signer,
    //     finalize, verify against that input's sighash, store ...
}
```

2. Each input's aggregated signature is applied to its own witness:

```rust
// via_verifier/node/via_verifier_coordinator/src/verifier/mod.rs
fn sign_transaction(&self, unsigned_tx: UnsignedBridgeTx) -> String {
    let mut unsigned_tx = unsigned_tx;
    let sighash_type = TapSighashType::All;
    for (input_index, musig2_signature) in self.final_sig_per_utxo_input.clone() {
        let mut final_sig_with_hashtype = musig2_signature.serialize().to_vec();
        final_sig_with_hashtype.push(sighash_type as u8);
        unsigned_tx.tx.input[input_index].witness =
            Witness::from(vec![final_sig_with_hashtype.clone()]);
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
        // ... broadcast_signed_transaction, after_broadcast_final_transaction,
        //     reinit signers ...
    }
    // ...
}
```

4. The withdrawals are marked as processed in one database transaction:

```rust
// via_verifier/node/via_verifier_coordinator/src/sessions/withdrawal.rs
async fn after_broadcast_final_transaction(
    &self,
    txid: Txid,
    session_op: &SessionOperation,
) -> anyhow::Result<bool> {
    let mut storage = self
        .master_connection_pool
        .connection_tagged("verifier task")
        .await?;
    let mut transaction = storage.start_transaction().await?;

    let id = transaction
        .via_withdrawal_dal()
        .insert_bridge_withdrawal_tx(&txid.as_byte_array().to_vec())
        .await?;

    let l1_withdrawals = self
        .parse_bridge_withdrawal(session_op.get_unsigned_bridge_tx().tx.clone())
        .await?;

    let withdrawals = get_withdrawal_requests(l1_withdrawals);

    transaction
        .via_withdrawal_dal()
        .mark_withdrawals_as_processed(id, &withdrawals)
        .await?;

    transaction.commit().await?;

    self.transaction_builder
        .utxo_manager_insert_transaction(session_op.get_unsigned_bridge_tx().tx.clone())
        .await;

    tracing::info!("Final withdrawal transaction broadcasted: txid {}", txid);

    Ok(true)
}
```

Note the elegant closure here: the coordinator re-parses its **own** broadcast transaction with the same `parse_bridge_withdrawal` logic the indexer uses, so the set of withdrawals marked processed is exactly what the transaction actually pays. The unsigned tx is also inserted into the UTXO manager's pending context so the next session can chain on its change output before confirmation.

### 3.4 Multi-transaction withdrawals (weight-based splitting)

- When the estimated transaction weight would exceed Bitcoin standard policy limits, the TransactionBuilder splits outputs across multiple bridge transactions, each constrained by `max_tx_weight`.
- `build_transaction_with_op_return` returns a vector of unsigned transactions (`Vec<UnsignedBridgeTx>`); the withdrawal session signs one per session (`unsigned_txs[0]`), and remaining withdrawals are picked up by subsequent sessions since they stay unprocessed in the database until paid.
- The weight limit is configurable via `ViaVerifierConfig::max_tx_weight()` (`core/lib/config/src/configs/via_verifier.rs`); the default reserves a safety buffer below the policy limit:

```rust
pub fn max_tx_weight(&self) -> u64 {
    // Reserve 20000 weight units below Bitcoin's standard limit as a safety buffer
    self.max_tx_weight
        .unwrap_or((MAX_STANDARD_TX_WEIGHT - 20000).into())
}
```

- References: weight constants and estimation in `via_verifier/lib/via_musig2/src/constants.rs` and `transaction_builder.rs`; withdrawal persistence in `via_verifier/lib/verifier_dal/src/withdrawals_dal.rs`; chunking tests in `via_verifier/lib/via_musig2/src/test/`

### 3.5 Withdrawal intake from Bitcoin (BTC Watch)

When any verifier's BTC Watch sees an executed bridge withdrawal on Bitcoin (a `VIA_WI` OP_RETURN spend from the bridge), the `WithdrawalProcessor` reconciles it idempotently, independent of who broadcast it:

```rust
// via_verifier/node/via_btc_watch/src/message_processors/withdrawal.rs
if let FullInscriptionMessage::BridgeWithdrawal(withdrawal_msg) = msg {
    let tx_id = withdrawal_msg.common.tx_id.as_byte_array().to_vec();
    let withdrawals = get_withdrawal_requests(withdrawal_msg.input.withdrawals);

    let id_opt = storage
        .via_withdrawal_dal()
        .get_bridge_withdrawal_id(&tx_id)
        .await?;

    let mut transaction = storage.start_transaction().await?;

    let id = match id_opt {
        Some(id) => id,
        None => {
            transaction
                .via_withdrawal_dal()
                .insert_bridge_withdrawal_tx(&tx_id)
                .await?
        }
    };

    transaction
        .via_withdrawal_dal()
        .mark_bridge_withdrawal_tx_as_processed(&tx_id)
        .await?;

    transaction
        .via_withdrawal_dal()
        .insert_withdrawals(&withdrawals)
        .await?;

    transaction
        .via_withdrawal_dal()
        .mark_withdrawals_as_processed(id, &withdrawals)
        .await?;

    transaction.commit().await?;

    METRICS.withdrawal_confirmed.inc();
}
```

The withdrawal set comes from the parsed `BridgeWithdrawalInput.withdrawals` (recovered from the transaction's outputs and OP_RETURN ids), so a non-coordinator verifier that never saw the signing session still converges to the same processed-withdrawals state.

## 4. Bitcoin Transaction Structure

The final Bitcoin withdrawal transaction has the following structure:

### 4.1 Inputs

- One or more UTXOs from the bridge address (MuSig2 multi-signature address)
- Each input uses Taproot key path spending with the aggregated MuSig2 signature

### 4.2 Outputs

1. Withdrawal Outputs: One output per withdrawal in this transaction
2. OP_RETURN Output: `VIA_WI` prefix, a 1-byte `WithdrawalVersion`, then the ids of the withdrawals paid by this transaction
3. Change Output (optional): Returns any excess funds back to the bridge address

### 4.3 Witness Structure

The witness for each input consists of a single item: **that input's** aggregated MuSig2 signature with the SIGHASH_ALL flag appended. Each input carries a different signature, produced by its own signing session over its own sighash (see `sign_transaction` in section 3.4).

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
- The OP_RETURN output ties each Bitcoin output to a specific L2 withdrawal id, making the payment set independently checkable

### 7.3 Transaction Validation

- The withdrawal transaction is verified to ensure it correctly includes all withdrawals
- Every verifier recomputes the per-input sighashes and checks the fee rate is within ±1 sat/vbyte of its own estimate before signing
- The transaction is signed using the MuSig2 protocol with the SIGHASH_ALL flag

### 7.4 Fee-Aware Validation Enhancements

Recent commits add fee-aware validation to the withdrawal finalization pipeline:

- Network fee rate tolerance
  - The fee rate used to build the unsigned withdrawal transaction must be within ±1 sat/vB of the current network estimate
  - Reference: [via_verifier/node/via_verifier_coordinator/src/sessions/withdrawal.rs](https://github.com/vianetwork/via-core/blob/main/via_verifier/node/via_verifier_coordinator/src/sessions/withdrawal.rs#L369)

- Equal fee distribution across outputs
  - Fees are calculated once and distributed equally across all withdrawal outputs using a strategy pattern
  - References:
    - Fee strategy trait and implementation: [via_verifier/lib/via_musig2/src/fee.rs](https://github.com/vianetwork/via-core/blob/main/via_verifier/lib/via_musig2/src/fee.rs#L9), [via_verifier/lib/via_musig2/src/fee.rs](https://github.com/vianetwork/via-core/blob/main/via_verifier/lib/via_musig2/src/fee.rs#L44), [via_verifier/lib/via_musig2/src/fee.rs](https://github.com/vianetwork/via-core/blob/main/via_verifier/lib/via_musig2/src/fee.rs#L52)
    - Per-user fee helper: [via_verifier/lib/via_verifier_types/src/transaction.rs](https://github.com/vianetwork/via-core/blob/main/via_verifier/lib/via_verifier_types/src/transaction.rs#L17)

- Output value checks with fee deduction
  - During verification, each withdrawal output is validated to match the expected user amount minus the per-user fee
  - Reference: [via_verifier/node/via_verifier_coordinator/src/sessions/withdrawal.rs](https://github.com/vianetwork/via-core/blob/main/via_verifier/node/via_verifier_coordinator/src/sessions/withdrawal.rs#L427)

- Filtering of uneconomical withdrawals
  - Withdrawals that cannot cover their share of the fee are excluded from the unsigned transaction prior to signing
  - Reference: [via_verifier/lib/via_musig2/src/fee.rs](https://github.com/vianetwork/via-core/blob/main/via_verifier/lib/via_musig2/src/fee.rs#L52)

See also:
- Bridge-level validation and indexing details: [bridge.md](bridge.md)
- Fee strategy design and behavior: [fee_mechanism.md](fee_mechanism.md)

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