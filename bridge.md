# Via L2 Bitcoin Bridge Documentation

## Overview

The Via L2 Bitcoin Bridge is a critical component of the Via L2 system, enabling the secure transfer of assets between the L1 (Bitcoin) blockchain and the L2 (Via) rollup. The bridge implements a two-way mechanism for:

1. **Deposits (L1→L2)**: Detecting Bitcoin transactions sent to the bridge address and crediting corresponding assets on L2.
2. **Withdrawals (L2→L1)**: Processing withdrawal requests initiated on L2 and executing the corresponding Bitcoin transactions.

The bridge leverages Bitcoin's native capabilities along with ZK proofs and MuSig2 multi-signature schemes to ensure security, decentralization, and efficiency.

## Architecture

The bridge consists of several interconnected components:

1. **Bitcoin Client**: Interfaces with the Bitcoin network to monitor transactions, fetch UTXOs, and broadcast signed transactions.
2. **Inscription Indexer**: Monitors the Bitcoin blockchain for inscriptions and extracts relevant messages.
3. **Withdrawal Client**: Processes withdrawal requests from L2 to L1.
4. **Verifier Network**: A set of verifiers that collectively sign withdrawal transactions using MuSig2.
5. **Transaction Builder**: Creates and manages Bitcoin transactions for withdrawals.

## Deposit Flow (L1→L2)

### Detection Mechanism

Deposits are detected by the `BitcoinInscriptionIndexer` which monitors the Bitcoin blockchain for transactions sent to the bridge address. The indexer is implemented in `via_btc_client/src/indexer/mod.rs`.

```rust
// Extract important transactions from a block
fn extract_important_transactions(
    &self,
    transactions: &[BitcoinTransaction],
) -> (
    Option<Vec<BitcoinTransaction>>,
    Option<Vec<BitcoinTransaction>>,
) {
    // ...
    let bridge_txs: Vec<BitcoinTransaction> = transactions
        .iter()
        .filter(|tx| {
            // Check if bridge address is in outputs (deposit destination)
            tx.output.iter().any(|output| {
                let script_pubkey = &output.script_pubkey;
                script_pubkey == &self.bridge_address.script_pubkey()
            }) && !tx.output.iter().any(|output| {
                if output.script_pubkey.is_op_return() {
                    // Extract OP_RETURN data
                    if let Some(op_return_data) = output.script_pubkey.as_bytes().get(2..) {
                        // Return true if it starts with withdrawal prefix (which will be negated)
                        op_return_data.starts_with(b"VIA_PROTOCOL:WITHDRAWAL")
                    } else {
                        false
                    }
                } else {
                    false // Not an OP_RETURN output
                }
            })
        })
        .cloned()
        .collect();
    // ...
}
```

The indexer identifies deposit transactions by checking if:
1. The transaction has an output to the bridge address
2. The transaction does not contain an OP_RETURN output with the withdrawal prefix

### Deposit Processing

Deposits are processed by the `L1ToL2MessageProcessor` in `via_verifier/node/via_btc_watch/src/message_processors/l1_to_l2.rs`. This processor:

1. Extracts L1ToL2 messages from Bitcoin transactions
2. Creates L1 transactions for the L2 system
3. Inserts these transactions into the database for processing by the sequencer

```rust
fn create_l1_tx_from_message(
    &self,
    tx_id: H256,
    serial_id: PriorityOpId,
    msg: &L1ToL2Message,
) -> Result<L1ToL2Transaction, MessageProcessorError> {
    let amount = msg.amount.to_sat() as i64;
    let eth_address_l2 = msg.input.receiver_l2_address;
    let calldata = msg.input.call_data.clone();

    let mantissa = U256::from(10_000_000_000u64); // Eth 18 decimals - BTC 8 decimals
    let value = U256::from(amount) * mantissa;
    // ...
    
    // Create L1 transaction for L2 processing
    let mut l1_tx = L1Tx {
        execute: Execute {
            contract_address: eth_address_l2,
            calldata: calldata.clone(),
            value: U256::zero(),
            factory_deps: vec![],
        },
        common_data: L1TxCommonData {
            sender: eth_address_l2,
            serial_id,
            // ...
            to_mint: value,
            refund_recipient: eth_address_l2,
            eth_block: msg.common.block_height as u64,
        },
        received_timestamp_ms: unix_timestamp_ms(),
    };
    // ...
}
```

### Deposit Types

The system supports two types of deposits:

1. **Inscription-based deposits**: Using Bitcoin inscriptions to include additional metadata
2. **OP_RETURN-based deposits**: Using OP_RETURN outputs to specify the L2 recipient address

### Deposit Validation and Processing

Deposits undergo validation before being processed:

1. **L2 Receiver Address Validation**: The system validates that the L2 receiver address is valid by checking that it's greater than or equal to a minimum valid address (`0x0000000000000000000000000000000000010001`). This prevents deposits to invalid or system-reserved addresses.

```rust
// Minimum valid L2 receiver address
const MIN_VALID_L2_RECEIVER_ADDRESS: &str = "0x0000000000000000000000000000000000010001";

// Validation function
pub fn is_valid_deposit(&self) -> bool {
    self.l2_receiver_address >= H160::from_str(MIN_VALID_L2_RECEIVER_ADDRESS).unwrap()
}
```

2. **Deposit Encapsulation**: Deposits are encapsulated in a `ViaL1Deposit` struct that contains:
   - L2 receiver address
   - Deposit amount
   - Calldata for the L2 transaction
   - Serial ID for priority operations
   - L1 block number where the deposit was detected

```rust
pub struct ViaL1Deposit {
    pub l2_receiver_address: Address,
    pub amount: u64,
    pub calldata: Vec<u8>,
    pub serial_id: PriorityOpId,
    pub l1_block_number: u64,
}
```

3. **L1 Transaction Creation**: Valid deposits are converted to L1 transactions for processing by the L2 system:
   - The deposit amount is converted from BTC (8 decimals) to ETH (18 decimals) using a mantissa of 10^10
   - Gas parameters are standardized with fixed values:
     - `MAX_FEE_PER_GAS`: 120,000,000
     - `GAS_LIMIT`: 300,000
     - `GAS_PER_PUBDATA_LIMIT`: Required L1-to-L2 gas per pubdata byte

4. **Invalid Deposit Handling**: Deposits with invalid L2 receiver addresses are logged and skipped, preventing them from being processed by the L2 system.

## Withdrawal Flow (L2→L1)

### Withdrawal Initiation

Withdrawals are initiated on L2 through the L2 bridge contract. The withdrawal request contains:
- The Bitcoin recipient address
- The amount to withdraw

The L2 contract emits a log that is included in the L2 batch data, which is then published to the data availability layer.

### Withdrawal Processing

The withdrawal process involves several steps:

1. **Withdrawal Detection**: The `WithdrawalClient` in `via_verifier/lib/via_withdrawal_client/src/client.rs` fetches withdrawal requests from the data availability layer.

```rust
pub async fn get_withdrawals(&self, blob_id: &str) -> anyhow::Result<Vec<WithdrawalRequest>> {
    let pubdata_bytes = self.fetch_pubdata(blob_id).await?;
    let pubdata = Pubdata::decode_pubdata(pubdata_bytes)?;
    let l2_bridge_metadata = WithdrawalClient::list_l2_bridge_metadata(&pubdata);
    let withdrawals = WithdrawalClient::get_valid_withdrawals(self.network, l2_bridge_metadata);
    Ok(withdrawals)
}
```

2. **Withdrawal Session Creation**: The `WithdrawalSession` in `via_verifier/node/via_verifier_coordinator/src/sessions/withdrawal.rs` creates a withdrawal session for processing.

```rust
async fn session(&self) -> anyhow::Result<Option<SessionOperation>> {
    // Get the l1 batches finalized but withdrawals not yet processed
    let l1_batches = self
        .master_connection_pool
        .connection_tagged("withdrawal session")
        .await?
        .via_votes_dal()
        .list_finalized_blocks_and_non_processed_withdrawals()
        .await?;
    
    // ...
    
    // Create unsigned transaction for withdrawals
    let unsigned_tx = self
        .create_unsigned_tx(withdrawals_to_process, proof_txid)
        .await?;
    
    // ...
}
```

3. **Transaction Building**: The `TransactionBuilder` in `via_verifier/lib/via_musig2/src/transaction_builder.rs` creates the Bitcoin transaction for the withdrawal.

```rust
pub async fn build_transaction_with_op_return(
    &self,
    mut outputs: Vec<TxOut>,
    op_return_prefix: &[u8],
    op_return_data: Vec<[u8; 32]>,
) -> Result<UnsignedBridgeTx> {
    // ...
    
    // Create OP_RETURN output with proof txid
    let op_return_data =
        TransactionBuilder::create_op_return_script(op_return_prefix, op_return_data)?;

    let op_return_output = TxOut {
        value: Amount::ZERO,
        script_pubkey: op_return_data,
    };
    
    // ...
}
```

4. **Multi-Signature Coordination**: The `ViaWithdrawalVerifier` in `via_verifier/node/via_verifier_coordinator/src/verifier/mod.rs` coordinates the MuSig2 signing process among verifiers.

```rust
async fn build_and_broadcast_final_transaction(
    &mut self,
    session_info: &SigningSessionResponse,
    session_op: &SessionOperation,
) -> anyhow::Result<bool> {
    // ...
    
    // Create final signature
    self.create_final_signature(message)
        .await?;

    // Sign and broadcast transaction
    let signed_tx = self.sign_transaction(unsigned_tx.clone(), musig2_signature);
    let txid = self
        .btc_client
        .broadcast_signed_transaction(&signed_tx)
        .await?;
    
    // ...
}
```

5. **Transaction Broadcasting**: Once enough verifiers have signed, the coordinator broadcasts the transaction to the Bitcoin network.

### Withdrawal Security

Withdrawals are secured through several mechanisms:

1. **ZK Proofs**: Each L2 batch is verified with a ZK proof, ensuring the validity of all operations including withdrawals.
2. **MuSig2 Multi-Signatures**: Withdrawals require signatures from a threshold of verifiers using MuSig2.
3. **Verifier Voting**: Verifiers vote on the validity of L2 batches before processing withdrawals.

## Bridge Address Management

The bridge uses a MuSig2 multi-signature address to secure funds. This address is controlled by the verifier network, with a threshold of signatures required to authorize withdrawals.

The bridge address is initialized during system bootstrapping:

```rust
fn process_bootstrap_message(
    state: &mut BootstrapState,
    message: FullInscriptionMessage,
    txid: Txid,
    network: Network,
) {
    match message {
        FullInscriptionMessage::SystemBootstrapping(sb) => {
            // ...
            let bridge_address = sb
                .input
                .bridge_musig2_address
                .require_network(network)
                .unwrap();
            state.bridge_address = Some(bridge_address);
            // ...
        }
        // ...
    }
}
```

## Interactions with Other Components

### L1 Watcher (via_btc_watch)

The L1 Watcher monitors the Bitcoin blockchain for relevant transactions and processes them accordingly:

```rust
async fn loop_iteration(
    &mut self,
    storage: &mut Connection<'_, Verifier>,
) -> Result<(), MessageProcessorError> {
    // ...
    
    let messages = self
        .indexer
        .process_blocks(self.last_processed_bitcoin_block + 1, to_block)
        .await?;

    for processor in self.message_processors.iter_mut() {
        processor
            .process_messages(storage, messages.clone(), &mut self.indexer)
            .await?;
    }
    
    // ...
}
```

### Verifier Network

The Verifier Network coordinates the signing of withdrawal transactions:

```rust
async fn loop_iteration(&mut self) -> Result<(), anyhow::Error> {
    // ...
    
    if self.is_coordinator() {
        self.create_new_session().await?;
        // ...
        
        if self
            .build_and_broadcast_final_transaction(&session_info, &session_op)
            .await?
        {
            return Ok(());
        }
    }
    
    // ...
}
```

### Sequencer/State Keeper

The Sequencer processes L1→L2 messages (deposits) as priority operations in the L2 system. These operations are inserted into the database by the L1ToL2MessageProcessor and then picked up by the Sequencer for inclusion in L2 blocks.

## Configuration Parameters

The bridge component uses several configuration parameters:

- **Bridge Address**: The Bitcoin address of the bridge (MuSig2 multi-signature address)
- **Verifier Public Keys**: Public keys of the verifiers participating in the MuSig2 scheme
- **Confirmation Threshold**: Number of Bitcoin confirmations required for deposits
- **Agreement Threshold**: Percentage of verifiers required to agree on a withdrawal
- **Poll Interval**: How frequently to check for new blocks and transactions

## Conclusion

The Via L2 Bitcoin Bridge provides a secure and efficient mechanism for transferring assets between Bitcoin (L1) and the Via L2 rollup. It leverages Bitcoin's native capabilities along with advanced cryptographic techniques like ZK proofs and MuSig2 multi-signatures to ensure the security and integrity of cross-chain transfers.

The bridge's design emphasizes:
- **Security**: Through multi-signatures, ZK proofs, and address validation
- **Decentralization**: By distributing control among multiple verifiers
- **Efficiency**: By batching withdrawals and optimizing transaction fees
- **Transparency**: By using Bitcoin's public ledger for all transfers
- **Safety**: By validating L2 receiver addresses and handling invalid deposits gracefully

The recent refactoring of the deposit logic has improved the system's robustness by:
1. Centralizing deposit validation and processing in a dedicated structure
2. Adding explicit L2 receiver address validation
3. Standardizing gas parameters for deposit transactions
4. Improving error handling for invalid deposits