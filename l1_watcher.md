# Via L2 Bitcoin L1 Watchtower / Listener / Indexer

> **CORRECTION (2026-04-11):** This document conflates two separate services:
> **BtcWatch** (`core/node/via_btc_watch/`) runs inside the core node and handles deposits, votes, and governance.
> **L1 Indexer** (`via_indexer/node/indexer/`) runs as a separate binary with its own database and handles
> deposits and withdrawals for external APIs. They share the `BitcoinInscriptionIndexer` parsing library
> but serve different purposes and maintain separate cursors.
> See the wiki page `via-bitcoin-watchers-btc-watch-and-l1-indexer.md` for verified documentation.

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

Example of client initialization (the network comes from the config):
```rust
let client = BitcoinClient::new(rpc_url, auth, config)?;
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

The indexer processes the full `FullInscriptionMessage` catalog (12 variants; see `inscription_interaction.md`), including:
- `SystemBootstrapping` - Initial system setup
- `ValidatorAttestation` - Verifier votes on proofs
- `L1BatchDAReference` / `ProofDAReference` - References to batch and proof data on Celestia
- `L1ToL2Message` - Deposits from L1 to L2
- `SystemContractUpgradeProposal` / `SystemContractUpgrade` and the `UpdateSequencer` / `UpdateGovernance` / `UpdateBridgeProposal` / `UpdateBridge` wallet-rotation flows
- `BridgeWithdrawal` - Executed bridge withdrawals

(There is no `ProposeSequencer` message anymore; sequencer rotation goes through the `VIA_PROTOCOL:SEQ` OP_RETURN flow.)

Example of indexer initialization (the wallets come from the database, bootstrapped from `bootstrap_txids` at storage init):
```rust
let indexer = BitcoinInscriptionIndexer::new(client, wallets);
```

### 3. Message Processors

Located in `via_verifier/node/via_btc_watch/src/message_processors/`, these components process different types of messages extracted by the indexer.

Key files:
- `mod.rs` - Defines the `MessageProcessor` trait
- `l1_to_l2.rs` - Processes L1-to-L2 transfers (deposits)
- `verifier.rs` - Processes verifier attestations and proof references
- `withdrawal.rs` - Processes executed bridge withdrawals (`BridgeWithdrawal`)
- `system_wallet.rs` - Processes system wallet rotation messages (sequencer, governance, bridge)
- `governance_upgrade.rs` - Processes governance protocol upgrades

Key functionalities:
- Processing L1-to-L2 transfers (deposits)
- Processing verifier attestations
- Finalizing transactions based on verifier votes
- Storing processed messages in the database

#### Withdrawal Processor: idempotent reconciliation

- The `WithdrawalProcessor` (`via_verifier/node/via_btc_watch/src/message_processors/withdrawal.rs`) consumes parsed `BridgeWithdrawal` messages (the bridge's own `VIA_WI` spends detected on Bitcoin).
- For each one it re-derives the paid withdrawal set from the parsed transaction (`get_withdrawal_requests(withdrawal_msg.input.withdrawals)`), then in a single database transaction: inserts or finds the bridge withdrawal record by txid, marks it processed, inserts the withdrawals, and marks them processed against that record.
- The flow is idempotent and independent of who broadcast the transaction, so a verifier that never participated in the signing session converges to the same state (see `withdrawal_finalization.md` for the verbatim code).
- Withdrawal-to-transaction attribution comes from the withdrawal ids embedded per output in the OP_RETURN data, not from a batch index field.

#### Governance upgrade processing (commits 076–077)

The watcher supports two-step governance upgrades: a witness-based proposal inscription (`SystemContractUpgradeProposal`) followed by a governance execution OP_RETURN transaction with prefix "VIA_PROTOCOL:UPGRADE" (`SystemContractUpgrade`). See the dedicated "Governance upgrade processing" section below (under "Interaction with Other Components") for the verbatim indexer, parser, and processor code.

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

        if let Some(last_protocol_version) = storage
            .via_protocol_versions_dal()
            .latest_protocol_semantic_version()
            .await
            .expect("Error load the protocol version")
        {
            check_if_supported_sequencer_version(last_protocol_version)
                .map_err(|e| MessageProcessorError::Internal(anyhow::anyhow!(e.to_string())))?;
        }

        let current_l1_block_number =
            self.indexer
                .fetch_block_height()
                .await
                .map_err(|e| MessageProcessorError::Internal(anyhow::anyhow!(e.to_string())))?
                .saturating_sub(self.config.block_confirmations) as u32;
        if current_l1_block_number <= last_processed_bitcoin_block {
            return Ok(());
        }

        let Some((last_l1_block_number, _)) =
            storage.via_l1_block_dal().get_last_l1_block().await?
        else {
            tracing::warn!("Reorg did not start yet");
            return Ok(());
        };

        let mut to_block = last_processed_bitcoin_block + L1_BLOCKS_CHUNK;
        if to_block > current_l1_block_number {
            to_block = current_l1_block_number;
        }

        // Clamp the to_batch to the last valid block number validated by the reorg detector
        if to_block > last_l1_block_number as u32 {
            to_block = last_l1_block_number as u32;
        }

        let from_block = last_processed_bitcoin_block + 1;

        if to_block < from_block {
            return Ok(());
        }

        let system_wallets_map = match storage
            .via_wallet_dal()
            .get_system_wallets_raw(last_processed_bitcoin_block as i64)
            .await?
        {
            Some(map) => map,
            None => {
                tracing::info!("Wait for storage init, block number {}", from_block);
                return Ok(());
            }
        };

        let system_wallets = SystemWallets::try_from(system_wallets_map)?;

        self.indexer.update_system_wallets(
            Some(system_wallets.sequencer),
            Some(system_wallets.bridge),
            Some(system_wallets.verifiers),
            Some(system_wallets.governance),
        );

        let mut messages = self
            .indexer
            .process_blocks(from_block, to_block)
            .await
            .map_err(|e| MessageProcessorError::Internal(e.into()))?;

        // Re-process blocks if system wallets were updated, since the new wallet state
        // may change how subsequent messages are interpreted.
        if let Some(block_number) = self
            .system_wallet_processor
            .process_messages(storage, messages.clone(), &mut self.indexer)
            .await
            .map_err(|e| MessageProcessorError::Internal(e.into()))?
        {
            // Process the blocks until where the update wallets block.
            to_block = block_number;

            messages = self
                .indexer
                .process_blocks(from_block, to_block)
                .await
                .map_err(|e| MessageProcessorError::Internal(e.into()))?;
        }

        for processor in self.message_processors.iter_mut() {
            processor
                .process_messages(storage, messages.clone(), &mut self.indexer)
                .await
                .map_err(|e| MessageProcessorError::Internal(e.into()))?;
        }

        // Check if the last processed block was updated by another thread. This could happen when a reorg is detected.
        let current_last_processed_bitcoin_block = storage
            .via_indexer_dal()
            .get_last_processed_l1_block(VerifierBtcWatch::module_name())
            .await? as u32;

        if current_last_processed_bitcoin_block != last_processed_bitcoin_block {
            tracing::info!(
                "The btc_watch last processed block was updated by another thread, skipping the block processing"
            );
            return Ok(());
        }

        storage
            .via_indexer_dal()
            .update_last_processed_l1_block(VerifierBtcWatch::module_name(), to_block)
            .await
            .map_err(|e| MessageProcessorError::DatabaseError(e.to_string()))?;

        tracing::info!(
            "The btc_watch processed blocks, from {} to {}",
            from_block,
            to_block,
        );

        Ok(())
    }
```

Note the progress cursor lives only in the database (`via_indexer_dal`), so restarts resume exactly where processing stopped, and the loop refuses to advance during a reorg.

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

### Deterministic deposit PriorityOpId and metadata

- Deposits now carry additional metadata extracted by the indexer:
  - `tx_index`: transaction index within the Bitcoin block
  - `output_vout`: output index (vout) within the transaction
  - The indexer wraps transactions as [`TransactionWithMetadata`](https://github.com/vianetwork/via-core/blob/main/core/lib/via_btc_client/src/types.rs) and plumbs `tx_index`/`output_vout` through the parsing pipeline:
    - Parser updates: [`MessageParser::parse_bridge_transaction`](https://github.com/vianetwork/via-core/blob/main/core/lib/via_btc_client/src/indexer/parser.rs) and inscription/OP_RETURN extraction now set `tx_index`/`output_vout`.
    - Extraction uses `TransactionWithMetadata` across system and bridge paths in [`BitcoinInscriptionIndexer::extract_important_transactions`](https://github.com/vianetwork/via-core/blob/main/core/lib/via_btc_client/src/indexer/mod.rs).

```rust
// core/lib/via_btc_client/src/types.rs
#[derive(Debug, Clone)]
pub struct TransactionWithMetadata {
    pub tx: Transaction,
    pub tx_index: usize,
    pub output_vout: Option<usize>,
}

impl TransactionWithMetadata {
    pub fn new(tx: Transaction, tx_index: usize) -> Self {
        Self {
            tx,
            tx_index,
            output_vout: None,
        }
    }

    pub fn set_output_vout(&mut self, output_vout: usize) {
        self.output_vout = Some(output_vout);
    }
}
```

- PriorityOpId derivation (replaces sequential assignment):
  - Deposits derive a stable priority id from (block, tx_index, vout) via [`ViaL1Deposit::priority_id`](https://github.com/vianetwork/via-core/blob/main/core/lib/types/src/l1/via_l1.rs).
  - Watchers no longer maintain a "next expected priority id" counter; processors construct deposits with required metadata and rely on `priority_id()`:
    - Sequencer watcher: [`L1ToL2MessageProcessor::create_l1_tx_from_message`](https://github.com/vianetwork/via-core/blob/main/core/node/via_btc_watch/src/message_processors/l1_to_l2.rs)
    - Verifier watcher: [`L1ToL2MessageProcessor::create_l1_tx_from_message`](https://github.com/vianetwork/via-core/blob/main/via_verifier/node/via_btc_watch/src/message_processors/l1_to_l2.rs)
    - Indexer-based ingestion in via_indexer aligns to the same scheme: [`L1ToL2MessageProcessor::create_l1_tx_from_message`](https://github.com/vianetwork/via-core/blob/main/via_indexer/node/indexer/src/message_processors/deposit.rs)

The deposit type itself, including the validity check (receiver must not be a kernel-space system contract address, and the deposited amount must cover the L2 gas cost) and the priority id derivation:

```rust
// core/lib/types/src/l1/via_l1.rs
/// Eth 18 decimals - BTC 8 decimals
const MANTISSA: u64 = 10_000_000_000;

/// Deposit default L2 gas price.
const MAX_FEE_PER_GAS: u64 = 120_000_000;

/// Gas limit to required to execute a deposit.
const GAS_LIMIT: u64 = 300_000;

/// Max system contracts kernel space address.
const MAX_SYSTEM_CONTRACT_ADDRESS: &str = "0x000000000000000000000000000000000000ffff";

#[derive(Debug, Clone)]

pub struct ViaL1Deposit {
    pub l2_receiver_address: Address,
    pub amount: u64,
    pub calldata: Vec<u8>,
    pub l1_block_number: u64,
    pub tx_index: usize,
    pub output_vout: usize,
}

impl ViaL1Deposit {
    pub fn is_valid_deposit(&self) -> bool {
        if self.l2_receiver_address <= H160::from_str(MAX_SYSTEM_CONTRACT_ADDRESS).unwrap() {
            return false;
        }

        // CHeck if the amount can cover the transaction cost.
        let gas_fee = U256::from(GAS_LIMIT) * U256::from(MAX_FEE_PER_GAS);
        self.value() >= gas_fee
    }

    pub fn l1_tx(&self) -> Option<L1Tx> {
        if !self.is_valid_deposit() {
            return None;
        }
        Some(L1Tx::from(self.clone()))
    }

    fn value(&self) -> U256 {
        U256::from(self.amount) * U256::from(MANTISSA)
    }

    pub fn priority_id(&self) -> PriorityOpId {
        PriorityOpId(
            ViaPriorityOpId::new(
                self.l1_block_number,
                self.tx_index as u64,
                self.output_vout as u64,
            )
            .raw(),
        )
    }
}
```

The sequencer-side processor (core node BtcWatch) builds the deposit from the parsed message and treats missing `tx_index`/`output_vout` as errors:

```rust
// core/node/via_btc_watch/src/message_processors/l1_to_l2.rs
impl L1ToL2MessageProcessor {
    fn create_l1_tx_from_message(
        &self,
        msg: &L1ToL2Message,
    ) -> Result<Option<L1Tx>, MessageProcessorError> {
        let deposit = ViaL1Deposit {
            l2_receiver_address: msg.input.receiver_l2_address,
            amount: msg.amount.to_sat(),
            calldata: msg.input.call_data.clone(),
            l1_block_number: msg.common.block_height as u64,
            tx_index: msg.common.tx_index.ok_or_else(|| {
                MessageProcessorError::Internal(anyhow::anyhow!("deposit missing tx_index"))
            })?,
            output_vout: msg.common.output_vout.ok_or_else(|| {
                MessageProcessorError::Internal(anyhow::anyhow!("deposit missing output_vout"))
            })?,
        };

        if let Some(l1_tx) = deposit.l1_tx() {
            tracing::info!(
                "Created L1 transaction with serial id {:?} (block {}) with deposit amount {} and tx hash {}",
                l1_tx.common_data.serial_id,
                l1_tx.common_data.eth_block,
                deposit.amount,
                l1_tx.common_data.canonical_tx_hash,
            );
            return Ok(Some(l1_tx));
        }
        Ok(None)
    }
}
```

Operational implications:
- Deterministic ordering: PriorityOpId is stable across replays and independent of in-memory counters.
- Processors must ensure `tx_index` and `output_vout` are present on parsed messages; missing fields are treated as errors in the updated processors.
## System wallets bootstrap and bridge configuration

Initialization:
- Main node adds a storage init layer that scans the configured bootstrap_txids, constructs SystemWallets (sequencer, verifiers, governance, bridge) and persists them in via_wallets.
- Verifier nodes perform the same wallet initialization path during their storage init stage and expose the loaded wallets to downstream components (e.g., the indexer / watchers).

System wallet updates at runtime:
- A dedicated SystemWalletProcessor parses wallet update inscriptions (sequencer, governance and bridge update flows).
- If the wallets set changes, the watcher re-processes the current block range to apply parsing/validation rules under the new wallet set (e.g., bridge address change affecting deposit classification).
- BTC sender validates that the active inscriber address equals the sequencer address from SystemWallets before sending inscriptions.

The shared wallet set type:

```rust
// core/lib/types/src/via_wallet.rs
#[derive(Debug, Clone, PartialEq)]
pub struct SystemWallets {
    pub sequencer: Address,
    pub verifiers: Vec<Address>,
    pub governance: Address,
    pub bridge: Address,
}

impl SystemWallets {
    pub fn is_valid_bridge_address(&self, bridge_address: Address) -> anyhow::Result<()> {
        if self.bridge != bridge_address {
            anyhow::bail!(
                "bridge address mismatch, expected one of {:?}, found {}",
                &self.bridge,
                bridge_address
            );
        }
        Ok(())
    }

    pub fn is_valid_verifier_address(&self, verifier: Address) -> anyhow::Result<()> {
        if !self.verifiers.contains(&verifier) {
            anyhow::bail!(
                "Verifier address not found in the verifiers set, expected one of {:?}, found {}",
                &self.verifiers,
                verifier
            );
        }
        Ok(())
    }

    pub fn is_valid_sequencer_address(&self, sequencer: Address) -> anyhow::Result<()> {
        if self.sequencer != sequencer {
            anyhow::bail!(
                "Sequencer address mismatch, expected {}, found {}",
                self.sequencer.to_string(),
                sequencer.to_string()
            );
        }
        Ok(())
    }
}
```

The core node's SystemWalletProcessor dispatches on the three wallet-rotation messages and returns the earliest affected block number, which the watch loop uses to re-process the range under the new wallet set:

```rust
// core/node/via_btc_watch/src/message_processors/system_wallet.rs
#[async_trait::async_trait]
impl MessageProcessor for SystemWalletProcessor {
    async fn process_messages(
        &mut self,
        storage: &mut Connection<'_, Core>,
        msgs: Vec<FullInscriptionMessage>,
        _: &mut BitcoinInscriptionIndexer,
    ) -> Result<Option<u32>, MessageProcessorError> {
        let mut l1_block_number: Option<u32> = None;

        for msg in FullInscriptionMessage::sort_messages(msgs) {
            let l1_block_number_opt = match msg {
                FullInscriptionMessage::UpdateGovernance(m) => {
                    self.handle_update_governance(storage, m).await?
                }
                FullInscriptionMessage::UpdateSequencer(m) => {
                    self.handle_update_sequencer(storage, m).await?
                }
                FullInscriptionMessage::UpdateBridge(m) => {
                    self.handle_update_bridge_proposal(storage, m).await?
                }
                _ => None,
            };

            // Keep the smallest (earliest) block number
            match (l1_block_number, l1_block_number_opt) {
                (Some(current), Some(new)) if new < current => l1_block_number = Some(new),
                (None, Some(new)) => l1_block_number = Some(new),
                _ => {}
            }
        }

        Ok(l1_block_number)
    }
}
```

Each handler (`handle_update_bridge_proposal`, `handle_update_sequencer`, `handle_update_governance`) loads the current wallets via `via_wallet_dal().get_system_wallets_raw(block_height)`, skips no-op updates, and persists a `SystemWalletsDetails` entry with `insert_wallets(...)`. The bridge handler additionally fetches the referenced proposal transaction from the BTC node, re-parses it with `MessageParser::parse_system_transaction`, and stores both the new bridge address and the new verifier set. The verifier node has a mirrored processor in `via_verifier/node/via_btc_watch/src/message_processors/system_wallet.rs`.

Bridge config:
- The watcher and API use the dedicated bridge configuration (via_bridge.toml or VIA_BRIDGE_* env variables) for thresholds and wiring. The active bridge address is sourced from the DB (via_wallets) at runtime.

Canonical chain gating (verifier watcher):
- New insertion rules enforce strict sequential numbering and link-by-hash to the canonical head.
- Duplicate proof reveal transactions are eliminated early via a proof_reveal_tx_exists check.
- Operators can request a canonical chain continuity report via `ViaVotesDal::verify_canonical_chain` (`via_verifier/lib/verifier_dal/src/via_votes_dal.rs`) to diagnose gaps and invalid links.

The gating logic lives in the verifier's `VerifierMessageProcessor` when it handles a `ProofDAReference` inscription:

```rust
// via_verifier/node/via_btc_watch/src/message_processors/verifier.rs
                FullInscriptionMessage::ProofDAReference(ref proof_msg) => {
                    let proof_reveal_tx_id = convert_txid_to_h256(proof_msg.common.tx_id);

                    if storage
                        .via_votes_dal()
                        .proof_reveal_tx_exists(proof_reveal_tx_id.as_bytes())
                        .await?
                    {
                        tracing::info!(
                            "Skipping duplicate proof reveal tx: {:?}",
                            proof_reveal_tx_id
                        );
                        continue;
                    }

                    let pubdata_msgs = indexer
                        .parse_transaction(&proof_msg.input.l1_batch_reveal_txid)
                        .await?;

                    if pubdata_msgs.len() != 1 {
                        return Err(MessageProcessorError::Internal(anyhow::Error::msg(
                            "Invalid pubdata msg lenght",
                        )));
                    }

                    let inscription = pubdata_msgs[0].clone();

                    let l1_batch_da_ref_inscription = match inscription {
                        FullInscriptionMessage::L1BatchDAReference(da_msg) => da_msg,
                        _ => {
                            return Err(MessageProcessorError::Internal(anyhow::Error::msg(
                                "Invalid inscription type",
                            )))
                        }
                    };

                    let new_l1_batch_number = l1_batch_da_ref_inscription.input.l1_batch_index.0;

                    tracing::info!(
                        "Processing ProofDAReference for batch {} with hash {:?}",
                        new_l1_batch_number,
                        l1_batch_da_ref_inscription.input.l1_batch_hash
                    );

                    if new_l1_batch_number == 0 {
                        tracing::info!(
                            "Skipping ProofDAReference message with l1_batch_number ZERO."
                        );
                        continue;
                    } else if new_l1_batch_number == 1 {
                        if storage.via_votes_dal().batch_exists(1).await? {
                            tracing::info!("Skipping duplicate genesis batch 1");
                            continue;
                        }
                    } else if new_l1_batch_number > 1 {
                        let last_batch_in_canonical_chain = match storage
                            .via_votes_dal()
                            .get_last_batch_in_canonical_chain()
                            .await?
                        {
                            Some(last_batch_in_canonical_chain) => last_batch_in_canonical_chain,
                            None => {
                                return Err(MessageProcessorError::Internal(anyhow::Error::msg(
                                    "Last batch in canonical chain not found",
                                )))
                            }
                        };

                        if last_batch_in_canonical_chain.0 + 1 != new_l1_batch_number {
                            // Possible reorg: validate whether the batch is a fork of a previously reverted batch.
                            // If this batch is valid (i.e., a fork of a previously valid batch),
                            // the verifier treats it as a new fork, implicitly marking all earlier batches as invalid.
                            let parent_hash = l1_batch_da_ref_inscription
                                .input
                                .prev_l1_batch_hash
                                .as_bytes()
                                .to_vec();

                            let exists = storage
                                .via_votes_dal()
                                .get_parent_batch_exists_for_l1_batch(
                                    new_l1_batch_number as i64,
                                    &parent_hash,
                                )
                                .await?;
                            if !exists {
                                tracing::info!(
                                    "Skipping ProofDAReference, no valid parent found for the new l1_batch_number: {:?}",
                                    new_l1_batch_number,
                                );
                                continue;
                            }

                            tracing::info!(
                                "A Revert batch was found with number {}, parent hash {}",
                                new_l1_batch_number,
                                l1_batch_da_ref_inscription.input.prev_l1_batch_hash
                            );

                            let from_l1_batch_number = (new_l1_batch_number - 1) as i64;

                            let mut transaction = storage.start_transaction().await?;
                            transaction
                                .via_votes_dal()
                                .delete_votable_transactions(from_l1_batch_number)
                                .await?;

                            transaction
                                .via_transactions_dal()
                                .reset_transactions(from_l1_batch_number)
                                .await?;

                            transaction.commit().await?;

                            METRICS.inscriptions_processed[&InscriptionStage::Reorg]
                                .set(from_l1_batch_number as usize);
                        } else {
                            if last_batch_in_canonical_chain.1
                                != l1_batch_da_ref_inscription.input.prev_l1_batch_hash.0
                            {
                                tracing::info!(
                                "Skipping ProofDAReference message with l1_batch_number: {:?}. Last batch in canonical chain: {:?}",
                                l1_batch_da_ref_inscription.input.l1_batch_index,
                                last_batch_in_canonical_chain
                            );
                                continue;
                            }
                        }
                    }

                    METRICS.inscriptions_processed[&InscriptionStage::IndexedL1Batch]
                        .set(new_l1_batch_number as usize);

                    storage
                        .via_votes_dal()
                        .insert_votable_transaction(
                            new_l1_batch_number,
                            l1_batch_da_ref_inscription.input.l1_batch_hash,
                            l1_batch_da_ref_inscription.input.prev_l1_batch_hash,
                            proof_msg.input.da_identifier.clone(),
                            proof_reveal_tx_id,
                            proof_msg.input.blob_id.clone(),
                            proof_msg.input.l1_batch_reveal_txid.to_string(),
                            l1_batch_da_ref_inscription.input.blob_id,
                        )
                        .await?;

                    tracing::info!(
                        "New votable transaction for L1 batch {:?}",
                        new_l1_batch_number
                    );
                }
```

External node:
- External nodes can optionally run the BTC Watch in follower mode. When enabled, governance upgrade processing is disabled by configuration (the watcher still indexes L1ToL2 and votable messages).

API signature changes in parser:
- System transaction parsing accepts an optional wallets context; callers pass the loaded wallets when available.

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

### Governance upgrade processing

- Two-phase design:
  - Proposal (witness-based): SystemContractUpgradeProposal is parsed from witness scripts in "system" transactions.
  - Execution (OP_RETURN): SystemContractUpgrade is parsed from OP_RETURN outputs with prefix "VIA_PROTOCOL:UPGRADE", carrying the referenced proposal txid; inputs authorize governance execution.
- Indexer extraction:
  - The indexer classifies important txs as (gov_txs, system_txs, bridge_txs); gov_txs are outputs to the configured governance address. See [core/lib/via_btc_client/src/indexer/mod.rs](https://github.com/vianetwork/via-core/blob/main/core/lib/via_btc_client/src/indexer/mod.rs).
  - Validity checks ensure the governance execution input belongs to the governance address (`is_valid_gov_message`, which delegates to `is_valid_gov_upgrade`):

```rust
// core/lib/via_btc_client/src/indexer/mod.rs
    async fn is_valid_gov_message(&self, message: &FullInscriptionMessage) -> bool {
        let maybe_input = match message {
            FullInscriptionMessage::SystemContractUpgrade(m) => m.input.inputs.first(),
            FullInscriptionMessage::UpdateBridge(m) => m.input.inputs.first(),
            FullInscriptionMessage::UpdateSequencer(m) => m.input.inputs.first(),
            FullInscriptionMessage::UpdateGovernance(m) => m.input.inputs.first(),
            _ => return false,
        };

        self.is_valid_gov_upgrade(maybe_input)
            .await
            .unwrap_or(false)
    }
```

```rust
// core/lib/via_btc_client/src/indexer/mod.rs
    #[instrument(skip(self, outpoint_opt), target = "bitcoin_indexer")]
    async fn is_valid_gov_upgrade(&self, outpoint_opt: Option<&OutPoint>) -> anyhow::Result<bool> {
        if let Some(outpoint) = outpoint_opt {
            let tx = self.client.get_transaction(&outpoint.txid).await?;
            if let Some(txout) = tx.output.get(outpoint.vout as usize) {
                return Ok(txout.script_pubkey == self.wallets.governance.script_pubkey());
            }
        }
        Ok(false)
    }
```

- Parsing:
  - OP_RETURN execution: `parse_protocol_upgrade_transactions()` dispatches to the per-message OP_RETURN parsers (protocol upgrade plus the governance, bridge, and sequencer wallet updates); `parse_op_return_protocol_upgrade()` extracts the proposal_tx_id and constructs a SystemContractUpgrade message including OutPoints.

```rust
// core/lib/via_btc_client/src/indexer/parser.rs
    #[instrument(skip(self, tx), target = "bitcoin_indexer::parser")]
    pub fn parse_protocol_upgrade_transactions(
        &mut self,
        tx: &TransactionWithMetadata,
        block_height: u32,
    ) -> Vec<FullInscriptionMessage> {
        let mut messages = Vec::new();

        if let Some(update_sequencer) = self.parse_op_return_update_governance(&tx.tx, block_height)
        {
            messages.push(update_sequencer);
        }

        if let Some(upgrade_protocol) = self.parse_op_return_protocol_upgrade(&tx.tx, block_height)
        {
            messages.push(upgrade_protocol);
        }

        if let Some(update_bridge) = self.parse_op_return_update_bridge(&tx.tx, block_height) {
            messages.push(update_bridge);
        }

        if let Some(update_sequencer) = self.parse_op_return_update_sequencer(&tx.tx, block_height)
        {
            messages.push(update_sequencer);
        }

        messages
    }
```

  - Proposal (witness): parse_system_transaction() continues to handle SystemContractUpgradeProposal in system transactions.
- Watcher processors:
  - Both Core and Verifier watchers wire a GovernanceUpgradesEventProcessor which receives a BitcoinClient handle to fetch the referenced proposal transaction, and a MessageParser instance:
    - [core/node/via_btc_watch/src/message_processors/governance_upgrade.rs](https://github.com/vianetwork/via-core/blob/main/core/node/via_btc_watch/src/message_processors/governance_upgrade.rs)
    - [via_verifier/node/via_btc_watch/src/message_processors/governance_upgrade.rs](https://github.com/vianetwork/via-core/blob/main/via_verifier/node/via_btc_watch/src/message_processors/governance_upgrade.rs)

The core node's processor, verbatim:

```rust
// core/node/via_btc_watch/src/message_processors/governance_upgrade.rs
/// Listens to operation events coming from the governance contract and saves new protocol upgrade proposals to the database.
#[derive(Debug)]
pub struct GovernanceUpgradesEventProcessor {
    /// Last protocol version seen. Used to skip events for already known upgrade proposals.
    last_seen_protocol_version: ProtocolSemanticVersion,
    /// BTC client
    btc_client: Arc<BitcoinClient>,
    /// Message parser
    message_parser: MessageParser,
    /// upgrade proposal
    upgrade: ViaProtocolUpgrade,
}

impl GovernanceUpgradesEventProcessor {
    pub fn new(
        btc_client: Arc<BitcoinClient>,
        last_seen_protocol_version: ProtocolSemanticVersion,
    ) -> Self {
        let message_parser = MessageParser::new(btc_client.get_network());
        Self {
            last_seen_protocol_version,
            btc_client,
            message_parser,
            upgrade: ViaProtocolUpgrade::default(),
        }
    }
}
#[async_trait::async_trait]
impl MessageProcessor for GovernanceUpgradesEventProcessor {
    async fn process_messages(
        &mut self,
        storage: &mut Connection<'_, Core>,
        msgs: Vec<FullInscriptionMessage>,
        _: &mut BitcoinInscriptionIndexer,
    ) -> Result<Option<u32>, MessageProcessorError> {
        let mut upgrades = Vec::new();
        for msg in msgs {
            if let FullInscriptionMessage::SystemContractUpgrade(system_contract_upgrade_msg) = &msg
            {
                let proposal_tx = self
                    .btc_client
                    .get_transaction(&system_contract_upgrade_msg.input.proposal_tx_id)
                    .await
                    .map_err(|err| {
                        MessageProcessorError::Internal(anyhow::anyhow!(
                            "Failed to fetch protocol upgrade transaction: {}, error {}",
                            system_contract_upgrade_msg.input.proposal_tx_id,
                            err
                        ))
                    })?;

                let messages = self.message_parser.parse_system_transaction(
                    &proposal_tx,
                    system_contract_upgrade_msg.common.block_height,
                    None,
                );

                for message in messages {
                    match message {
                        FullInscriptionMessage::SystemContractUpgradeProposal(
                            system_contract_upgrade_proposal_msg,
                        ) => {
                            // Ignore if old version
                            if system_contract_upgrade_proposal_msg.input.version
                                <= self.last_seen_protocol_version
                            {
                                tracing::info!(
                                    "Upgrade transaction with version {} already processed, skipping",
                                    system_contract_upgrade_proposal_msg.input.version
                                );
                                continue;
                            }

                            tracing::info!(
                                "Received upgrades with versions: {:?}",
                                system_contract_upgrade_proposal_msg.input.version
                            );
                            let tx = self.upgrade.create_protocol_upgrade_tx(
                                system_contract_upgrade_proposal_msg.input.version,
                                system_contract_upgrade_proposal_msg.input.system_contracts,
                            )?;

                            let upgrade = ProtocolUpgrade {
                                version: system_contract_upgrade_proposal_msg.input.version,
                                bootloader_code_hash: Some(
                                    system_contract_upgrade_proposal_msg
                                        .input
                                        .bootloader_code_hash,
                                ),
                                default_account_code_hash: Some(
                                    system_contract_upgrade_proposal_msg
                                        .input
                                        .default_account_code_hash,
                                ),
                                evm_emulator_code_hash: system_contract_upgrade_proposal_msg
                                    .input
                                    .evm_emulator_code_hash,
                                tx: Some(tx),
                                timestamp: 0,
                                verifier_address: None,
                                verifier_params: None,
                            };
                            upgrades.push((
                                upgrade,
                                system_contract_upgrade_proposal_msg
                                    .input
                                    .recursion_scheduler_level_vk_hash,
                            ));
                        }
                        _ => (),
                    }
                }
            }
        }

        let Some(last_upgrade) = upgrades.last() else {
            return Ok(None);
        };

        let last_version = last_upgrade.0.version;
        for (upgrade, recursion_scheduler_level_vk_hash) in upgrades {
            let latest_semantic_version = storage
                .protocol_versions_dal()
                .latest_semantic_version()
                .await
                .map_err(DalError::generalize)?
                .context("expected some version to be present in DB")?;

            if upgrade.version > latest_semantic_version {
                let latest_version = storage
                    .protocol_versions_dal()
                    .get_protocol_version_with_latest_patch(latest_semantic_version.minor)
                    .await
                    .map_err(DalError::generalize)?
                    .with_context(|| {
                        format!(
                            "expected minor version {} to be present in DB",
                            latest_semantic_version.minor as u16
                        )
                    })?;

                let new_version =
                    latest_version.apply_upgrade(upgrade, Some(recursion_scheduler_level_vk_hash));

                storage
                    .protocol_versions_dal()
                    .save_protocol_version_with_tx(&new_version)
                    .await
                    .map_err(DalError::generalize)?;

                METRICS.inscriptions_processed[&InscriptionStage::Upgrade]
                    .set(new_version.version.minor as usize);
            }
        }
        self.last_seen_protocol_version = last_version;

        Ok(None)
    }
}
```

  - The processor constructs the L2 ProtocolUpgradeTx using ViaProtocolUpgrade::create_protocol_upgrade_tx(), filling canonical_tx_hash with the L2 canonical transaction hash derived from the proposal:

```rust
// core/lib/types/src/via_protocol_upgrade.rs
impl ViaProtocolUpgrade {
    pub fn create_protocol_upgrade_tx(
        &self,
        version: ProtocolSemanticVersion,
        system_contracts: Vec<(Address, H256)>,
    ) -> anyhow::Result<ProtocolUpgradeTx> {
        let canonical_tx_hash = self.get_canonical_tx_hash(version, system_contracts.clone())?;

        let tx = ProtocolUpgradeTx {
            execute: Execute {
                contract_address: Some(CONTRACT_DEPLOYER_ADDRESS),
                calldata: self.get_calldata(system_contracts)?,
                value: U256::zero(),
                factory_deps: vec![],
            },
            common_data: ProtocolUpgradeTxCommonData {
                sender: CONTRACT_FORCE_DEPLOYER_ADDRESS,
                upgrade_id: version.minor,
                max_fee_per_gas: U256::zero(),
                gas_limit: U256::from(GAS_LIMIT),
                gas_per_pubdata_limit: U256::from(GAS_PER_PUB_DATA_BYTE_LIMIT),
                eth_block: 0,
                canonical_tx_hash,
                to_mint: U256::zero(),
                refund_recipient: Address::zero(),
            },
            received_timestamp_ms: unix_timestamp_ms(),
        };

        Ok(tx)
    }
```

- State changes:
  - On acceptance, new protocol version and updated base system contracts hashes are saved via protocol_versions_dal.save_protocol_version_with_tx(...), enabling nodes to switch verification keys post-upgrade.

## Configuration Parameters

The watcher itself is configured by `ViaBtcWatchConfig`; the chunk size is a compile-time constant exported from the same module:

```rust
// core/lib/config/src/configs/via_btc_watch.rs
use std::time::Duration;

use serde::{Deserialize, Serialize};

/// Total L1 blocks to process at a time.
pub const L1_BLOCKS_CHUNK: u32 = 50;

/// Configuration for the Bitcoin watch crate.
#[derive(Debug, Deserialize, Default, Serialize, Clone, PartialEq)]
pub struct ViaBtcWatchConfig {
    /// Service interval in milliseconds.
    pub poll_interval: u64,

    /// Minimum confirmation blocks for an inscription to be processed.
    pub block_confirmations: u64,

    /// The starting L1 block number from which indexing begins
    pub start_l1_block_number: u32,

    /// When set to true, the btc_watch starts indexing L1 blocks from the "start_l1_block_number".
    pub restart_indexing: bool,
}

impl ViaBtcWatchConfig {
    /// Converts `self.poll_interval` into `Duration`.
    pub fn poll_interval(&self) -> Duration {
        Duration::from_millis(self.poll_interval)
    }
}
```

- `poll_interval` - Interval (milliseconds) for polling new blocks
- `block_confirmations` - Number of confirmations required before processing a message (the loop subtracts this from the current tip)
- `start_l1_block_number` - The L1 block number from which indexing begins if no previous state exists or `restart_indexing` is true. The system will start processing from this exact block number.
- `restart_indexing` - Boolean flag; if true, forces the watcher to start indexing from `start_l1_block_number`, ignoring any previously saved state

Connection-level parameters are not part of `ViaBtcWatchConfig`: `rpc_url` and `node_auth` come from the Bitcoin client wiring (secrets), the `network` and fee settings live in `ViaBtcClientConfig` (`core/lib/config/src/configs/via_btc_client.rs`), and `bootstrap_txids` (used to build the initial SystemWallets) belong to the genesis/bootstrap configuration consumed by the storage init layer.

The verifier agreement threshold is not a watcher config value: it is the shared constant `BATCH_FINALIZATION_THRESHOLD = 0.66` (`core/lib/via_consensus/src/consensus.rs`), imported by the verifier message processor.

## Metrics

There is no `btc_poll` metric; each watcher registers its own vise metrics struct. The core node's BtcWatch:

```rust
// core/node/via_btc_watch/src/metrics.rs
use vise::{Counter, EncodeLabelSet, EncodeLabelValue, Family, Gauge, LabeledFamily, Metrics};

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, EncodeLabelValue, EncodeLabelSet)]
#[metrics(label = "stage", rename_all = "snake_case")]
pub enum InscriptionStage {
    Vote,
    Upgrade,
}

#[derive(Debug, Metrics)]
#[metrics(prefix = "via_server_btc_watch")]
pub struct ViaBtcWatcherMetrics {
    #[metrics(labels = ["role", "address"])]
    pub system_wallets: LabeledFamily<(String, String), Counter, 2>,

    /// Number of inscriptions processed, labeled by type.
    pub inscriptions_processed: Family<InscriptionStage, Gauge<usize>>,

    /// Deposit processed.
    pub deposit: Counter,

    /// Counter to store the layer errors.
    pub errors: Counter,
}

#[vise::register]
pub static METRICS: vise::Global<ViaBtcWatcherMetrics> = vise::Global::new();
```

The verifier's watcher adds withdrawal and reorg tracking under its own prefix:

```rust
// via_verifier/node/via_btc_watch/src/metrics.rs
use vise::{Counter, EncodeLabelSet, EncodeLabelValue, Family, Gauge, LabeledFamily, Metrics};

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, EncodeLabelValue, EncodeLabelSet)]
#[metrics(label = "stage", rename_all = "snake_case")]
pub enum InscriptionStage {
    IndexedL1Batch,
    Upgrade,
    Finalized,
    Reorg,
    Vote,
}

#[derive(Debug, Metrics)]
#[metrics(prefix = "via_verifier_btc_watch")]
pub struct ViaVerifierBtcWatcherMetrics {
    #[metrics(labels = ["role", "address"])]
    pub system_wallets: LabeledFamily<(String, String), Counter, 2>,

    /// Number of inscriptions processed, labeled by type.
    pub inscriptions_processed: Family<InscriptionStage, Gauge<usize>>,

    /// Deposit processed.
    pub deposit: Counter,

    /// Withdrawal confirmed.
    pub withdrawal_confirmed: Counter,

    /// Errors
    pub errors: Counter,
}

#[vise::register]
pub static METRICS: vise::Global<ViaVerifierBtcWatcherMetrics> = vise::Global::new();
```

## Code References

### Bitcoin Client

```rust
// core/lib/via_btc_client/src/client/mod.rs
#[derive(Debug)]
pub struct BitcoinClient {
    rpc: Arc<dyn BitcoinRpc>,
    pub config: ViaBtcClientConfig,
}

impl BitcoinClient {
    #[instrument(skip(auth), target = "bitcoin_client")]
    pub fn new(
        rpc_url: &str,
        auth: NodeAuth,
        config: ViaBtcClientConfig,
    ) -> BitcoinClientResult<Self>
    where
        Self: Sized,
    {
        debug!("Creating new BitcoinClient");
        let rpc = BitcoinRpcClient::new(rpc_url, auth)?;
        Ok(Self {
            rpc: Arc::new(rpc),
            config,
        })
    }
}
```

The network is derived from `config.network()`; the config also carries the fee estimation settings (`external_apis`, `fee_strategies`, `use_rpc_for_fee_rate`; see `fee_mechanism.md`).

### Inscription Indexer

```rust
// core/lib/via_btc_client/src/indexer/mod.rs
/// The main indexer struct for processing Bitcoin inscriptions
#[derive(Debug, Clone)]
pub struct BitcoinInscriptionIndexer {
    client: Arc<dyn BitcoinOps>,
    wallets: Arc<SystemWallets>,
    parser: MessageParser,
}

impl BitcoinInscriptionIndexer {
    #[instrument(
        skip(client, wallets)
        target = "bitcoin_indexer"
    )]
    pub fn new(client: Arc<BitcoinClient>, wallets: Arc<SystemWallets>) -> Self {
        Self {
            client: client.clone(),
            parser: MessageParser::new(client.get_network()),
            wallets,
        }
    }
}
```

The addresses (sequencer, governance, bridge, verifiers) are no longer individual fields; they come from the shared `SystemWallets`, loaded from the database at startup and updated at runtime by the `SystemWalletProcessor`. `process_block` classifies transactions into **three** groups via `extract_important_transactions(&block.txdata)`:

- `gov_txs`: transactions paying the governance address, parsed as OP_RETURN execution messages (`parse_protocol_upgrade_transaction`), then validated with `is_valid_gov_message` (the spending input must belong to the governance address)
- `system_txs`: witness-based inscriptions from known system addresses, parsed with `parse_system_transaction` and validated with `is_valid_system_message`
- `bridge_txs`: transactions touching the bridge address, parsed with `parse_bridge_transaction` (deposits in, `BridgeWithdrawal` spends out); bridge withdrawal validation is async, fetching the referenced input UTXO to confirm its script belongs to the bridge

### BTC Watch

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

Polling interval, confirmation depth, start block, and `restart_indexing` all live in `ViaBtcWatchConfig`; `L1_BLOCKS_CHUNK` is exported from the same config module (`zksync_config::configs::via_btc_watch::L1_BLOCKS_CHUNK`). The full verbatim `loop_iteration` body (reorg gate, DB-persisted cursor, protocol version check, chunk clamping, wallet-aware re-processing) is shown in the Scanning/Indexing Strategy section above. The core node's `BtcWatch` (`core/node/via_btc_watch/src/lib.rs`) mirrors the same structure over the Core database.

## Conclusion

The L1 Watchtower is a critical component of the Via L2 system, serving as the bridge between the Bitcoin L1 chain and the Layer 2 rollup. It monitors the Bitcoin blockchain for relevant events, processes them, and makes them available to other components of the system. Its polling-based approach, with configurable confirmation requirements, ensures reliable operation even in the face of chain reorganizations.

The component is designed to be robust and efficient, with mechanisms for error handling, retries, and metrics tracking. It plays a crucial role in enabling the Via L2 system to leverage Bitcoin's security and decentralization while providing the scalability and functionality of a Layer 2 solution.

---

## Priority OpId Encoding

- PriorityOpId encoding and dedicated type
  - The encoding moved to a dedicated type, `ViaPriorityOpId`, with revised bit distribution: 28 bits for block_number, 20 bits for tx_index, 16 bits for vout.
  - Deposits now build their priority id using this type via `ViaL1Deposit::priority_id()` (see the verbatim code in "Deterministic deposit PriorityOpId and metadata" above).
  - Ordering semantics are unchanged: the raw u64 remains the sort key for deterministic ordering and replayability.

```rust
// core/lib/types/src/l1/priority_id.rs
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct ViaPriorityOpId(pub u64);

/// VIA priorityId
/// 28 bits for block (268M blocks = 5,100 years when block time = 10min)
/// 20 bits for tx_index (1M transactions per block)
/// 16 bits for vout (65k outputs per transaction)
impl ViaPriorityOpId {
    // Constants for bit field sizes
    const BLOCK_BITS: u32 = 28;
    const TX_INDEX_BITS: u32 = 20;
    const VOUT_BITS: u32 = 16;

    // Bit masks
    const BLOCK_MASK: u64 = (1u64 << Self::BLOCK_BITS) - 1; // 0xFFFFFFF
    const TX_INDEX_MASK: u64 = (1u64 << Self::TX_INDEX_BITS) - 1; // 0xFFFFF
    const VOUT_MASK: u64 = (1u64 << Self::VOUT_BITS) - 1; // 0xFFFF

    // Bit positions
    const TX_INDEX_SHIFT: u32 = Self::VOUT_BITS;
    const BLOCK_SHIFT: u32 = Self::TX_INDEX_BITS + Self::VOUT_BITS;

    // Maximum values
    pub const MAX_BLOCK_NUMBER: u64 = Self::BLOCK_MASK;
    pub const MAX_TX_INDEX: u64 = Self::TX_INDEX_MASK;
    pub const MAX_VOUT: u64 = Self::VOUT_MASK;

    /// Creates a new PriorityOpId from components
    pub fn new(block_number: u64, tx_index: u64, vout: u64) -> Self {
        debug_assert!(
            block_number <= Self::MAX_BLOCK_NUMBER,
            "Block number {} exceeds maximum {}",
            block_number,
            Self::MAX_BLOCK_NUMBER
        );
        debug_assert!(
            tx_index <= Self::MAX_TX_INDEX,
            "TX index {} exceeds maximum {}",
            tx_index,
            Self::MAX_TX_INDEX
        );
        debug_assert!(
            vout <= Self::MAX_VOUT,
            "VOut {} exceeds maximum {}",
            vout,
            Self::MAX_VOUT
        );

        Self(
            ((block_number & Self::BLOCK_MASK) << Self::BLOCK_SHIFT)
                | ((tx_index & Self::TX_INDEX_MASK) << Self::TX_INDEX_SHIFT)
                | (vout & Self::VOUT_MASK),
        )
    }

    /// Extracts the block number
    pub fn block_number(&self) -> u64 {
        (self.0 >> Self::BLOCK_SHIFT) & Self::BLOCK_MASK
    }

    /// Extracts the transaction index
    pub fn tx_index(&self) -> u64 {
        (self.0 >> Self::TX_INDEX_SHIFT) & Self::TX_INDEX_MASK
    }

    /// Extracts the output vout
    pub fn vout(&self) -> u64 {
        self.0 & Self::VOUT_MASK
    }

    /// Gets the raw u64 value
    pub fn raw(&self) -> u64 {
        self.0
    }
}
```

- Async bridge-withdrawal validation in the indexer
  - Bridge withdrawals are validated against the BTC node: the indexer fetches the referenced input UTXO and checks that its script_pubkey equals the configured bridge address. See [`core/lib/via_btc_client/src/indexer/mod.rs`](https://github.com/vianetwork/via-core/blob/main/core/lib/via_btc_client/src/indexer/mod.rs).
  - Note: In the "process_block" example above, the filter line is illustrative; the actual implementation performs async checks in a loop prior to collecting valid messages.

- OP_RETURN parsing offset fix
  - The start offset to read the proof reveal txid from OP_RETURN data was corrected to begin after the prefix and a single delimiter byte (+1). See [`core/lib/via_btc_client/src/indexer/parser.rs:775`](https://github.com/vianetwork/via-core/blob/main/core/lib/via_btc_client/src/indexer/parser.rs#L775).

- Removal of sequential priority-id tracking in the indexer ingestion path
  - The via_indexer L1→L2 processor no longer keeps a "next expected priority id" counter; deposits rely entirely on deterministic derivation from on-chain metadata. See [`via_indexer/node/indexer/src/message_processors/deposit.rs`](https://github.com/vianetwork/via-core/blob/main/via_indexer/node/indexer/src/message_processors/deposit.rs).

- Database schema note for withdrawals table
  - The indexer DAL dropped the UNIQUE constraint on bridge_withdrawals.block_number to allow multiple withdrawal bundles per block. See [`via_indexer/lib/via_indexer_dal/migrations/20250604191948_deposit_withdraw.up.sql`](https://github.com/vianetwork/via-core/blob/main/via_indexer/lib/via_indexer_dal/migrations/20250604191948_deposit_withdraw.up.sql).
