# Via System: Bottlenecks and Optimization Notes

All claims in this document are verified against the `via-core` source tree. Every
bottleneck cites the constant, config default, or code path that creates it. Ideas that
are not implemented in code are collected at the end and explicitly marked as
suggestions. This document contains no measured throughput or latency figures, because
none exist in the repository; any TPS or millisecond number you see elsewhere for Via
should be treated as unmeasured speculation.

## Summary

Via is a Bitcoin ZK rollup (zksync-era fork) that anchors batch commitments and proofs
to Bitcoin via inscriptions, posts pubdata to Celestia, and finalizes withdrawals with
an n-of-n MuSig2 verifier network. The hard bottlenecks, in rough order of severity:

1. **Bitcoin L1 anchoring throughput**: the BTC sender aggregates exactly one batch per
   commit inscription and allows one inscription in flight, so batch finality is paced
   by Bitcoin block time.
2. **n-of-n MuSig2 signing liveness**: every registered verifier must contribute a
   nonce and a partial signature; one offline verifier halts withdrawals. There is a
   single coordinator, and it processes one signing session at a time.
3. **Withdrawal batching limits**: at most 7 withdrawals per session, minimum 660 sats
   per withdrawal, transaction weight capped just under Bitcoin's standardness limit.
4. **Celestia DA limits**: blobs are capped at 1,973,786 bytes by config, and the
   dispatcher splits pubdata into 500 KiB chunks submitted per batch in sequence.
5. **L1 indexing lag**: the BTC watcher scans at most 50 Bitcoin blocks per poll tick
   (the standalone indexer uses 100), minus a confirmation depth.
6. **Fee rate clamping**: fee estimates are clamped to 1 to 100 sat/vB on mainnet, so
   during sustained fee spikes above 100 sat/vB inscriptions can lag confirmation.
7. **State keeper and DB limits**: 250 transaction slots per L1 batch, a 10,000,000
   entry mempool cache, and a Postgres pool of 50 connections per instance.
8. **Prover pipeline (inherited from zksync-era)**: GPU-based proof generation with a
   600 s proof timeout, a prover queue capacity of 10, setup keys loaded from disk, and
   a 3,600 s proof compressor timeout.

## 1. Bitcoin L1 Anchoring Throughput

Batch commitments and proofs reach Bitcoin as inscriptions produced by
`via_btc_sender`. The aggregation policy is deliberately conservative: one batch per
commit inscription, one proof per proof inscription, one inscription in flight.

```toml
# etc/env/base/via_btc_sender.toml
[via_btc_sender]
# The interval in milliseconds between btc sender cycles.
poll_interval = 5000
# The max aggregated commit batches to process in one inscription.
max_aggregated_blocks_to_commit = 1
# The max aggregated proof batches to process in one inscription.
max_aggregated_proofs_to_commit = 1
# The max number of inscriptions in flight.
max_txs_in_flight = 1
```

The config struct documents this as intentional:

```rust
// core/lib/config/src/configs/via_btc_sender.rs
pub struct ViaBtcSenderConfig {
    /// Service interval in milliseconds.
    pub poll_interval: u64,

    // Number of blocks to commit at time, should be 'one'.
    pub max_aggregated_blocks_to_commit: i32,

    // Number of proofs to commit at time, should be 'one'.
    pub max_aggregated_proofs_to_commit: i32,

    // The max number of inscription in flight
    pub max_txs_in_flight: i64,
    ...
}
```

The aggregator wires these limits into publish criteria; a batch is published when the
count limit is hit or a timestamp deadline passes:

```rust
// core/node/via_btc_sender/src/aggregator.rs
impl ViaAggregator {
    pub fn new(config: ViaBtcSenderConfig) -> Self {
        Self {
            commit_l1_block_criteria: vec![
                Box::from(ViaNumberCriterion {
                    limit: config.max_aggregated_blocks_to_commit as u32,
                }),
                Box::from(TimestampDeadlineCriterion {
                    deadline_seconds: config.block_time_to_commit(),
                }),
            ],
            commit_proof_criteria: vec![
                Box::from(ViaNumberCriterion {
                    limit: config.max_aggregated_proofs_to_commit as u32,
                }),
                Box::from(TimestampDeadlineCriterion {
                    deadline_seconds: config.block_time_to_proof(),
                }),
            ],
            config,
        }
    }
```

**Why it is a bottleneck**: with one batch per inscription and one inscription in
flight, the commit pipeline advances at most one batch per confirmed Bitcoin
transaction. Bitcoin averages one block per ten minutes, so batch anchoring (and
therefore proof finality on L1) is paced by Bitcoin, not by the sequencer. This is an
architectural constraint of the design, not a bug; raising the aggregation limits
(`max_aggregated_blocks_to_commit`) is the code-level knob if throughput ever needs to
grow, at the cost of larger inscriptions.

## 2. n-of-n MuSig2 Signing Liveness (Withdrawal Finalization)

Withdrawals are signed by the verifier network using MuSig2. The coordinator requires a
partial signature from every registered verifier: the number of required signers is the
size of the verifier public key set, with no threshold parameter.

```rust
// via_verifier/node/via_verifier_coordinator/src/coordinator/api_impl.rs
        return ok_json(SigningSessionResponse {
            session_op: session_op_bytes,
            required_signers: self_.state.verifiers_pub_keys.len(),
            received_nonces,
            received_partial_signatures,
            created_at: session.created_at,
        });
```

Each verifier (including the coordinator) polls and advances at most one signing
session per iteration:

```rust
// via_verifier/node/via_verifier_coordinator/src/verifier/mod.rs
    pub async fn run(mut self, mut stop_receiver: watch::Receiver<bool>) -> anyhow::Result<()> {
        let mut timer = tokio::time::interval(self.verifier_config.polling_interval());

        while !*stop_receiver.borrow_and_update() {
            tokio::select! {
                _ = timer.tick() => { /* continue iterations */ }
                _ = stop_receiver.changed() => break,
            }

            match self.loop_iteration().await {
                Ok(()) => {}
                Err(err) => {
                    METRICS.errors.inc();
                    tracing::error!("Failed to process verifier withdrawal task: {err}");
                }
            }
        }
```

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

            if session_info.session_op.is_empty() {
                tracing::debug!("Empty session, nothing to process");
                return Ok(());
            }
```

The default verifier poll interval is 10 seconds:

```toml
# etc/env/base/via_verifier.toml
[via_verifier]
# Interval between polling db for verification requests (in ms).
poll_interval = 10000
```

**Why it is a bottleneck**:
- **Liveness**: n-of-n means a single offline, stuck, or malicious verifier stops all
  withdrawal signing. There is no k-of-n fallback in the code.
- **Single coordinator**: only the node with the coordinator role creates sessions and
  aggregates signatures (`is_coordinator()` gates session creation), so the coordinator
  is a single point of failure for withdrawal processing.
- **Sequential sessions**: one session per loop iteration, with a 10 s poll interval,
  bounds how quickly successive withdrawal batches can be signed regardless of how many
  withdrawals are queued.

## 3. Withdrawal Batching Limits

The withdrawal session builder caps each Bitcoin transaction at 7 withdrawal outputs
and skips withdrawals below 660 sats:

```rust
// via_verifier/node/via_verifier_coordinator/src/sessions/withdrawal.rs
const OP_RETURN_WITHDRAW_PREFIX: &[u8] = b"VIA_WI";
const WITHDRAWAL_VERSION: WithdrawalVersion = WithdrawalVersion::Version0;
const WITHDRAWAL_LIMIT: u32 = 7;
```

```rust
// via_verifier/node/via_verifier_coordinator/src/sessions/withdrawal.rs
        // Set the minimum amount to withdraw + fee = 660 sats.
        let min_value = 660;

        let no_processed_withdrawals = storage
            .via_withdrawal_dal()
            .list_no_processed_withdrawals(min_value, WITHDRAWAL_LIMIT)
            .await?;
```

The transaction builder is configured with the same limit and a weight cap:

```rust
// via_verifier/node/via_verifier_coordinator/src/sessions/withdrawal.rs
            max_tx_weight: MAX_STANDARD_TX_WEIGHT as u64,
            max_output_per_tx: WITHDRAWAL_LIMIT as usize,
```

The verifier config default leaves headroom under Bitcoin's standardness weight limit
(400,000 weight units):

```rust
// core/lib/config/src/configs/via_verifier.rs
        // well within node acceptance limits
        self.max_tx_weight
            .unwrap_or((MAX_STANDARD_TX_WEIGHT - 20000).into())
```

**Why it is a bottleneck**: withdrawal throughput is at most 7 withdrawals per signing
session, and sessions are sequential (section 2). A withdrawal queue drains at 7 per
completed MuSig2 round. The 660 sat minimum also means dust-level withdrawals are never
processed and sit unprocessed in the queue by design.

## 4. Celestia Data Availability Limits

The Celestia blob size limit is a config value, defaulting to 1,973,786 bytes:

```toml
# etc/env/base/via_celestia.toml
[via_celestia_client]
# The da blob size limit.
blob_size_limit = 1973786
```

```rust
// core/lib/config/src/configs/via_celestia.rs (default)
            blob_size_limit: 1973786,
```

The DA dispatcher does not push a whole batch as one blob; it splits pubdata into
500 KiB chunks and dispatches them per batch, batch by batch:

```rust
// core/node/via_da_dispatcher/src/da_dispatcher.rs
/// The max blob size posted to DA layer.
const BLOB_CHUNK_SIZE: usize = 500 * 1024;
```

```rust
// core/node/via_da_dispatcher/src/da_dispatcher.rs
        for batch in batches {
            let chunks: Vec<Vec<u8>> = batch
                .pubdata
                .clone()
                .chunks(BLOB_CHUNK_SIZE)
                .map(|chunk| chunk.to_vec())
                .collect();

            self._dispatch_chunks(batch.l1_batch_number, chunks, false, "da_dispatcher")
                .await?;

            METRICS
                .last_dispatched_l1_batch
                .set(batch.l1_batch_number.0 as usize);
            METRICS.blob_size.observe(batch.pubdata.len());
        }
```

**Why it is a bottleneck**: batch pubdata size directly multiplies into the number of
sequential 500 KiB blob submissions per batch, each waiting on the Celestia light node.
Large batches therefore stretch DA dispatch latency linearly. The blob size limit also
caps how large a single chunk could ever be made. There is a fallback DA client
(`core/lib/via_da_clients/src/fallback/client.rs`) that defers to the primary client's
`blob_size_limit()`, so the limit holds across backends.

## 5. L1 Indexing Lag (BTC Watch)

The sequencer-side watcher scans Bitcoin in fixed chunks of 50 blocks per poll, and the
standalone indexer uses 100:

```rust
// core/lib/config/src/configs/via_btc_watch.rs
/// Total L1 blocks to process at a time.
pub const L1_BLOCKS_CHUNK: u32 = 50;
```

```rust
// via_indexer/node/indexer/src/lib.rs
/// Total L1 blocks to process at a time.
pub const L1_BLOCKS_CHUNK: u32 = 100;
```

The scan window is also clamped by a confirmation depth and by the reorg detector's
last validated block:

```rust
// core/node/via_btc_watch/src/lib.rs
        let mut to_block = last_processed_bitcoin_block + L1_BLOCKS_CHUNK;
        if to_block > current_l1_block_number {
            to_block = current_l1_block_number;
        }

        // Clamp the to_batch to the last valid block number validated by the reorg detector
        if to_block > last_l1_block_number as u32 {
            to_block = last_l1_block_number as u32;
        }
```

The default poll interval is 5 seconds:

```toml
# etc/env/base/via_btc_watch.toml
[via_btc_watch]
# The interval in milliseconds between btc sender cycles.
poll_interval = 5000
# The number of L1 block to mark the inscription as finalized.
block_confirmations = 0
```

**Why it is a bottleneck**: after downtime or on first sync, catch-up speed is at most
50 Bitcoin blocks per 5 s tick (100 for the indexer), and deposits are only seen once
their block is within the confirmed, reorg-validated window. During normal operation
this adds up to one poll interval plus `block_confirmations` blocks of latency to
deposit detection.

## 6. Bitcoin Fee Rate Clamping

Fee estimates from the Bitcoin node are clamped to a network-specific range, 1 to
100 sat/vB on mainnet:

```rust
// core/lib/via_btc_client/src/client/fee_limits.rs
/// Provides network-specific fee rate limits for Bitcoin transactions
///
/// These limits help protect against fee estimation spikes and drops, particularly on testnet
/// where fee estimates can be unreliable due to inconsistent mining patterns.
#[derive(Debug, Clone, Copy)]
pub struct FeeRateLimits {
    min_fee_rate: u64,
    max_fee_rate: u64,
}

impl FeeRateLimits {
    /// Create fee rate limits (in sat/vB) appropriate for the given Bitcoin network
    pub fn from_network(network: BitcoinNetwork) -> Self {
        match network {
            // Limits based on the data from https://dune.com/dataalways/bitcoin-fee-tracker
            BitcoinNetwork::Bitcoin => Self {
                min_fee_rate: 1,
                max_fee_rate: 100,
            },
```

**Why it is a bottleneck**: if mainnet fees stay above 100 sat/vB for a sustained
period, inscriptions and withdrawal transactions are fee-capped below the market rate
and may sit unconfirmed. The BTC sender has a stuck-inscription detector
(`stuck_inscription_block_number = 6` in `etc/env/base/via_btc_sender.toml`), but the
fee ceiling itself is a hard clamp in code.

## 7. State Keeper, Mempool, and Database Limits

Sequencer-side limits, all inherited zksync-era mechanisms with Via's configured
values:

```toml
# etc/env/base/chain.toml
# Denotes the amount of slots for transactions in the block.
transaction_slots = 250
```

```toml
# etc/env/base/chain.toml
[chain.mempool]
delay_interval = 100
sync_interval_ms = 10
sync_batch_size = 1000
capacity = 10_000_000
stuck_tx_timeout = 86400 # 1 day in seconds
remove_stuck_txs = true
```

```toml
# etc/env/base/database.toml
pool_size = 50
```

The slots limit is a batch seal criterion:

```rust
// core/lib/config/src/configs/chain.rs
    /// The max number of slots for txs in a block before it should be sealed by the slots sealer.
    pub transaction_slots: usize,
```

**Why it is a bottleneck**: 250 transaction slots per L1 batch bounds batch size by
count regardless of gas, and each batch then flows through the one-batch-per-inscription
Bitcoin pipeline (section 1). The 50-connection Postgres pool bounds concurrent DAL
work per instance; the in-memory mempool cache is capped at 10,000,000 entries. Note
that the file-based config (`etc/env/file_based/general.yaml`) uses
`transaction_slots: 8192`, so the effective limit depends on which config the
deployment loads.

## 8. Prover Pipeline (Inherited from zksync-era)

Via reuses the zksync-era FRI prover stack. The load-bearing configured limits:

```toml
# etc/env/base/fri_prover.toml
[fri_prover]
setup_data_path = "data/keys"
prometheus_port = 3315
max_attempts = 10
generation_timeout_in_secs = 600
setup_load_mode = "FromDisk"
specialized_group_id = 100
queue_capacity = 10
```

```toml
# etc/env/base/fri_proof_compressor.toml
generation_timeout_in_secs = 3600
```

The witness generator timeout is 900 s (`etc/env/base/fri_witness_generator.toml`).

**Why it is a bottleneck**: proof generation is the compute-heavy tail of the pipeline.
A single proof attempt may take up to 600 s before timing out (with up to
`max_attempts = 10` retries), the in-process queue holds 10 jobs, setup keys are loaded
from disk (`FromDisk`), and proof compression for the final wrapper may take up to an
hour before its timeout. The prover binaries are built for GPU execution (the repo
ships CUDA docker-compose runners such as `docker-compose-gpu-runner-cuda-12-0.yml`),
so prover throughput scales with GPU instances, not sequencer capacity.

Specific hardware sizing figures that previously appeared in this document (1 NVIDIA
GPU / 2 CPU / 4 GB RAM per prover, 1 TB NFS setup-key volume, 12 GB compressor memory,
100 GB Bitcoin and Celestia node disks, 8 GB Celestia memory, Bitcoin dbcache 75 MB and
maxmempool 150 MB) do not exist anywhere in `via-core` and have been removed. They
describe a particular deployment, not the system. Setup keys are large in practice, but
no size is pinned in this repository.

## Optimization Ideas (Suggestions Only, Not Implemented)

None of the following exists in `via-core` today. They are listed as plausible
directions, without cost-benefit scores or ROI timelines, because no measurements exist
to support such scores.

- **Raise BTC sender aggregation limits**: `max_aggregated_blocks_to_commit` and
  `max_aggregated_proofs_to_commit` are config knobs already plumbed through
  `ViaAggregator`; committing multiple batches per inscription would amortize Bitcoin
  block time across batches. Requires validating inscription size against Bitcoin
  standardness limits.
- **Threshold signing (k-of-n)**: replacing the n-of-n MuSig2 requirement (for example
  with FROST) would remove the single-verifier liveness failure mode. This is a
  protocol change, not a tuning change, and affects the bridge address derivation.
- **Coordinator failover**: electing a backup coordinator would remove the single point
  of failure in withdrawal session creation.
- **Parallel or larger withdrawal sessions**: raising `WITHDRAWAL_LIMIT` (bounded by
  `max_tx_weight`) or pipelining sessions would drain withdrawal queues faster.
- **Parallel Celestia chunk submission**: `_dispatch_chunks` currently submits blobs
  for a batch in sequence; concurrent submission (or pubdata compression before
  chunking) would cut DA dispatch latency for large batches.
- **Adaptive fee ceiling**: making `FeeRateLimits` responsive to sustained fee regimes
  would avoid stuck inscriptions during long fee spikes.
- **Prover horizontal scaling**: standard zksync-era practice; add GPU prover instances
  consuming from the same queue. Setup-key storage and distribution strategy is a
  deployment concern.

## Source Notes

Verified against `via-core` at the paths cited inline. Key anchors:
`core/node/via_btc_sender/src/{aggregator.rs,publish_criterion.rs}`,
`core/node/via_btc_watch/src/lib.rs`, `via_indexer/node/indexer/src/lib.rs`,
`core/node/via_da_dispatcher/src/da_dispatcher.rs`,
`core/lib/via_da_clients/src/celestia/client.rs`,
`via_verifier/node/via_verifier_coordinator/src/{verifier/mod.rs,coordinator/api_impl.rs,sessions/withdrawal.rs}`,
`core/lib/via_btc_client/src/client/fee_limits.rs`,
`core/lib/config/src/configs/{via_btc_sender.rs,via_btc_watch.rs,via_celestia.rs,via_verifier.rs,chain.rs}`,
and `etc/env/base/*.toml`.
