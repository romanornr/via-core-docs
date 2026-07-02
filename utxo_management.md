# UTXO Management

Via manages Bitcoin UTXOs in two independent places, and they solve different problems:

1. **Bridge side** (`UtxoManager`, `via_verifier/lib/via_musig2/src/utxo_manager.rs`): selects and tracks the bridge address's UTXOs for MuSig2 withdrawal transactions.
2. **Inscriber side** (`core/lib/via_btc_client/src/inscriber/mod.rs`): manages the sequencer/verifier wallet's P2WPKH UTXOs that fund commit/reveal inscription pairs.

> Historical note: an earlier auto-generated version of this document described a `UtxoSelector` with multiple selection strategies (LargestFirst, BranchAndBound, etc.), a `UtxoConsolidator`, a `FeeEstimator`, and a `via_utxo_management.toml` config. None of that exists in the codebase. Everything below is verified against source.

## Bridge side: `UtxoManager`

```rust
// via_verifier/lib/via_musig2/src/utxo_manager.rs
const DEFAULT_CAPACITY: usize = 100;

#[derive(Debug, Clone)]
pub struct UtxoManager {
    /// Btc client
    btc_client: Arc<dyn BitcoinOps>,
    /// The transactions executed by the wallet
    context: Arc<RwLock<VecDeque<Transaction>>>,
    /// The minimum amount to merge utxos
    minimum_amount: Amount,
    /// The maximum number of utxos to merge in a single tx
    merge_limit: usize,
}
```

The `context` deque is the key idea: it holds this wallet's **broadcast-but-unconfirmed transactions**, so UTXO selection can account for pending spends and pending change without waiting for confirmations.

### Methods

| Method | What it does |
|--------|-------------|
| `new(btc_client, minimum_amount, merge_limit)` | Construct with an empty in-memory context |
| `get_available_utxos(address)` | Fetch confirmed UTXOs from the node, then replay the context: add pending change outputs paying `address`, remove UTXOs spent by pending txs. Result sorted by value descending |
| `select_utxos_by_target_value(utxos, target)` | Walk the (already sorted, largest-first) list, accumulating until `total >= target`; errors on overflow or insufficient funds |
| `get_utxos_to_merge(bridge_address)` | Collect up to `merge_limit` UTXOs whose value is at least `minimum_amount`; returns them only if more than one (consolidation candidates) |
| `sync_context_with_blockchain()` | Pop confirmed transactions off the front of the context deque (stops at the first unconfirmed one) |
| `insert_transaction(tx)` | Push a newly broadcast tx onto the context (deduplicated by txid) |
| `get_btc_client()` | Access the underlying client |

### The pending-transaction overlay: `get_available_utxos`

```rust
pub async fn get_available_utxos(
    &self,
    address: Address,
) -> anyhow::Result<Vec<(OutPoint, TxOut)>> {
    // fetch utxos from client
    let mut utxos = self.btc_client.fetch_utxos(&address).await?;
    let context = self.context.read().await;

    {
        if context.is_empty() {
            // Sort UTXOs by value in descending order (big to small)
            utxos.sort_by(|a, b| b.1.value.cmp(&a.1.value));
            return Ok(utxos);
        }
    }

    for tx in context.iter() {
        // Add the output utxos to the list
        for (i, out) in tx.output.iter().enumerate() {
            if out.script_pubkey == address.script_pubkey() {
                let outpoint = OutPoint {
                    txid: tx.compute_txid(),
                    vout: i as u32,
                };
                utxos.push((outpoint, out.clone()));
            }
        }

        // Remove the inputs used utxos
        for input in &tx.input {
            if let Some(index) = utxos
                .iter()
                .position(|(op, _)| op == &input.previous_output)
            {
                utxos.remove(index);
            }
        }
    }

    // Sort UTXOs by value in descending order (big to small)
    utxos.sort_by(|a, b| b.1.value.cmp(&a.1.value));

    Ok(utxos)
}
```

### Selection strategy

There is exactly **one** selection strategy: largest-first accumulation. Sorting happens in `get_available_utxos()` (descending by value); `select_utxos_by_target_value()` just accumulates in order:

```rust
pub async fn select_utxos_by_target_value(
    &self,
    utxos: &[(OutPoint, TxOut)],
    target_amount: Amount,
) -> anyhow::Result<Vec<(OutPoint, TxOut)>> {
    // Simple implementation - could be improved with better UTXO selection algorithm
    let mut selected = Vec::new();
    let mut total = Amount::ZERO;

    for utxo in utxos {
        selected.push(utxo.clone());
        total = total
            .checked_add(utxo.1.value)
            .ok_or_else(|| anyhow::anyhow!("Amount overflow during UTXO selection"))?;

        if total >= target_amount {
            break;
        }
    }

    if total < target_amount {
        return Err(anyhow::anyhow!(
            "Insufficient funds: have {}, need {}",
            total, target_amount
        ));
    }

    Ok(selected)
}
```

### Consolidation

Consolidation is not a background job with thresholds; it is just `get_utxos_to_merge()`, which the withdrawal flow can use to sweep up to `merge_limit` bridge UTXOs of at least `minimum_amount` into one transaction. With fewer than two candidates it returns nothing:

```rust
pub async fn get_utxos_to_merge(
    &self,
    bridge_address: Address,
) -> anyhow::Result<Vec<(OutPoint, TxOut)>> {
    let mut utxos_to_merge = Vec::new();
    let available_utxos = self.get_available_utxos(bridge_address).await?;

    // If the amount is greater than the minimum amount to merge
    for (outpoint, txout) in available_utxos.iter() {
        if txout.value >= self.minimum_amount {
            utxos_to_merge.push((*outpoint, txout.clone()));
        }
        if utxos_to_merge.len() == self.merge_limit {
            break;
        }
    }
    if utxos_to_merge.len() > 1 {
        return Ok(utxos_to_merge);
    }
    Ok(vec![])
}
```

### Context maintenance

```rust
pub async fn sync_context_with_blockchain(&self) -> anyhow::Result<()> {
    loop {
        let tx = {
            match self.context.read().await.front() {
                Some(tx) => tx.clone(),
                None => break,
            }
        };

        let txid = tx.compute_txid();
        let is_confirmed = self
            .btc_client
            .check_tx_confirmation(&txid, CTX_REQUIRED_CONFIRMATIONS)
            .await?;

        if is_confirmed {
            self.context.write().await.pop_front();
        } else {
            break;
        }
    }
    Ok(())
}

pub async fn insert_transaction(&self, tx: Transaction) {
    for ctx_tx in self.context.read().await.iter() {
        if ctx_tx.compute_txid() == tx.compute_txid() {
            return;
        }
    }
    self.context.write().await.push_back(tx);
}
```

Note the deque discipline in `sync_context_with_blockchain`: it only ever pops from the **front** and stops at the first unconfirmed transaction. Because pending transactions can chain (a later tx spends an earlier tx's change), confirming them out of order is not meaningful; the FIFO order mirrors the dependency order.

### Chained spending safety

The withdrawal `TransactionBuilder` uses `UtxoManager` like this:

1. `sync_context_with_blockchain()` to drop confirmed txs from the context
2. `get_available_utxos(bridge_address)` for the effective spendable set (confirmed + pending change, minus pending spends)
3. `select_utxos_by_target_value(...)` for the outputs plus fee
4. After building, `insert_transaction(unsigned_tx)` so the next session sees this spend as pending

This is what lets multiple withdrawal transactions chain safely before earlier ones confirm, without double-spending bridge UTXOs.

## Inscriber side: wallet UTXOs for inscriptions

The Inscriber (`core/lib/via_btc_client/src/inscriber/mod.rs`) manages the operator wallet's P2WPKH UTXOs that pay for commit/reveal pairs:

- `sync_context_with_blockchain()` refreshes the wallet's UTXO view before building; the caller is expected to invoke it before each inscription
- `prepare_commit_tx_input()` selects wallet UTXOs, and when earlier inscriptions are still in flight it spends the **newest pending reveal transaction's change output**, forming a FIFO chain of unconfirmed inscription transactions
- Fee handling shifts 20% of the estimated commit fee onto the reveal transaction and adds incentives (see the fee constants in `inscriber/fee.rs`)

So both sides use the same pattern: an in-memory view of pending transactions layered over the node's confirmed UTXO set, enabling safe chaining of unconfirmed spends.

## Monitoring notes

Useful signals when operating either side:

- Growth of the pending context (bridge) or in-flight inscription queue (sequencer) indicates confirmation lag
- `Insufficient funds` errors from `select_utxos_by_target_value` mean the bridge balance minus pending spends cannot cover the withdrawal set
- Consolidation frequency is governed by `minimum_amount` and `merge_limit` passed to `UtxoManager::new`
