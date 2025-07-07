# Via L2 Bitcoin L1 Watchtower / Listener / Indexer

## Overview

The L1 Watchtower component in Via L2 is responsible for monitoring the Bitcoin blockchain for relevant events and transactions that affect the Layer 2 system. It serves as the bridge between the Bitcoin L1 chain and the Via L2 rollup, ensuring that deposits, system messages, and other critical information are properly indexed and processed.

## Architecture

The L1 Watchtower consists of several key components:

1. **Bitcoin Client** - Connects to a Bitcoin node via RPC to fetch blocks and transactions
2. **Inscription Indexer** - Parses Bitcoin transactions to identify and extract Via protocol messages
3. **Message Processors** - Process different types of messages (L1-to-L2 transfers, verifier attestations)
4. **Polling Mechanism** - Regularly checks for new blocks and processes them

### Component Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                     L1 Watchtower                           │
│                                                             │
│  ┌───────────────┐    ┌───────────────┐    ┌──────────────┐ │
│  │ Bitcoin Client│───▶│Inscription    │───▶│Message       │ │
│  │ (RPC)         │    │Indexer        │    │Processors    │ │
│  └───────────────┘    └───────────────┘    └──────────────┘ │
│          ▲                                        │         │
│          │                                        ▼         │
│  ┌───────────────┐                       ┌──────────────┐   │
│  │ Bitcoin Node  │                       │ Database     │   │
│  └───────────────┘                       └──────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. Bitcoin Client (`via_btc_client`)

Located in `core/lib/via_btc_client/src/client/`, this component is responsible for connecting to and interacting with a Bitcoin node.

Key files:
- `mod.rs` - Defines the `BitcoinClient` struct and implements the `BitcoinOps` trait
- `rpc_client.rs` - Implements the RPC client for communicating with the Bitcoin node

Key functionalities:
- Connecting to a Bitcoin node via RPC
- Fetching blocks, transactions, and UTXOs
- Broadcasting signed transactions
- Checking transaction confirmations
- Estimating fee rates

Example of client initialization:
```rust
let client = BitcoinClient::new(rpc_url, network, auth)?;
```

### 2. Inscription Indexer (`BitcoinInscriptionIndexer`)

Located in `core/lib/via_btc_client/src/indexer/`, this component is responsible for parsing Bitcoin transactions to identify and extract Via protocol messages.

Key files:
- `mod.rs` - Defines the `BitcoinInscriptionIndexer` struct and its methods
- `parser.rs` - Implements the `MessageParser` for extracting messages from transactions

Key functionalities:
- Bootstrapping the indexer with initial system transactions
- Processing blocks to extract Via protocol messages
- Validating system messages and bridge messages
- Handling chain reorganizations

The indexer processes several types of messages:
- `SystemBootstrapping` - Initial system setup
- `ProposeSequencer` - Proposing a new sequencer
- `ValidatorAttestation` - Verifier votes on proposals or proofs
- `L1BatchDAReference` - References to L1 batch data
- `ProofDAReference` - References to proof data
- `L1ToL2Message` - Deposits from L1 to L2

Example of indexer initialization:
```rust
let indexer = BitcoinInscriptionIndexer::new(rpc_url, network, auth, bootstrap_txids).await?;
```

### 3. Message Processors

Located in `via_verifier/node/via_btc_watch/src/message_processors/`, these components process different types of messages extracted by the indexer.

Key files:
- `mod.rs` - Defines the `MessageProcessor` trait
- `l1_to_l2.rs` - Processes L1-to-L2 transfers (deposits)
- `verifier.rs` - Processes verifier attestations and proof references

Key functionalities:
- Processing L1-to-L2 transfers (deposits)
- Processing verifier attestations
- Finalizing transactions based on verifier votes
- Storing processed messages in the database

### 4. BTC Watch (`VerifierBtcWatch`)

Located in `via_verifier/node/via_btc_watch/src/lib.rs`, this component orchestrates the polling and processing of Bitcoin blocks.

Key functionalities:
- Initializing the indexer and message processors
- Polling for new blocks at regular intervals
- Processing blocks to extract and handle messages
- Handling errors and retries
- Tracking metrics

## L1 Indexer Integration

The L1 Watchtower now integrates with the dedicated L1 Indexer service for enhanced Bitcoin blockchain monitoring and data management.

### Architecture Integration

The L1 Indexer operates as a complementary service to the traditional L1 Watchtower, providing specialized indexing capabilities:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Enhanced L1 Monitoring                             │
│                                                                             │
│  ┌─────────────────┐              ┌─────────────────┐                      │
│  │   L1 Watcher    │              │   L1 Indexer    │                      │
│  │   (Traditional) │◄────────────►│   (Dedicated)   │                      │
│  │                 │              │                 │                      │
│  │ • Inscription   │              │ • Deposit       │                      │
│  │   Processing    │              │   Tracking      │                      │
│  │ • Message       │              │ • Withdrawal    │                      │
│  │   Validation    │              │   Monitoring    │                      │
│  │ • Verifier      │              │ • UTXO          │                      │
│  │   Coordination  │              │   Management    │                      │
│  └─────────────────┘              └─────────────────┘                      │
│           │                                │                               │
│           ▼                                ▼                               │
│  ┌─────────────────┐              ┌─────────────────┐                      │
│  │ Main Database   │              │ L1 Indexer DB   │                      │
│  │ • Transactions  │              │ • Deposits      │                      │
│  │ • L1 Batches    │              │ • Withdrawals   │                      │
│  │ • Events        │              │ • UTXOs         │                      │
│  │ • Verifier Data │              │ • Metadata      │                      │
│  └─────────────────┘              └─────────────────┘                      │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Dual-Service Operation

The system now supports two operational modes:

#### 1. Integrated Mode
Both L1 Watcher and L1 Indexer run within the same process:

```rust
pub struct IntegratedL1Monitor {
    watcher: VerifierBtcWatch,
    indexer: L1IndexerService,
    coordination_channel: mpsc::Receiver<IndexerEvent>,
}

impl IntegratedL1Monitor {
    pub async fn new(config: &L1MonitorConfig) -> Result<Self, L1MonitorError> {
        let (tx, rx) = mpsc::channel(1000);
        
        let watcher = VerifierBtcWatch::new(&config.watcher_config).await?;
        let indexer = L1IndexerService::new(&config.indexer_config, tx).await?;
        
        Ok(Self {
            watcher,
            indexer,
            coordination_channel: rx,
        })
    }
    
    pub async fn run(&mut self) -> Result<(), L1MonitorError> {
        // Start both services concurrently
        let watcher_handle = tokio::spawn(async move {
            self.watcher.run().await
        });
        
        let indexer_handle = tokio::spawn(async move {
            self.indexer.run().await
        });
        
        // Coordinate between services
        let coordination_handle = tokio::spawn(async move {
            self.coordinate_services().await
        });
        
        // Wait for all services
        tokio::try_join!(watcher_handle, indexer_handle, coordination_handle)?;
        Ok(())
    }
    
    async fn coordinate_services(&mut self) -> Result<(), L1MonitorError> {
        while let Some(event) = self.coordination_channel.recv().await {
            match event {
                IndexerEvent::DepositDetected(deposit) => {
                    // Notify watcher of new deposit for processing
                    self.watcher.handle_deposit_event(deposit).await?;
                }
                IndexerEvent::WithdrawalProcessed(withdrawal) => {
                    // Update watcher state with withdrawal information
                    self.watcher.handle_withdrawal_event(withdrawal).await?;
                }
                IndexerEvent::BlockProcessed(block_height) => {
                    // Synchronize block processing state
                    self.watcher.sync_block_height(block_height).await?;
                }
            }
        }
        Ok(())
    }
}
```

#### 2. Standalone Mode
L1 Watcher and L1 Indexer run as separate services:

```bash
# Start L1 Indexer as standalone service
via-indexer --config /path/to/l1_indexer.toml &

# Start L1 Watcher with indexer integration
via_server --components btc_watcher --l1-indexer-url http://localhost:8081
```

### Data Synchronization

The L1 Watcher and L1 Indexer maintain synchronized state through several mechanisms:

#### 1. Shared Block Height Tracking

```rust
pub struct BlockHeightSynchronizer {
    watcher_client: Arc<WatcherClient>,
    indexer_client: Arc<IndexerClient>,
    sync_interval: Duration,
}

impl BlockHeightSynchronizer {
    pub async fn synchronize_block_heights(&self) -> Result<(), SyncError> {
        let watcher_height = self.watcher_client.get_last_processed_block().await?;
        let indexer_height = self.indexer_client.get_last_processed_block().await?;
        
        match watcher_height.cmp(&indexer_height) {
            std::cmp::Ordering::Greater => {
                // Watcher is ahead, update indexer
                self.indexer_client.sync_to_block(watcher_height).await?;
            }
            std::cmp::Ordering::Less => {
                // Indexer is ahead, update watcher
                self.watcher_client.sync_to_block(indexer_height).await?;
            }
            std::cmp::Ordering::Equal => {
                // Heights are synchronized
                tracing::debug!("Block heights synchronized at {}", watcher_height);
            }
        }
        
        Ok(())
    }
}
```

#### 2. Cross-Service Event Notification

```rust
#[derive(Debug, Clone)]
pub enum CrossServiceEvent {
    DepositDetected {
        txid: Txid,
        amount: u64,
        l2_receiver: Address,
        block_height: u32,
    },
    WithdrawalInitiated {
        withdrawal_id: u64,
        recipient: String,
        amount: u64,
        l2_batch: u64,
    },
    InscriptionProcessed {
        txid: Txid,
        inscription_data: String,
        message_type: String,
    },
    BlockReorg {
        old_tip: u32,
        new_tip: u32,
        affected_blocks: Vec<u32>,
    },
}

pub struct EventBridge {
    watcher_events: mpsc::Sender<CrossServiceEvent>,
    indexer_events: mpsc::Sender<CrossServiceEvent>,
}

impl EventBridge {
    pub async fn relay_event(&self, event: CrossServiceEvent) -> Result<(), EventBridgeError> {
        // Send event to both services
        self.watcher_events.send(event.clone()).await?;
        self.indexer_events.send(event).await?;
        Ok(())
    }
}
```

### Enhanced Message Processing

The integration enables enhanced message processing capabilities:

#### 1. Deposit Correlation

```rust
pub struct DepositCorrelator {
    watcher_deposits: HashMap<Txid, WatcherDeposit>,
    indexer_deposits: HashMap<Txid, IndexerDeposit>,
}

impl DepositCorrelator {
    pub async fn correlate_deposits(&mut self) -> Result<Vec<CorrelatedDeposit>, CorrelationError> {
        let mut correlated = Vec::new();
        
        for (txid, watcher_deposit) in &self.watcher_deposits {
            if let Some(indexer_deposit) = self.indexer_deposits.get(txid) {
                // Validate consistency between watcher and indexer data
                if self.validate_deposit_consistency(watcher_deposit, indexer_deposit)? {
                    correlated.push(CorrelatedDeposit {
                        txid: *txid,
                        watcher_data: watcher_deposit.clone(),
                        indexer_data: indexer_deposit.clone(),
                        correlation_status: CorrelationStatus::Validated,
                    });
                } else {
                    tracing::warn!("Deposit data inconsistency detected for txid: {}", txid);
                    correlated.push(CorrelatedDeposit {
                        txid: *txid,
                        watcher_data: watcher_deposit.clone(),
                        indexer_data: indexer_deposit.clone(),
                        correlation_status: CorrelationStatus::Inconsistent,
                    });
                }
            }
        }
        
        Ok(correlated)
    }
}
```

#### 2. Withdrawal Validation

```rust
pub struct WithdrawalValidator {
    watcher_client: Arc<WatcherClient>,
    indexer_client: Arc<IndexerClient>,
}

impl WithdrawalValidator {
    pub async fn validate_withdrawal(
        &self,
        withdrawal_id: u64,
    ) -> Result<WithdrawalValidationResult, ValidationError> {
        // Get withdrawal data from both services
        let watcher_withdrawal = self.watcher_client.get_withdrawal(withdrawal_id).await?;
        let indexer_withdrawal = self.indexer_client.get_withdrawal(withdrawal_id).await?;
        
        // Cross-validate withdrawal data
        let validation_checks = vec![
            self.validate_withdrawal_amount(&watcher_withdrawal, &indexer_withdrawal),
            self.validate_recipient_address(&watcher_withdrawal, &indexer_withdrawal),
            self.validate_transaction_structure(&watcher_withdrawal, &indexer_withdrawal),
            self.validate_signature_data(&watcher_withdrawal, &indexer_withdrawal),
        ];
        
        let all_valid = validation_checks.iter().all(|&result| result);
        
        Ok(WithdrawalValidationResult {
            withdrawal_id,
            is_valid: all_valid,
            validation_details: validation_checks,
            watcher_data: watcher_withdrawal,
            indexer_data: indexer_withdrawal,
        })
    }
}
```

### Configuration Integration

The integrated system supports unified configuration:

```toml
# etc/env/base/via_l1_monitor.toml

[l1_monitor]
mode = "integrated"  # "integrated" or "standalone"
coordination_enabled = true
sync_interval = 30   # seconds

[watcher]
poll_interval = 10
confirmations_for_btc_msg = 6
l1_blocks_chunk = 100
restart_indexing = false

[indexer]
database_url = "postgresql://user:password@localhost/via_l1_indexer"
batch_size = 100
polling_interval = 10
confirmation_blocks = 6

[synchronization]
block_height_sync_interval = 60
event_bridge_buffer_size = 1000
correlation_check_interval = 300
```

### Monitoring and Metrics

Enhanced monitoring capabilities for the integrated system:

```rust
pub struct L1MonitorMetrics {
    // Watcher metrics
    pub watcher_blocks_processed: Counter,
    pub watcher_messages_processed: Counter,
    pub watcher_processing_time: Histogram,
    
    // Indexer metrics
    pub indexer_blocks_processed: Counter,
    pub indexer_deposits_found: Counter,
    pub indexer_withdrawals_processed: Counter,
    
    // Integration metrics
    pub sync_operations: Counter,
    pub correlation_checks: Counter,
    pub cross_service_events: Counter,
    pub validation_failures: Counter,
}

impl L1MonitorMetrics {
    pub fn record_sync_operation(&self, operation_type: &str, duration: Duration) {
        self.sync_operations.inc();
        // Record operation-specific metrics
    }
    
    pub fn record_correlation_result(&self, result: CorrelationStatus) {
        self.correlation_checks.inc();
        match result {
            CorrelationStatus::Validated => {
                // Record successful correlation
            }
            CorrelationStatus::Inconsistent => {
                self.validation_failures.inc();
            }
        }
    }
}
```

### Troubleshooting Integration Issues

Common integration issues and solutions:

#### 1. Block Height Desynchronization

```bash
# Check block heights
curl http://localhost:8080/watcher/status | jq '.last_processed_block'
curl http://localhost:8081/indexer/status | jq '.last_processed_block'

# Force synchronization
via-l1-monitor sync-block-heights --force
```

#### 2. Event Bridge Failures

```bash
# Check event bridge status
via-l1-monitor check-event-bridge

# Restart event bridge
via-l1-monitor restart-event-bridge
```

#### 3. Data Correlation Issues

```bash
# Run correlation check
via-l1-monitor correlate-deposits --from-block 100000 --to-block 100100

# Validate withdrawal data
via-l1-monitor validate-withdrawals --withdrawal-id 12345
```

This integration provides a robust, scalable solution for Bitcoin L1 monitoring while maintaining backward compatibility with existing L1 Watcher functionality.


## Scanning/Indexing Strategy

The L1 Watchtower uses a polling-based approach to monitor the Bitcoin blockchain:

1. During initialization:
   - If no previous state exists or `restart_indexing` is true, the system sets `last_processed_bitcoin_block` to `start_l1_block_number - 1`
   - This ensures that processing will begin exactly from the block specified by `start_l1_block_number`

2. The `VerifierBtcWatch` component runs a loop that polls for new blocks at regular intervals (configured by `poll_interval`).
3. For each iteration, it:
   - Fetches the current Bitcoin block height
   - Determines the range of blocks to process (from last processed block + 1 to current height - confirmations)
   - Processes blocks in chunks (defined by `L1_BLOCKS_CHUNK`) up to the determined `to_block`
   - Calls the indexer to process this range of blocks
   - Passes the extracted messages to each message processor
   - Updates the last processed block number persistently in the `via_indexer_metadata` database table

```rust
async fn loop_iteration(&mut self, storage: &mut Connection<'_, Verifier>) -> Result<(), MessageProcessorError> {
    let current_l1_block_number = self.indexer.fetch_block_height().await?
        .saturating_sub(self.confirmations_for_btc_msg) as u32;
    
    if current_l1_block_number <= self.last_processed_bitcoin_block {
        return Ok(());
    }

    // Determine the next chunk to process
    let mut to_block = self.last_processed_bitcoin_block + L1_BLOCKS_CHUNK; // L1_BLOCKS_CHUNK is a constant
    if to_block > current_l1_block_number {
        to_block = current_l1_block_number;
    }

    let messages = self.indexer
        .process_blocks(self.last_processed_bitcoin_block + 1, to_block) // Process the chunk
        .await?;

    for processor in self.message_processors.iter_mut() {
        processor.process_messages(storage, messages.clone(), &mut self.indexer).await?;
    }

    // Update persistent state in the database
    storage.via_indexer_dal().update_last_processed_l1_block(VerifierBtcWatch::module_name(), to_block).await?;
    self.last_processed_bitcoin_block = to_block; // Update in-memory state
    Ok(())
}
```

## Handling Bitcoin Chain Reorganizations

The system includes mechanisms to handle Bitcoin chain reorganizations (reorgs):

1. The `are_blocks_connected` method in `BitcoinInscriptionIndexer` checks if two blocks are connected:
   ```rust
   pub async fn are_blocks_connected(
       &self,
       parent_hash: &BlockHash,
       child_hash: &BlockHash,
   ) -> BitcoinClientResult<bool> {
       let child_block = self.client.fetch_block_by_hash(child_hash).await?;
       let are_connected = child_block.header.prev_blockhash == *parent_hash;
       Ok(are_connected)
   }
   ```

2. When a reorg is detected (TODO comment in the code indicates this is planned but not fully implemented):
   ```rust
   // TODO: check block header is belong to a valid chain of blocks (reorg detection and management)
   ```

3. The system is designed to wait for a configurable number of confirmations (`confirmations_for_btc_msg`) before processing transactions, reducing the risk of reorgs affecting the system.

## Events Monitored

The L1 Watchtower monitors the following events on the Bitcoin blockchain:

1. **System Messages** - Inscriptions from sequencers and verifiers:
   - System bootstrapping
   - Sequencer proposals
   - Validator attestations
   - L1 batch data references
   - Proof data references

2. **Bridge Transactions** - Deposits to the bridge address:
   - L1-to-L2 transfers via inscriptions
   - L1-to-L2 transfers via OP_RETURN

The system identifies relevant transactions by:
- Checking transaction inputs for known addresses (sequencer, verifiers)
- Checking transaction outputs for the bridge address
- Parsing inscriptions and OP_RETURN data for Via protocol messages

## Data Processing and Storage

After extracting messages from Bitcoin transactions, the L1 Watchtower processes and stores them:

1. **L1-to-L2 Messages** (Deposits):
   - Creates L1 transactions with appropriate parameters
   - Stores them in the database via `via_transactions_dal`
   - These will be included in L2 blocks by the sequencer

2. **Verifier Messages** (Attestations and Proofs):
   - Stores votable transactions in the database
   - Records verifier votes
   - Finalizes transactions when they receive enough votes

## Interaction with Other Components

The L1 Watchtower interacts with several other components in the Via L2 system:

1. **Bridge** - The L1 Watchtower monitors deposits to the bridge address and creates corresponding L1-to-L2 messages.

2. **Sequencer** - The L1 Watchtower processes sequencer proposals and L1 batch data references published by the sequencer.

3. **Verifier Network** - The L1 Watchtower processes attestations from verifiers and finalizes transactions based on verifier votes.

4. **Database** - The L1 Watchtower reads its starting block and persists its progress (last processed L1 block) in the `via_indexer_metadata` table.

5. **State Keeper** - The L1-to-L2 messages processed by the L1 Watchtower are used by the State Keeper to update the L2 state.

## Configuration Parameters

Key configuration parameters for the L1 Watchtower include:

- `rpc_url` - URL of the Bitcoin node RPC endpoint
- `network` - Bitcoin network (Mainnet, Testnet, Regtest)
- `node_auth` - Authentication for the Bitcoin node
- `confirmations_for_btc_msg` - Number of confirmations required before processing a message
- `bootstrap_txids` - Transaction IDs used for bootstrapping the system
- `poll_interval` - Interval for polling new blocks
- `start_l1_block_number` - The L1 block number from which indexing begins if no previous state exists or `restart_indexing` is true. The system will start processing from this exact block number.
- `restart_indexing` - Boolean flag; if true, forces the watcher to start indexing from `start_l1_block_number`, ignoring any previously saved state
- `zk_agreement_threshold` - Threshold for verifier agreement (e.g., 2/3)

## Metrics

The L1 Watchtower tracks several metrics to monitor its performance:

- `btc_poll` - Number of times Bitcoin was polled
- `inscriptions_processed` - Number of inscriptions processed, labeled by type
- `errors` - Number of errors encountered, labeled by error type

## Code References

### Bitcoin Client

```rust
// core/lib/via_btc_client/src/client/mod.rs
pub struct BitcoinClient {
    rpc: Arc<dyn BitcoinRpc>,
    network: BitcoinNetwork,
}

impl BitcoinClient {
    pub fn new(rpc_url: &str, network: BitcoinNetwork, auth: NodeAuth) -> BitcoinClientResult<Self> {
        let rpc = BitcoinRpcClient::new(rpc_url, auth)?;
        Ok(Self {
            rpc: Arc::new(rpc),
            network,
        })
    }
    
    // Methods for interacting with Bitcoin
    async fn fetch_block(&self, block_height: u128) -> BitcoinClientResult<Block> {
        self.rpc.get_block_by_height(block_height).await
    }
    
    async fn get_transaction(&self, txid: &Txid) -> BitcoinClientResult<Transaction> {
        self.rpc.get_transaction(txid).await
    }
    
    // ... other methods
}
```

### Inscription Indexer

```rust
// core/lib/via_btc_client/src/indexer/mod.rs
pub struct BitcoinInscriptionIndexer {
    client: Arc<dyn BitcoinOps>,
    parser: MessageParser,
    bridge_address: Address,
    sequencer_address: Address,
    verifier_addresses: Vec<Address>,
    starting_block_number: u32,
}

impl BitcoinInscriptionIndexer {
    pub async fn process_block(
        &mut self,
        block_height: u32,
    ) -> BitcoinIndexerResult<Vec<FullInscriptionMessage>> {
        let block = self.client.fetch_block(block_height as u128).await?;
        
        let mut valid_messages = Vec::new();
        let (system_tx, bridge_tx) = self.extract_important_transactions(&block.txdata);
        
        // Process system transactions
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
        
        // Process bridge transactions
        if let Some(bridge_tx) = bridge_tx {
            let parsed_messages: Vec<_> = bridge_tx
                .iter()
                .flat_map(|tx| self.parser.parse_bridge_transaction(tx, block_height))
                .collect();
                
            let messages: Vec<_> = parsed_messages
                .into_iter()
                .filter(|message| self.is_valid_bridge_message(message))
                .collect();
                
            valid_messages.extend(messages);
        }
        
        Ok(valid_messages)
    }
    
    // ... other methods
}
```

### BTC Watch

```rust
// via_verifier/node/via_btc_watch/src/lib.rs
pub struct VerifierBtcWatch {
    indexer: BitcoinInscriptionIndexer,
    poll_interval: Duration,
    confirmations_for_btc_msg: u64,
    last_processed_bitcoin_block: u32,
    pool: ConnectionPool<Verifier>,
    message_processors: Vec<Box<dyn MessageProcessor>>,
    start_l1_block_number: u32,
}

impl VerifierBtcWatch {
    pub async fn run(mut self, mut stop_receiver: watch::Receiver<bool>) -> anyhow::Result<()> {
        let mut timer = tokio::time::interval(self.poll_interval);
        
        while !*stop_receiver.borrow_and_update() {
            tokio::select! {
                _ = timer.tick() => { /* continue iterations */ }
                _ = stop_receiver.changed() => break,
            }
            
            let mut storage = self.pool.connection_tagged("via_btc_watch").await?;
            match self.loop_iteration(&mut storage).await {
                Ok(()) => { /* everything went fine */ }
                Err(err) => {
                    // Handle errors
                }
            }
        }
        
        Ok(())
    }
    
    async fn loop_iteration(
        &mut self,
        storage: &mut Connection<'_, Verifier>,
    ) -> Result<(), MessageProcessorError> {
        let current_l1_block_number = self.indexer.fetch_block_height().await?
            .saturating_sub(self.confirmations_for_btc_msg) as u32;
            
        if current_l1_block_number <= self.last_processed_bitcoin_block {
            return Ok(());
        }
        
        // Determine the next chunk to process
        let mut to_block = self.last_processed_bitcoin_block + L1_BLOCKS_CHUNK; // L1_BLOCKS_CHUNK is a constant
        if to_block > current_l1_block_number {
            to_block = current_l1_block_number;
        }
        
        let messages = self.indexer
            .process_blocks(self.last_processed_bitcoin_block + 1, to_block) // Process the chunk
            .await?;
            
        for processor in self.message_processors.iter_mut() {
            processor.process_messages(storage, messages.clone(), &mut self.indexer).await?;
        }
        
        // Update persistent state in the database
        storage.via_indexer_dal().update_last_processed_l1_block(VerifierBtcWatch::module_name(), to_block).await?;
        self.last_processed_bitcoin_block = to_block; // Update in-memory state
        Ok(())
    }
}
```

## Conclusion

The L1 Watchtower is a critical component of the Via L2 system, serving as the bridge between the Bitcoin L1 chain and the Layer 2 rollup. It monitors the Bitcoin blockchain for relevant events, processes them, and makes them available to other components of the system. Its polling-based approach, with configurable confirmation requirements, ensures reliable operation even in the face of chain reorganizations.

The component is designed to be robust and efficient, with mechanisms for error handling, retries, and metrics tracking. It plays a crucial role in enabling the Via L2 system to leverage Bitcoin's security and decentralization while providing the scalability and functionality of a Layer 2 solution.