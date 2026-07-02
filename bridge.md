# Via L2 Bitcoin Bridge Documentation

## Overview

The Via L2 Bitcoin Bridge is a critical component of the Via L2 system, enabling the secure transfer of assets between the L1 (Bitcoin) blockchain and the L2 (Via) rollup. The bridge implements a two-way mechanism for:

1. **Deposits (L1→L2)**: Detecting Bitcoin transactions sent to the bridge address and crediting corresponding assets on L2.
2. **Withdrawals (L2→L1)**: Processing withdrawal requests initiated on L2 and executing the corresponding Bitcoin transactions.

The bridge leverages Bitcoin's native capabilities along with ZK proofs and MuSig2 multi-signature schemes to ensure security, decentralization, and efficiency.

## Key Features

The bridge system provides comprehensive functionality for secure and efficient cross-chain operations:

### Core Capabilities
- **Deposit validation**: receiver must be above the system-contract address space and the value must cover the fixed L2 transaction cost (`ViaL1Deposit::is_valid_deposit()`)
- **Deterministic deposit ordering**: priority ids derived from `(block, tx_index, vout)`, stable across replays
- **Per-input MuSig2 withdrawals**: n-of-n signing with weight-based transaction splitting and fee-rate tolerance checks
- **Idempotent reconciliation**: executed withdrawals are re-derived from the on-chain transaction by any node
- **Governance-driven wallet rotation**: bridge/sequencer/governance addresses live in `SystemWallets` and update at runtime via OP_RETURN governance flows

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

`extract_important_transactions(&block.txdata)` classifies each block's transactions into three groups using the `SystemWallets` addresses: `gov_txs` (paying the governance address), `system_txs` (witness inscriptions from known system addresses), and `bridge_txs` (touching the bridge address). Within `bridge_txs`:

1. A transaction **paying** the bridge address without a `VIA_WI` OP_RETURN is a deposit candidate, parsed by `parse_bridge_transaction` (inscription-based or OP_RETURN-based deposit)
2. A transaction **spending from** the bridge with a `VIA_WI` OP_RETURN is an executed withdrawal (`BridgeWithdrawal`); its origin is validated asynchronously by `is_valid_bridge_withdrawal()`, which fetches the referenced input UTXO and checks its script belongs to the bridge (`core/lib/via_btc_client/src/indexer/mod.rs`)

The indexer attaches `tx_index` and `output_vout` metadata (`TransactionWithMetadata`) to every parsed deposit; this feeds the deterministic priority id described at the end of this document.

### Deposit Processing

Deposits are processed by the `L1ToL2MessageProcessor` in `via_verifier/node/via_btc_watch/src/message_processors/l1_to_l2.rs`. This processor:

1. Extracts L1ToL2 messages from Bitcoin transactions
2. Creates L1 transactions for the L2 system
3. Inserts these transactions into the database for processing by the sequencer

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

Note what changed from earlier revisions: no `serial_id` parameter is passed in (the serial id comes from `deposit.priority_id()` inside the `From<ViaL1Deposit> for L1Tx` conversion), missing `tx_index`/`output_vout` metadata is a hard error, and invalid deposits yield `Ok(None)` rather than an error. The satoshi amount is scaled by `10_000_000_000` (8 to 18 decimals) into `to_mint`, and the sender and refund recipient are the L2 receiver.

### Deposit Types

The system supports two types of deposits:

1. **Inscription-based deposits**: Using Bitcoin inscriptions to include additional metadata
2. **OP_RETURN-based deposits**: Using OP_RETURN outputs to specify the L2 recipient address

### Deposit Validation and Processing

The deposit validation system implements comprehensive security measures and advanced processing logic:

#### The `ViaL1Deposit` type

Deposits are encapsulated by `ViaL1Deposit` (`core/lib/types/src/l1/via_l1.rs`):

```rust
#[derive(Debug, Clone)]
pub struct ViaL1Deposit {
    pub l2_receiver_address: Address,
    pub amount: u64,
    pub calldata: Vec<u8>,
    pub l1_block_number: u64,
    pub tx_index: usize,
    pub output_vout: usize,
}
```

Note there is no serial-id field: the priority id is **derived** from `(l1_block_number, tx_index, output_vout)` by `priority_id()` (see the Deterministic PriorityOpId section below).

#### Validation

```rust
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
}
```

Two real checks, not a validation-status machine:
- The receiver must be **above the system-contract address space** (`MAX_SYSTEM_CONTRACT_ADDRESS`), so deposits cannot mint into kernel addresses
- The deposited value (satoshis scaled to 18 decimals) must **cover the fixed L2 transaction cost** (`GAS_LIMIT * MAX_FEE_PER_GAS`); dust deposits that could not pay for their own L2 inclusion are dropped

Invalid deposits simply yield `None` from `l1_tx()`; the message processor logs and skips them. The L1→L2 transaction is built by the `From<ViaL1Deposit> for L1Tx` conversion with the fixed gas parameters, and for plain bridging deposits the calldata is empty (the whole amount is minted to the receiver).

## Withdrawal Flow (L2→L1)

### Withdrawal Initiation

Withdrawals are initiated on L2 through the L2 bridge contract. The withdrawal request contains:
- The Bitcoin recipient address
- The amount to withdraw

The L2 contract emits a log that is included in the L2 batch data, which is then published to the data availability layer.

### Withdrawal Processing

For fee distribution semantics and empty-transaction behavior details, see [withdrawal_finalization.md](withdrawal_finalization.md).

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

2. **Session Timeout Management**: The coordinator implements session timeout functionality to manage concurrent sessions and prevent deadlocks:

```rust
pub async fn is_session_timeout(&self) -> bool {
    let created_at = self.state.signing_session.read().await.created_at.clone();
    created_at + self.state.session_timeout < seconds_since_epoch()
}
```

**Session Timeout Features:**
- **Configurable Timeout**: Session timeout is configurable via `session_timeout` parameter (default: 30 seconds)
- **Automatic Cleanup**: Sessions that exceed the timeout are automatically cleaned up
- **Concurrent Session Management**: Prevents new sessions from starting while active sessions are in progress
- **Deadlock Prevention**: Ensures that stuck sessions don't block the system indefinitely

**Configuration:**
```toml
# etc/env/base/via_verifier.toml
session_timeout = 30  # Session timeout in seconds
```

3. **Withdrawal Session Creation**: The `WithdrawalSession` (`via_verifier/node/via_verifier_coordinator/src/sessions/withdrawal.rs`) pulls unprocessed withdrawals directly from the withdrawal DAL and builds the unsigned transaction in one step:

```rust
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

4. **Transaction Building**: `build_transaction_with_op_return(outputs, config)` returns `Vec<UnsignedBridgeTx>`, splitting by weight and output count:

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

Full internals (chunking, fee distribution, `UnsignedBridgeTx` fields) in `musig2_implementation.md`.

5. **Multi-Signature Coordination**: The `ViaWithdrawalVerifier` coordinates one MuSig2 session **per transaction input** (`signer_per_utxo_input`); aggregation fills `final_sig_per_utxo_input`, and each input gets its own aggregated Schnorr signature in its witness:

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

The full nonce/partial-signature round code is in `withdrawal_finalization.md` and `musig2_implementation.md`.

6. **Transaction Broadcasting**: Once every verifier has contributed partial signatures for all inputs, the coordinator broadcasts the transaction to the Bitcoin network and marks the paid withdrawals processed in the same database transaction (`after_broadcast_final_transaction`, verbatim in `withdrawal_finalization.md`).

### Withdrawal Security

Withdrawals are secured through several mechanisms:

1. **ZK Proofs**: Each L2 batch is verified with a ZK proof, ensuring the validity of all operations including withdrawals.
2. **MuSig2 Multi-Signatures**: Withdrawals require signatures from **every** verifier (n-of-n; `required_signers = verifiers_pub_keys.len()`), not a threshold.
3. **Verifier Voting**: Batches finalize at 2/3 of attestations (`BATCH_FINALIZATION_THRESHOLD = 0.66`) before their withdrawals become signable.
4. **Independent Re-verification**: Every verifier recomputes the per-input sighashes and enforces the ±1 sat/vB fee-rate tolerance before contributing a partial signature.

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

## Bitcoin Client Layer

There is no multi-client manager or connection pool; the node registers a single `BtcClientLayer` providing the Bitcoin client resource to all components:

```rust
// via_verifier/bin/verifier_server/src/node_builder.rs
fn add_btc_client_layer(mut self) -> anyhow::Result<Self> {
    self.node.add_layer(BtcClientLayer::new(
        self.configs.via_btc_client_config.clone(),
        self.configs.secrets.via_l1.clone().unwrap(),
        self.configs.wallets.clone(),
        Some(self.configs.via_bridge_config.bridge_address.clone()),
    ));
    Ok(self)
}
```

Role separation happens through the wallet set (`SystemWallets`: sequencer, governance, bridge, verifiers), not through separate client instances.

## Bootstrapping and Wallets

There is no dedicated testnet JSON bundle or `TestnetBridgeConfig`; bootstrapping is uniform across networks. The wallet set is a single shared type:

```rust
// core/lib/types/src/via_wallet.rs
#[derive(Debug, Clone, PartialEq)]
pub struct SystemWallets {
    pub sequencer: Address,
    pub verifiers: Vec<Address>,
    pub governance: Address,
    pub bridge: Address,
}
```

The flow:

- The genesis configuration provides `bootstrap_txids` (the Bitcoin transactions carrying the `SystemBootstrapping` inscription and related setup)
- At storage initialization, `ViaBootstrap::process_bootstrap_messages()` (`core/lib/via_btc_client/src/bootstrap/mod.rs`) scans those transactions, constructs the `SystemWallets`, and persists them in the `via_wallets` table
- All downstream components (indexer, watchers, API) read the active addresses from the database at runtime; wallet rotations via governance flow through `BitcoinInscriptionIndexer::update_system_wallets()` so parsing follows the new addresses without config changes
- The BTC sender validates that its active inscriber address equals `SystemWallets.sequencer` before emitting inscriptions

UTXO handling for the bridge is the `UtxoManager` documented in `utxo_management.md` (pending-transaction context over the node's UTXO set, largest-first selection); there is no separate wallet-RPC UTXO manager.

### Bridge Withdrawal Validation and Indexing

This release adds a full validation and indexing path for bridge withdrawals on Bitcoin. The system validates that withdrawal transactions actually originate from the bridge address, processes withdrawal inscriptions, and persists them with detailed metadata.

- Withdrawal origin validation
  - The indexer validates that the first input being spent belongs to the bridge wallet by matching the script_pubkey:
    - [BitcoinInscriptionIndexer::is_valid_bridge_withdrawal()](core/lib/via_btc_client/src/indexer/mod.rs:376)
  - The standalone L1 indexer processor also enforces this during withdrawal message processing by verifying each input's script_pubkey against the bridge:
    - Script pubkey check: [via_indexer/node/indexer/src/message_processors/withdrawal.rs](via_indexer/node/indexer/src/message_processors/withdrawal.rs:63)

- Withdrawal message processor (Indexer layer)
  - The indexer processes BridgeWithdrawal inscriptions and extracts transaction metadata:
    - Main loop: [WithdrawalProcessor::process_messages()](via_indexer/node/indexer/src/message_processors/withdrawal.rs:31)
    - Bridge tx id bytes handling (LE/BE): [via_indexer/node/indexer/src/message_processors/withdrawal.rs](via_indexer/node/indexer/src/message_processors/withdrawal.rs:41)
    - Proof reveal tx id bytes handling (LE/BE): [via_indexer/node/indexer/src/message_processors/withdrawal.rs](via_indexer/node/indexer/src/message_processors/withdrawal.rs:74)
  - It computes the transaction fee as total input value minus output amount and captures vsize/total_size:
    - Fee computation and sizing capture: [via_indexer/node/indexer/src/message_processors/withdrawal.rs](via_indexer/node/indexer/src/message_processors/withdrawal.rs:72)
  - Each per-recipient withdrawal is normalized into indexed entries:
    - WithdrawalParam creation: [via_indexer/node/indexer/src/message_processors/withdrawal.rs](via_indexer/node/indexer/src/message_processors/withdrawal.rs:89)

- DAL helpers and database persistence
  - Data models for bridge withdrawals:
    - [BridgeWithdrawalParam](via_indexer/lib/via_indexer_dal/src/models/withdraw.rs)
    - [WithdrawalParam](via_indexer/lib/via_indexer_dal/src/models/withdraw.rs)
  - The OP_RETURN layout is `VIA_WI ++ version byte ++ withdrawals metadata` (no `index_withdrawal` field; withdrawal attribution comes from per-output withdrawal ids). Canonical layout in [inscription_interaction.md](inscription_interaction.md).
  - The DAL exposes a single insertion API that persists the bridge withdrawal row along with all constituent withdrawals:
    - [ViaTransactionsDal::insert_withdraw()](https://github.com/vianetwork/via-core/blob/main/via_indexer/lib/via_indexer_dal/src/via_transactions_dal.rs#L113)
  - Persisted fields include:
    - Bridge withdrawal: tx_id, l1_batch_reveal_tx_id, block_number, fee, vsize, total_size, withdrawals_count
      - Insert SQL: [via_indexer/lib/via_indexer_dal/src/via_transactions_dal.rs](https://github.com/vianetwork/via-core/blob/main/via_indexer/lib/via_indexer_dal/src/via_transactions_dal.rs#L126)
    - Individual withdrawals: tx_index, receiver, value
      - Insert SQL: [via_indexer/lib/via_indexer_dal/src/via_transactions_dal.rs](https://github.com/vianetwork/via-core/blob/main/via_indexer/lib/via_indexer_dal/src/via_transactions_dal.rs#L147)
  - The processor finalizes by inserting the fully-formed withdrawal into the DAL:
    - [via_indexer/node/indexer/src/message_processors/withdrawal.rs](https://github.com/vianetwork/via-core/blob/main/via_indexer/node/indexer/src/message_processors/withdrawal.rs#L105)

- Where this integrates in the overall withdrawal flow
  - After L1 detection and validation, the coordinator builds unsigned transactions and coordinates MuSig2 signing as documented above.
  - For fee logic, see the new Withdrawal Fee Strategy section in the fee documentation:
    - [fee_mechanism.md](fee_mechanism.md)

## Bridge configuration separation and wallets bootstrap

- Bridge parameters (coordinator key, verifier keys, bridge address, thresholds) are provided by a dedicated bridge configuration (via_bridge.toml or VIA_BRIDGE_*), not by genesis.
- On startup, nodes derive and persist SystemWallets by scanning bootstrap transactions; consumers (indexer, watchers) use these addresses for validation and parsing.
- The active bridge address is retrieved from the database at runtime by the API and watchers.
- Verifier configuration optionally accepts a bridge address merkle root to validate Taproot tree membership when required.

System wallet update flows

- Updates to sequencer, governance and bridge follow the same governance execution pattern used for protocol upgrades:
  - Proposals are witness-based inscriptions (for bridge updates: UpdateBridgeProposal with new MuSig2 address and verifier set).
  - Executions are OP_RETURN transactions signed from governance, with ASCII prefixes:
    - "VIA_PROTOCOL:SEQ" (sequencer), "VIA_PROTOCOL:GOV" (governance) and "VIA_PROTOCOL:BRI" (bridge), plus the appropriate payload (address or proposal txid).
- The watcher persists the new SystemWallets and immediately applies them to subsequent parsing (e.g., deposit detection uses the new bridge address).

Transferring UTXOs from the old bridge

- Governance-controlled script path allows consolidating or relocating UTXOs from the old bridge address to a governance wallet.
- Example utilities:
  - Compute aggregate keys / governance script: [rust compute_musig2.rs](via_verifier/lib/via_musig2/examples/compute_musig2.rs)
  - Transfer UTXOs via governance script path: [rust transfer_utxos_from_bridge.rs](via_verifier/lib/via_musig2/examples/transfer_utxos_from_bridge.rs)
- Operator runbook and examples are provided in the upgrade guide under "Transfer the UTXOs from the old bridge address".

## Database Tracking

There is no `bridge_operations` table or `BridgeTransactionTracker`. Bridge state is tracked in two places:

**Verifier side** (`via_verifier/lib/verifier_dal/src/withdrawals_dal.rs`): withdrawals parsed from finalized batches are inserted individually and marked processed once a bridge transaction pays them. The DAL surface:

```rust
pub async fn insert_l1_batch_bridge_withdrawals(/* ... */)
pub async fn insert_bridge_withdrawal_tx(&mut self, tx_id: &[u8]) -> DalResult<i64>
pub async fn mark_bridge_withdrawal_tx_as_processed(&mut self, tx_id: &[u8]) -> DalResult<()>
pub async fn bridge_withdrawal_exists(&mut self, tx_id: &[u8]) -> DalResult<bool>
pub async fn get_bridge_withdrawal_id(&mut self, tx_id: &[u8]) -> DalResult<Option<i64>>
pub async fn insert_withdrawals(/* ... */)
pub async fn mark_withdrawals_as_processed(/* ... */)
pub async fn mark_withdrawal_as_processed(/* ... */)
pub async fn check_if_withdrawal_exists_unprocessed(/* ... */)
pub async fn check_if_withdrawal_exists(/* ... */)
```

with the row model:

```rust
// via_verifier/lib/verifier_dal/src/models/withdrawal.rs
pub struct Withdrawal {
    pub l2_tx_hash: String,
    pub l2_tx_index: i64,
    pub receiver: String,
    pub value: i64,
}
```

The withdrawal session reads `list_no_processed_withdrawals(min_value, limit)` from `via_withdrawal_dal`; post-broadcast reconciliation uses the insert/mark methods above (verbatim flows in `withdrawal_finalization.md`).

**Indexer side** (`via_indexer/lib/via_indexer_dal/src/via_transactions_dal.rs`): the standalone L1 indexer persists executed bridge withdrawals via `insert_withdraw(withdrawal: Withdrawal)`, keyed by the L2 withdrawal id, for external API consumption (schema in `l1_indexer.md`).

## Configuration Examples and Best Practices

### The Real Bridge Configuration

The dedicated bridge config is intentionally small (`core/lib/config/src/configs/via_bridge.rs`):

```rust
pub struct ViaBridgeConfig {
    /// The verifiers public keys.
    pub verifiers_pub_keys: Vec<String>,

    /// The bridge address.
    pub bridge_address: String,
}
```

Everything else lives in the configs that own it:
- Bitcoin RPC endpoint, fee APIs: `via_btc_client` (four fields; see `fee_mechanism.md`)
- Confirmations, poll interval, start block: `via_btc_watch`
- Session/weight parameters, `bridge_address_merkle_root`: `via_verifier`
- Deposit validity: code constants in `ViaL1Deposit::is_valid_deposit()` (system-address floor, gas-fee coverage), not config
- Withdrawal minimum (660 sats) and `WITHDRAWAL_LIMIT`: constants in the withdrawal session

The active bridge address is read from the database (`via_wallets`) at runtime; the config value seeds wiring, and governance-driven bridge migrations update the database.

## Operational Guidelines

### Bridge Deployment Process

There is no `via-bridge` CLI. The real tooling:

1. **Key generation and bridge address derivation**: the runnable examples in `via_verifier/lib/via_musig2/examples/` (`key_generation_setup.rs` to generate keypairs, `compute_musig2.rs` to derive the aggregated MuSig2 bridge address, optionally with the governance script path)
2. **Bootstrap**: inscribe the `SystemBootstrapping` message (declaring verifier keys, bridge address, governance and sequencer addresses) and set its txid in the genesis config's `bootstrap_txids`; nodes derive and persist `SystemWallets` from it at storage init
3. **Monitoring**: metrics come from the built-in vise metrics (Prometheus exporter layer) and the healthcheck endpoints; there is no separate monitor process

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

The real exported signals are the vise metrics: the BTC watcher's `deposit` counter, `withdrawal_confirmed` counter, and `inscriptions_processed` gauges; the coordinator's `session_time` gauge and error counters; and the reorg/reverter module metrics. Watch the pending UTXO context depth and `Insufficient funds` selection errors for bridge-liquidity pressure (see `utxo_management.md`).

## Troubleshooting Common Bridge Issues

### Deposit Issues

**Deposits not being detected:**
1. Verify the Bitcoin node is synced and the RPC reachable
2. Check the active bridge address in the `via_wallets` table matches what users deposit to (a governance bridge migration changes it)
3. Check the watcher's last-processed block in the indexer table against the Bitcoin tip; the watcher pauses entirely while a reorg marker is present
4. Confirm the deposit passes `ViaL1Deposit::is_valid_deposit()`: receiver above the system address space, and value covering the fixed L2 gas cost

### Withdrawal Issues

**Withdrawals not being signed:**
1. All verifiers must be online (n-of-n); check each verifier can reach `coordinator_http_url`
2. Verifier public key sets must be identical (and identically ordered) across nodes
3. `bridge_address_merkle_root` must match across nodes; a mismatch invalidates every partial signature
4. A stuck session clears after `session_timeout`; check the coordinator's session endpoint and `verify_partial_signature` rejections in logs
5. Verifiers refuse to sign during reorg/sync; check the gating state

**High transaction fees / stuck withdrawals:**
- The fee rate is re-checked at signing time (±1 sat/vB); a fast-moving mempool can reject sessions until rates stabilize
- Withdrawals below 660 sats are filtered at the session level and stay pending
- Consolidate bridge UTXOs (`get_utxos_to_merge`) during low-fee periods

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

- **Bridge Address**: The Bitcoin address of the bridge (MuSig2 aggregated key; active value read from `via_wallets` at runtime)
- **Verifier Public Keys**: Public keys of the verifiers participating in the MuSig2 scheme (`ViaBridgeConfig.verifiers_pub_keys`)
- **Confirmation Threshold**: Number of Bitcoin confirmations before the watcher processes messages (`via_btc_watch`)
- **Agreement Threshold**: Batch finalization requires 2/3 of attestations (shared constant, not config); withdrawals require all verifiers (n-of-n)
- **Poll Interval**: How frequently to check for new blocks and transactions (`via_btc_watch`)

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
### Withdrawal Tracking: Current Data Model

An earlier revision tracked withdrawal sessions through a `ViaBridgeDal` keyed by votable transaction and candidate index (its `bridge_tx` migration, `20250714093607_bridge_tx.up.sql`, still exists in the migrations directory). The current design tracks **individual withdrawals** instead, in `withdrawals_dal.rs`:

- Withdrawals from finalized batches are inserted individually (keyed by their withdrawal id: L2 tx hash + log index provenance)
- A session selects up to `WITHDRAWAL_LIMIT` unprocessed withdrawals of at least 660 sats; whatever a broadcast transaction pays gets marked processed against the recorded bridge withdrawal txid
- Reconciliation is idempotent and driven by the on-chain transaction itself (both the coordinator's `after_broadcast_final_transaction` and any verifier's BTC-watch `WithdrawalProcessor` converge to the same state)
- No no-op records are needed: a batch with no withdrawals simply contributes nothing to the pending set

Cross-reference: fee-aware construction and validation (fee-rate tolerance, output filtering) in [fee_mechanism.md](fee_mechanism.md); verbatim flows in [withdrawal_finalization.md](withdrawal_finalization.md).

---

## Deterministic PriorityOpId (deposits) - updated encoding

- Deposits now derive a deterministic PriorityOpId from Bitcoin L1 metadata: `(block_number, tx_index, output_vout)` provided by the indexer. This removes the need for any sequential counters in watchers or indexer ingestion paths.
- Encoding and type:
  - Bit distribution updated to 28 bits for block_number, 20 bits for tx_index, and 16 bits for vout, encapsulated by [`rust ViaPriorityOpId::new()`](https://github.com/vianetwork/via-core/blob/main/core/lib/types/src/l1/priority_id.rs#L60).
  - The deposit serial id is produced via [`rust ViaL1Deposit::priority_id()`](https://github.com/vianetwork/via-core/blob/main/core/lib/types/src/l1/via_l1.rs#L269), which wraps the raw u64 from the dedicated type.
  - This supersedes the earlier 28/24/12 layout; ordering semantics remain by u64 comparison for stable replay.
- Indexer requirements:
  - The indexer must attach `tx_index` and `output_vout` when parsing deposits so the bridge can derive the id deterministically. See transport metadata plumbed via `TransactionWithMetadata` in [core/lib/via_btc_client/src/indexer/](https://github.com/vianetwork/via-core/blob/main/core/lib/via_btc_client/src/indexer/).
- Operational implications:
  - Stable, replayable ordering of deposits across restarts/reindexing.
  - Ingestion paths no longer maintain "next expected priority id" state; the via_indexer deposit processor was simplified accordingly (see [via_indexer/node/indexer/src/message_processors/deposit.rs](https://github.com/vianetwork/via-core/blob/main/via_indexer/node/indexer/src/message_processors/deposit.rs)).
