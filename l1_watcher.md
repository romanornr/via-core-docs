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