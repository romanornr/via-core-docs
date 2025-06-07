# Via L2 Bitcoin Bridge Documentation

## Overview

The Via L2 Bitcoin Bridge is a critical component of the Via L2 system, enabling the secure transfer of assets between the L1 (Bitcoin) blockchain and the L2 (Via) rollup. The bridge implements a two-way mechanism for:

1. **Deposits (L1→L2)**: Detecting Bitcoin transactions sent to the bridge address and crediting corresponding assets on L2.
2. **Withdrawals (L2→L1)**: Processing withdrawal requests initiated on L2 and executing the corresponding Bitcoin transactions.

The bridge leverages Bitcoin's native capabilities along with ZK proofs and MuSig2 multi-signature schemes to ensure security, decentralization, and efficiency.

## Key Features

The bridge system provides comprehensive functionality for secure and efficient cross-chain operations:

### Core Capabilities
- **Advanced L1 Deposit Validation**: Comprehensive validation checks for L2 receiver addresses and minimum deposit amounts
- **Streamlined L1 Transaction Creation**: Optimized transaction structure with standardized calldata handling for deposits
- **Multi-Client Bitcoin Integration**: Support for multiple Bitcoin client instances with advanced resource management
- **Comprehensive Testnet Support**: Full testnet configuration and bootstrapping capabilities
- **Advanced Wallet Integration**: Wallet-specific Bitcoin RPC URL support for optimized bridge operations
- **Enhanced Database Operations**: Advanced L1 batch details queries for comprehensive bridge transaction tracking

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

The deposit validation system implements comprehensive security measures and advanced processing logic:

#### 1. Enhanced L2 Receiver Address Validation

The system implements comprehensive L2 receiver address validation through the `ViaL1Deposit::is_valid_deposit()` function:

```rust
// Enhanced validation with stricter checks
impl ViaL1Deposit {
    pub fn is_valid_deposit(&self) -> bool {
        // Check minimum valid L2 receiver address
        const MIN_VALID_L2_RECEIVER_ADDRESS: H160 = H160([
            0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
            0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
            0x00, 0x01, 0x00, 0x01
        ]);
        
        // Validate L2 receiver address is above minimum threshold
        if self.l2_receiver_address < MIN_VALID_L2_RECEIVER_ADDRESS {
            return false;
        }
        
        // Additional validation checks
        self.validate_minimum_deposit_amount() && 
        self.validate_address_format()
    }
    
    fn validate_minimum_deposit_amount(&self) -> bool {
        // Minimum deposit amount validation (e.g., 1000 satoshis)
        const MIN_DEPOSIT_AMOUNT: u64 = 1000;
        self.amount >= MIN_DEPOSIT_AMOUNT
    }
    
    fn validate_address_format(&self) -> bool {
        // Ensure address is not zero address or other reserved addresses
        !self.l2_receiver_address.is_zero()
    }
}
```

**Validation Requirements:**
- L2 receiver address must be >= `0x0000000000000000000000000000000000010001`
- Minimum deposit amount must be met (prevents dust attacks)
- Address format validation (no zero addresses or reserved addresses)
- Enhanced security checks for system-reserved address ranges

#### 2. Improved Deposit Encapsulation

Deposits are encapsulated in a comprehensive `ViaL1Deposit` struct with validation fields:

```rust
pub struct ViaL1Deposit {
    pub l2_receiver_address: Address,
    pub amount: u64,
    pub calldata: Vec<u8>,           // Now always empty for deposits
    pub serial_id: PriorityOpId,
    pub l1_block_number: u64,
    pub validation_status: DepositValidationStatus,
    pub created_at: u64,             // Timestamp for tracking
}

#[derive(Debug, Clone, PartialEq)]
pub enum DepositValidationStatus {
    Pending,
    Valid,
    Invalid(String),  // Reason for invalidity
}
```

#### 3. Simplified L1 Transaction Creation

L1 deposit transactions use **empty calldata** by default, providing a streamlined transaction structure:

```rust
fn create_l1_tx_from_deposit(
    &self,
    deposit: &ViaL1Deposit,
) -> Result<L1ToL2Transaction, DepositProcessingError> {
    // Calldata is now always empty for deposits
    let calldata = Vec::new();  // Simplified - no longer uses deposit.calldata
    
    let mantissa = U256::from(10_000_000_000u64); // BTC to ETH decimal conversion
    let value = U256::from(deposit.amount) * mantissa;
    
    let l1_tx = L1Tx {
        execute: Execute {
            contract_address: deposit.l2_receiver_address,
            calldata,  // Always empty for deposits
            value: U256::zero(),
            factory_deps: vec![],
        },
        common_data: L1TxCommonData {
            sender: deposit.l2_receiver_address,
            serial_id: deposit.serial_id,
            to_mint: value,
            refund_recipient: deposit.l2_receiver_address,
            eth_block: deposit.l1_block_number,
            // Standardized gas parameters
            max_fee_per_gas: U256::from(120_000_000u64),
            gas_limit: U256::from(300_000u64),
            gas_per_pubdata_limit: REQUIRED_L1_TO_L2_GAS_PER_PUBDATA_BYTE.into(),
        },
        received_timestamp_ms: unix_timestamp_ms(),
    };
    
    Ok(L1ToL2Transaction { l1_tx, deposit_info: deposit.clone() })
}
```

**Key Features:**
- **Empty Calldata**: All deposit transactions use empty calldata for consistency
- **Streamlined Structure**: Optimized transaction creation process
- **Standardized Parameters**: Fixed gas parameters for predictable behavior

#### 4. Advanced Invalid Deposit Handling

```rust
impl DepositProcessor {
    pub async fn process_deposits(&mut self, deposits: Vec<ViaL1Deposit>) -> ProcessingResult {
        let mut valid_deposits = Vec::new();
        let mut invalid_deposits = Vec::new();
        
        for mut deposit in deposits {
            if deposit.is_valid_deposit() {
                deposit.validation_status = DepositValidationStatus::Valid;
                valid_deposits.push(deposit);
            } else {
                let reason = self.get_validation_failure_reason(&deposit);
                deposit.validation_status = DepositValidationStatus::Invalid(reason.clone());
                invalid_deposits.push(deposit);
                
                // Comprehensive logging for invalid deposits
                tracing::warn!(
                    "Invalid deposit detected: txid={}, reason={}, amount={}, receiver={}",
                    deposit.txid,
                    reason,
                    deposit.amount,
                    deposit.l2_receiver_address
                );
            }
        }
        
        // Process valid deposits
        self.create_l1_transactions(valid_deposits).await?;
        
        // Store invalid deposits for audit trail
        self.store_invalid_deposits(invalid_deposits).await?;
        
        Ok(ProcessingResult::success())
    }
}
```

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

## Bitcoin Client Resource Layer Integration

The bridge now supports enhanced Bitcoin client resource management with multiple client instances for improved reliability and performance:

### Multiple Bitcoin Client Support

```rust
pub struct BridgeBitcoinClientManager {
    sender_client: Arc<BitcoinClient>,
    verifier_client: Arc<BitcoinClient>,
    bridge_client: Arc<BitcoinClient>,
    resource_pool: BitcoinClientResourcePool,
}

impl BridgeBitcoinClientManager {
    pub fn new(config: &BridgeConfig) -> Result<Self, BridgeError> {
        Ok(Self {
            sender_client: Arc::new(BitcoinClient::new(&config.sender_rpc_url)?),
            verifier_client: Arc::new(BitcoinClient::new(&config.verifier_rpc_url)?),
            bridge_client: Arc::new(BitcoinClient::new(&config.bridge_rpc_url)?),
            resource_pool: BitcoinClientResourcePool::new(config.max_connections),
        })
    }
    
    pub async fn get_client_for_operation(&self, operation: BridgeOperation) -> Arc<BitcoinClient> {
        match operation {
            BridgeOperation::SendTransaction => self.sender_client.clone(),
            BridgeOperation::VerifyTransaction => self.verifier_client.clone(),
            BridgeOperation::BridgeMonitoring => self.bridge_client.clone(),
        }
    }
}
```

### Enhanced Resource Management

```rust
pub struct BitcoinClientResourcePool {
    connections: Arc<Mutex<Vec<BitcoinClientConnection>>>,
    max_connections: usize,
    active_connections: AtomicUsize,
}

impl BitcoinClientResourcePool {
    pub async fn acquire_connection(&self) -> Result<BitcoinClientConnection, ResourceError> {
        let mut connections = self.connections.lock().await;
        
        if let Some(connection) = connections.pop() {
            Ok(connection)
        } else if self.active_connections.load(Ordering::Relaxed) < self.max_connections {
            self.create_new_connection().await
        } else {
            Err(ResourceError::NoAvailableConnections)
        }
    }
    
    pub async fn release_connection(&self, connection: BitcoinClientConnection) {
        let mut connections = self.connections.lock().await;
        connections.push(connection);
    }
}
```

### Separation of Concerns

The bridge now clearly separates different operational roles:

- **Sender Role**: Handles transaction broadcasting and fee management
- **Verifier Role**: Manages transaction verification and validation
- **Bridge Role**: Monitors deposits and withdrawal requests

## Testnet Bootstrapping Support

Enhanced testnet support with dedicated bootstrapping configuration:

### Testnet Configuration Structure

```
etc/env/via/genesis/testnet/
├── bootstrap_transactions.json
├── verifier_keys.json
├── bridge_config.json
└── network_params.json
```

### Bootstrap Transaction IDs

```json
{
  "testnet_bootstrap_transactions": {
    "genesis_tx": "a1b2c3d4e5f6789012345678901234567890abcdef1234567890abcdef123456",
    "bridge_init_tx": "b2c3d4e5f6789012345678901234567890abcdef1234567890abcdef1234567a",
    "verifier_setup_tx": "c3d4e5f6789012345678901234567890abcdef1234567890abcdef1234567ab2"
  },
  "confirmation_requirements": {
    "min_confirmations": 1,
    "bootstrap_confirmations": 3
  }
}
```

### Testnet Bridge Configuration

```rust
pub struct TestnetBridgeConfig {
    pub network: Network,
    pub bootstrap_txids: Vec<Txid>,
    pub min_confirmations: u32,
    pub bridge_address: Address,
    pub verifier_pubkeys: Vec<PublicKey>,
    pub development_mode: bool,
}

impl TestnetBridgeConfig {
    pub fn load_from_file(path: &Path) -> Result<Self, ConfigError> {
        let config_data = std::fs::read_to_string(path)?;
        let config: TestnetBridgeConfig = serde_json::from_str(&config_data)?;
        
        // Validate testnet-specific requirements
        if config.network != Network::Testnet {
            return Err(ConfigError::InvalidNetwork);
        }
        
        Ok(config)
    }
    
    pub fn is_bootstrap_complete(&self, current_height: u64) -> bool {
        // Check if all bootstrap transactions are confirmed
        self.bootstrap_txids.iter().all(|txid| {
            self.get_transaction_confirmations(txid).unwrap_or(0) >= self.min_confirmations
        })
    }
}
```

## Enhanced Wallet Integration

The bridge now supports wallet-specific Bitcoin RPC URLs and improved wallet management:

### Wallet-Specific Configuration

```rust
pub struct WalletBridgeConfig {
    pub wallet_name: String,
    pub bitcoin_rpc_url: String,
    pub wallet_rpc_url: String,  // Wallet-specific RPC endpoint
    pub utxo_management: UtxoManagementConfig,
    pub fee_strategy: FeeStrategy,
}

impl WalletBridgeConfig {
    pub fn new(wallet_name: String, base_rpc_url: String) -> Self {
        Self {
            wallet_rpc_url: format!("{}/wallet/{}", base_rpc_url, wallet_name),
            wallet_name,
            bitcoin_rpc_url: base_rpc_url,
            utxo_management: UtxoManagementConfig::default(),
            fee_strategy: FeeStrategy::Conservative,
        }
    }
}
```

### Enhanced UTXO Handling

```rust
pub struct WalletUtxoManager {
    wallet_config: WalletBridgeConfig,
    utxo_cache: Arc<Mutex<HashMap<OutPoint, TxOut>>>,
    reserved_utxos: Arc<Mutex<HashSet<OutPoint>>>,
}

impl WalletUtxoManager {
    pub async fn get_available_utxos(&self) -> Result<Vec<Utxo>, WalletError> {
        let client = BitcoinClient::new(&self.wallet_config.wallet_rpc_url)?;
        let utxos = client.list_unspent(None, None, None, None, None).await?;
        
        // Filter out reserved UTXOs
        let reserved = self.reserved_utxos.lock().await;
        let available_utxos: Vec<Utxo> = utxos
            .into_iter()
            .filter(|utxo| !reserved.contains(&utxo.outpoint))
            .collect();
            
        Ok(available_utxos)
    }
    
    pub async fn reserve_utxos(&self, utxos: Vec<OutPoint>) -> Result<(), WalletError> {
        let mut reserved = self.reserved_utxos.lock().await;
        for utxo in utxos {
            reserved.insert(utxo);
        }
        Ok(())
    }
}
```

## Database Query Improvements

Enhanced L1 batch details queries for better bridge transaction tracking:

### Improved Batch Queries

```rust
impl ViaVotesDal<'_, '_> {
    pub async fn get_l1_batch_details_for_bridge(
        &mut self,
        batch_number: L1BatchNumber,
    ) -> DalResult<Option<L1BatchDetailsForBridge>> {
        let query = sqlx::query_as!(
            L1BatchDetailsForBridge,
            r#"
            SELECT 
                l1_batches.number,
                l1_batches.timestamp,
                l1_batches.l1_tx_count,
                l1_batches.l2_tx_count,
                l1_batches.hash,
                l1_batches.commitment,
                l1_batches.compressed_write_logs,
                l1_batches.compressed_contracts,
                l1_batches.eth_prove_tx_id,
                l1_batches.eth_commit_tx_id,
                l1_batches.eth_execute_tx_id,
                bridge_operations.deposit_count,
                bridge_operations.withdrawal_count,
                bridge_operations.total_deposit_amount,
                bridge_operations.total_withdrawal_amount
            FROM l1_batches
            LEFT JOIN bridge_operations ON l1_batches.number = bridge_operations.l1_batch_number
            WHERE l1_batches.number = $1
            "#,
            batch_number.0 as i64
        );
        
        let result = query.fetch_optional(self.storage.conn()).await?;
        Ok(result)
    }
    
    pub async fn list_finalized_blocks_and_non_processed_withdrawals_enhanced(
        &mut self,
    ) -> DalResult<Vec<L1BatchWithBridgeOperations>> {
        let query = sqlx::query_as!(
            L1BatchWithBridgeOperations,
            r#"
            SELECT 
                l1_batches.*,
                bridge_operations.withdrawal_count,
                bridge_operations.pending_withdrawals,
                bridge_operations.processed_withdrawals
            FROM l1_batches
            INNER JOIN bridge_operations ON l1_batches.number = bridge_operations.l1_batch_number
            WHERE l1_batches.is_finished = true
            AND bridge_operations.withdrawal_count > bridge_operations.processed_withdrawals
            ORDER BY l1_batches.number ASC
            "#
        );
        
        let batches = query.fetch_all(self.storage.conn()).await?;
        Ok(batches)
    }
}
```

### Enhanced Data Availability

```rust
pub struct BridgeTransactionTracker {
    pub batch_number: L1BatchNumber,
    pub deposit_transactions: Vec<DepositTransaction>,
    pub withdrawal_transactions: Vec<WithdrawalTransaction>,
    pub processing_status: BatchProcessingStatus,
    pub confirmation_count: u32,
}

impl BridgeTransactionTracker {
    pub async fn track_batch_operations(
        &self,
        dal: &mut ViaVotesDal<'_, '_>,
    ) -> Result<BatchOperationSummary, TrackingError> {
        let batch_details = dal
            .get_l1_batch_details_for_bridge(self.batch_number)
            .await?
            .ok_or(TrackingError::BatchNotFound)?;
            
        let operation_summary = BatchOperationSummary {
            batch_number: self.batch_number,
            total_deposits: batch_details.deposit_count,
            total_withdrawals: batch_details.withdrawal_count,
            deposit_amount: batch_details.total_deposit_amount,
            withdrawal_amount: batch_details.total_withdrawal_amount,
            processing_status: self.processing_status.clone(),
        };
        
        Ok(operation_summary)
    }
}
```

## Configuration Examples and Best Practices

### Complete Bridge Configuration

```toml
[bridge]
# Network configuration
network = "testnet"  # or "mainnet"
confirmation_threshold = 6

# Bitcoin client configuration
[bridge.bitcoin_clients]
sender_rpc_url = "http://localhost:18332"
verifier_rpc_url = "http://localhost:18333"
bridge_rpc_url = "http://localhost:18334"
max_connections = 10

# Wallet configuration
[bridge.wallet]
wallet_name = "via_bridge_wallet"
wallet_rpc_url = "http://localhost:18332/wallet/via_bridge_wallet"
utxo_selection_strategy = "conservative"
fee_rate_strategy = "medium"

# Bridge address and keys
[bridge.security]
bridge_address = "tb1qw508d6qejxtdg4y5r3zarvary0c5xw7kxpjzsx"
verifier_threshold = 3
total_verifiers = 5

# Deposit validation
[bridge.deposits]
min_deposit_amount = 1000  # satoshis
min_l2_receiver_address = "0x0000000000000000000000000000000000010001"
max_calldata_size = 0  # Empty calldata for deposits

# Withdrawal configuration
[bridge.withdrawals]
batch_size = 10
max_fee_rate = 50  # sat/vB
min_output_amount = 546  # dust limit

# Database configuration
[bridge.database]
connection_pool_size = 20
query_timeout_seconds = 30
batch_query_limit = 100

# Testnet bootstrapping
[bridge.testnet]
bootstrap_transactions = [
    "a1b2c3d4e5f6789012345678901234567890abcdef1234567890abcdef123456",
    "b2c3d4e5f6789012345678901234567890abcdef1234567890abcdef1234567a"
]
development_mode = true
fast_confirmations = true
```

### Production Configuration Example

```toml
[bridge]
network = "mainnet"
confirmation_threshold = 6

[bridge.bitcoin_clients]
sender_rpc_url = "https://bitcoin-rpc-sender.example.com"
verifier_rpc_url = "https://bitcoin-rpc-verifier.example.com"
bridge_rpc_url = "https://bitcoin-rpc-bridge.example.com"
max_connections = 50

[bridge.security]
bridge_address = "bc1qw508d6qejxtdg4y5r3zarvary0c5xw7kw508d6qejxtdg4y5r3zarvary0c5xw7kw5rljs90"
verifier_threshold = 7
total_verifiers = 10

[bridge.deposits]
min_deposit_amount = 10000  # 0.0001 BTC minimum
strict_validation = true

[bridge.withdrawals]
batch_size = 50
max_fee_rate = 20
security_delay_blocks = 144  # 24 hours
```

## Operational Guidelines

### Bridge Deployment Process

1. **Pre-deployment Checklist**
   ```bash
   # Verify Bitcoin client connectivity
   curl -X POST -H "Content-Type: application/json" \
     -d '{"jsonrpc":"1.0","id":"test","method":"getblockchaininfo","params":[]}' \
     http://localhost:18332
   
   # Validate bridge configuration
   via-bridge validate-config --config bridge.toml
   
   # Test wallet connectivity
   via-bridge test-wallet --wallet-name via_bridge_wallet
   ```

2. **Bootstrap Sequence**
   ```bash
   # Initialize bridge with testnet configuration
   via-bridge init --network testnet --config bridge.toml
   
   # Generate verifier keys
   via-bridge generate-keys --count 5 --threshold 3
   
   # Deploy bridge address
   via-bridge deploy-address --keys verifier_keys.json
```

3. **Monitoring Setup**
   ```bash
   # Start bridge monitoring
   via-bridge monitor --config bridge.toml --log-level info
   
   # Enable metrics collection
   via-bridge metrics --port 9090 --enable-prometheus
   ```

### Best Practices for Bridge Operations

#### Security Best Practices

1. **Multi-Client Setup**: Always use separate Bitcoin RPC endpoints for different operations
2. **Key Management**: Store verifier keys in secure hardware modules
3. **Monitoring**: Implement comprehensive monitoring for all bridge operations
4. **Backup Strategy**: Maintain regular backups of wallet and configuration data

#### Performance Optimization

1. **Connection Pooling**: Configure appropriate connection pool sizes based on load
2. **Batch Processing**: Process deposits and withdrawals in batches for efficiency
3. **UTXO Management**: Implement proper UTXO selection and consolidation strategies
4. **Fee Management**: Use dynamic fee estimation for optimal transaction costs

#### Operational Monitoring

```rust
pub struct BridgeMonitor {
    pub deposit_rate: f64,
    pub withdrawal_rate: f64,
    pub average_confirmation_time: Duration,
    pub failed_transactions: u64,
    pub wallet_balance: u64,
}

impl BridgeMonitor {
    pub async fn collect_metrics(&mut self) -> Result<BridgeMetrics, MonitoringError> {
        let metrics = BridgeMetrics {
            deposits_per_hour: self.calculate_deposit_rate().await?,
            withdrawals_per_hour: self.calculate_withdrawal_rate().await?,
            average_fee_rate: self.get_average_fee_rate().await?,
            utxo_count: self.get_utxo_count().await?,
            pending_operations: self.get_pending_operations().await?,
        };
        
        Ok(metrics)
    }
}
```

## Troubleshooting Common Bridge Issues

### Deposit Issues

#### Issue: Deposits Not Being Detected
```bash
# Check Bitcoin client connectivity
via-bridge check-btc-client --endpoint http://localhost:18332

# Verify bridge address monitoring
via-bridge check-address --address tb1qw508d6qejxtdg4y5r3zarvary0c5xw7kxpjzsx

# Check indexer status
via-bridge indexer-status --last-block
```

**Solution Steps:**
1. Verify Bitcoin client is synced and accessible
2. Check bridge address configuration
3. Ensure indexer is processing blocks correctly
4. Validate deposit transaction format

#### Issue: Invalid Deposit Validation
```bash
# Check deposit validation logs
tail -f /var/log/via-bridge/deposits.log | grep "Invalid deposit"

# Validate specific deposit
via-bridge validate-deposit --txid <transaction_id>
```

**Common Causes:**
- L2 receiver address below minimum threshold
- Deposit amount below minimum requirement
- Invalid transaction format

### Withdrawal Issues

#### Issue: Withdrawal Transactions Not Being Signed
```bash
# Check verifier connectivity
via-bridge check-verifiers --config bridge.toml

# Verify MuSig2 session status
via-bridge musig2-status --session-id <session_id>
```

**Solution Steps:**
1. Ensure sufficient verifiers are online
2. Check verifier key configuration
3. Verify MuSig2 session coordination
4. Check withdrawal transaction format

#### Issue: High Transaction Fees
```bash
# Check current fee rates
via-bridge fee-estimate --target-blocks 6

# Optimize UTXO selection
via-bridge optimize-utxos --wallet via_bridge_wallet
```

**Optimization Strategies:**
- Implement UTXO consolidation during low-fee periods
- Use dynamic fee estimation
- Batch multiple withdrawals when possible

### Configuration Issues

#### Issue: Bitcoin Client Connection Failures
```bash
# Test all configured endpoints
via-bridge test-endpoints --config bridge.toml

# Check authentication
via-bridge test-auth --rpc-url http://localhost:18332
```

**Common Solutions:**
- Verify RPC credentials and permissions
- Check network connectivity and firewall rules
- Ensure Bitcoin client is running and synced

#### Issue: Database Query Timeouts
```bash
# Check database connectivity
via-bridge test-db --connection-string postgresql://...

# Analyze slow queries
via-bridge analyze-queries --slow-threshold 5s
```

**Performance Improvements:**
- Increase connection pool size
- Optimize database indexes
- Implement query result caching

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

## System Architecture Summary

The bridge system implements comprehensive functionality designed for security and reliability:

### Deposit Processing Features
- **Advanced Validation**: Comprehensive L2 receiver address validation with minimum thresholds
- **Standardized Calldata**: All deposit transactions use empty calldata for consistency
- **Minimum Amount Enforcement**: Deposits below minimum threshold are rejected
- **Comprehensive Error Handling**: Advanced logging and tracking of invalid deposits

### Transaction Creation Features
- **Streamlined Structure**: Optimized L1 transaction creation process
- **Standardized Parameters**: Fixed gas parameters for predictable behavior
- **High Reliability**: Comprehensive error handling and validation throughout the process

### Resource Management Features
- **Multi-Client Support**: Separate Bitcoin clients for different operations
- **Advanced Connection Pooling**: Optimized resource utilization and reliability
- **Wallet-Specific Configuration**: Support for wallet-specific RPC endpoints

## Conclusion

The Via L2 Bitcoin Bridge provides a secure and efficient mechanism for transferring assets between Bitcoin (L1) and the Via L2 rollup. It leverages Bitcoin's native capabilities along with advanced cryptographic techniques like ZK proofs and MuSig2 multi-signatures to ensure the security and integrity of cross-chain transfers.

The bridge's comprehensive design emphasizes:
- **Security**: Through advanced multi-signatures, ZK proofs, and comprehensive address validation
- **Decentralization**: By distributing control among multiple verifiers with sophisticated coordination
- **Efficiency**: By batching withdrawals, optimizing transaction fees, and streamlined processing
- **Transparency**: By using Bitcoin's public ledger for all transfers with comprehensive tracking
- **Reliability**: Through multiple Bitcoin client support and advanced resource management
- **Safety**: By validating L2 receiver addresses with comprehensive requirements and handling invalid deposits gracefully

The system's comprehensive architecture provides robust functionality through:

1. **Advanced Deposit Validation**: Comprehensive L2 receiver address validation and minimum deposit amount requirements
2. **Streamlined Transaction Creation**: Standardized L1 deposit transactions with optimized calldata handling
3. **Multi-Client Bitcoin Integration**: Multiple Bitcoin client instances for optimal reliability and separation of concerns
4. **Comprehensive Testnet Support**: Full testnet bootstrapping configuration and support
5. **Advanced Wallet Integration**: Wallet-specific Bitcoin RPC URL support and comprehensive UTXO management
6. **Enhanced Database Operations**: Advanced L1 batch details queries for comprehensive bridge transaction tracking and data availability
7. **Flexible Configuration**: Comprehensive configuration options for different deployment scenarios
8. **Operational Excellence**: Detailed operational guidelines, troubleshooting procedures, and best practices

These features ensure the bridge operates with optimal security, reliability, and operational efficiency while maintaining the core principles of decentralization and transparency that are fundamental to the Via L2 system.