# Via L2 Bitcoin ZK-Rollup: Loadnext Tool Documentation

## 1. Introduction

Via ships two load-testing crates, both inherited from the ZKsync Era `loadnext` utility:

| Crate | Path | Binary | State |
|---|---|---|---|
| `loadnext` | `core/tests/loadnext` | `loadnext` | Unmodified ZKsync Era load tester. Its L1 side talks to an Ethereum web3 RPC (`L1_RPC_ADDRESS`, default `http://127.0.0.1:8545`) and uses ERC-20 minting and Ethereum deposits. Via has no Ethereum L1, so the L1-dependent paths (deposits, L1 execute, final withdrawal) cannot work against a Via deployment. Only the pure L2 transaction paths are meaningful. |
| `via_loadnext` | `core/tests/via_loadnext` | `via_loadnext` | Via-adapted fork. The Ethereum L1 side is replaced with a Bitcoin regtest client (`via_btc_client`): deposits are real Bitcoin transactions to the bridge address, and withdrawals target Bitcoin addresses. |

The rest of this document describes `via_loadnext`, the Via-specific fork. A grep for `btc|Bitcoin` in `core/tests/loadnext/src` returns nothing; all Bitcoin integration lives in `core/tests/via_loadnext`.

The `via` CLI wires it up as `via run loadtest`:

```typescript
// infrastructure/via/src/run.ts
export async function loadtest(...args: string[]) {
    console.log(args);
    await utils.spawn(`cargo run --release --bin via_loadnext -- ${args.join(' ')}`);
}
```

## 2. Purpose

Inherited from upstream loadnext (see `core/tests/via_loadnext/README.md`, which is still the ZKsync README):

- Simulates many independent accounts sending quasi-random requests to the server.
- Sends both correct and deliberately corrupted transactions and compares outcomes to expectations.
- Reports average TPS at the end of the run.
- Marks the test as failed rather than crashing if the server becomes unresponsive.

## 3. Account Model

Each test account has an Ethereum-style L2 identity plus a Bitcoin identity. Both are generated from the loadtest RNG; the Bitcoin side is hardcoded to regtest:

```rust
// core/tests/via_loadnext/src/account_pool.rs
#[derive(Debug, Clone)]
pub struct BtcAccount {
    pub btc_private_key: BitcoinPrivateKey,
    pub btc_address: BitcoinAddress,
}

/// Pool of accounts to be used in the test.
/// Each account is represented as `zksync::Wallet` in order to provide convenient interface of interaction with ZKsync.
#[derive(Debug)]
pub struct AccountPool {
    /// Main wallet that will be used to initialize all the test wallets.
    pub eth_master_wallet: SyncWallet,
    /// Main wallet that will be used to initialize all the test wallets.
    pub btc_master_wallet: BtcAccount,
    /// Collection of test wallets and their Ethereum private keys.
    pub eth_accounts: VecDeque<TestWallet>,
    /// Collection of test wallets and their Bitcoin private keys.
    pub btc_accounts: VecDeque<BtcAccount>,
    /// Pool of addresses of the test accounts.
    pub addresses: AddressPool,
}
```

```rust
// core/tests/via_loadnext/src/account_pool.rs
pub struct TestWallet {
    /// Pre-initialized wallet object.
    pub wallet: SyncWallet,
    /// Wallet with corrupted signer.
    pub corrupted_wallet: CorruptedSyncWallet,
    /// Contract bytecode and calldata to be used for sending `Execute` transactions.
    pub test_contract: &'static TestContract,
    /// Address of the deployed contract to be used for sending
    /// `Execute` transaction.
    pub deployed_contract_address: Arc<OnceCell<Address>>,
    /// RNG object derived from a common loadtest seed and the wallet private key.
    pub rng: LoadtestRng,
}
```

`AddressPool` holds both address kinds and hands out random targets (`random_evm_address`, `random_btc_address`). Other inherited components: `AccountLifespan` (per-account command loop), `Executor` (orchestration), `RequestLimiters` (semaphores for API requests and PubSub subscriptions), `ReportBuilder`/`ReportCollector` (results), and the vise-based `LOADTEST_METRICS`.

## 4. Command Model

### 4.1 Transaction commands

The Via fork trims the upstream `TxType` (which had `Deposit, WithdrawToSelf, WithdrawToOther, DeployContract, L1Execute, L2Execute`) down to four variants, three of which are weighted for random selection:

```rust
// core/tests/via_loadnext/src/command/tx_command.rs
static WEIGHTS: OnceCell<[(TxType, f32); 3]> = OnceCell::new();

/// Type of transaction. It doesn't copy the ZKsync operation list, because
/// it divides some transactions in subcategories (e.g. to new account / to existing account; to self / to other; etc)/
#[derive(Debug, Copy, Clone, PartialEq)]
pub enum TxType {
    Deposit,
    Withdraw,
    DeployContract,
    L2Execute,
}

impl TxType {
    pub fn initialize_weights(transaction_weights: &TransactionWeights) {
        WEIGHTS
            .set([
                (TxType::Deposit, transaction_weights.deposit),
                (TxType::L2Execute, transaction_weights.l2_transactions),
                (TxType::Withdraw, transaction_weights.withdrawal),
            ])
            .unwrap();
    }
}
```

`TxCommand` carries an optional Bitcoin recipient, populated only for withdrawals:

```rust
// core/tests/via_loadnext/src/command/tx_command.rs
/// Complete description of a transaction that must be executed by a test wallet.
#[derive(Debug, Clone)]
pub struct TxCommand {
    /// Type of operation.
    pub command_type: TxType,
    /// Whether and how transaction should be corrupted.
    pub modifier: IncorrectnessModifier,
    /// Recipient address.
    pub to: Address,
    pub to_btc: Option<BitcoinAddress>,
    /// Transaction amount (0 if not applicable).
    pub amount: U256,
}
```

```rust
// core/tests/via_loadnext/src/command/tx_command.rs (inside new_with_type)
        if matches!(command.command_type, TxType::Withdraw) {
            command.to_btc = Some(addresses.random_btc_address(rng));
        }

        // Check whether we should use a self as a target.
        if command.command_type.is_target_self() {
            command.to = own_address;
        }

        // Fix incorrectness modifier:
        // L1 txs should always have `None` modifier.
        if matches!(command.command_type, TxType::Deposit) {
            command.modifier = IncorrectnessModifier::None;
        }
```

There are no `Transfer`, `BtcDeposit`, `BtcWithdraw`, or `BtcTransfer` command types. `Deposit` IS the Bitcoin deposit and `Withdraw` IS the withdrawal to a Bitcoin address.

### 4.2 Error simulation

```rust
// core/tests/via_loadnext/src/command/tx_command.rs
pub enum IncorrectnessModifier {
    ZeroFee,
    IncorrectSignature,

    // Last option goes for no modifier,
    // since it's more convenient than dealing with `Option<IncorrectnessModifier>`.
    None,
}
```

Weighting gives roughly 90% probability of no modifier. Expected outcomes:

```rust
// core/tests/via_loadnext/src/command/tx_command.rs
impl IncorrectnessModifier {
    pub fn expected_outcome(self) -> ExpectedOutcome {
        match self {
            Self::None => ExpectedOutcome::TxSucceed,
            Self::ZeroFee | Self::IncorrectSignature => ExpectedOutcome::ApiRequestFailed,
        }
    }
}
```

### 4.3 API and PubSub commands

```rust
// core/tests/via_loadnext/src/command/api.rs
#[derive(Debug, Copy, Clone)]
pub enum ApiRequestType {
    /// Requests block with full transactions list.
    BlockWithTxs,
    /// Requests account balance.
    Balance,
    /// Requests account-deployed contract events.
    GetLogs,
}
```

```rust
// core/tests/via_loadnext/src/command/pubsub.rs
#[derive(Debug, Copy, Clone, PartialEq, Eq, Hash)]
pub enum SubscriptionType {
    /// Subscribes for new block headers.
    BlockHeaders,
    /// Subscribes for new transactions.
    PendingTransactions,
    /// Subscribes for new logs.
    Logs,
}
```

## 5. Bitcoin Integration

### 5.1 Deposits (L1 Bitcoin to L2)

`account/btc_deposit.rs` builds and signs a raw regtest Bitcoin transaction with `via_btc_client::client::BitcoinClient`: it selects UTXOs of the depositor, pays the bridge address, and encodes the L2 recipient in an `OP_RETURN` output. The fee is hardcoded to 0.0001 BTC and the network is hardcoded to regtest:

```rust
// core/tests/via_loadnext/src/account/btc_deposit.rs
pub async fn deposit(
    amount: u64,
    receiver_l2_address: EVMAddress,
    depositor_private_key: PrivateKey,
    bridge_musig2_address_str: String,
    rpc_url: String,
    rpc_username: String,
    rpc_password: String,
) -> Result<bitcoin::Txid> {
    let secp = Secp256k1::new();

    let amount = Amount::from_sat(amount);
    let fees = Amount::from_btc(0.0001)?;

    let network = bitcoin::Network::Regtest;
```

```rust
// core/tests/via_loadnext/src/account/btc_deposit.rs
    // OP_RETURN output with L2 address.
    ...
        script_pubkey: ScriptBuf::new_op_return(receiver_l2_address.to_fixed_bytes()),
```

Per-account deposits check the account's L1 BTC balance first and skip (not fail) when it is insufficient:

```rust
// core/tests/via_loadnext/src/account/tx_command_executor.rs
    async fn execute_btc_deposit(&self, command: &TxCommand) -> Result<SubmitResult, ClientError> {
        let btc_balance = self.l1_btc_balances().await?;
        if btc_balance.is_zero()
            || btc_balance < command.amount
            || bitcoin::Amount::from_sat(btc_balance.as_u64())
                < bitcoin::Amount::from_btc(0.0001).unwrap()
        {
            // We don't have either funds in L1 to pay for tx or to deposit.
            // It's not a problem with the server, thus we mark this operation as skipped.
            let label = ReportLabel::skipped("No L1 balance");
            return Ok(SubmitResult::ReportLabel(label));
        }

        let deposit_response = btc_deposit::deposit(
            command.amount.as_u64(),
            command.to,
            self.btc_wallet.btc_private_key,
            self.config.bridge_address.clone(),
            self.config.l1_btc_rpc_address.clone(),
            self.config.l1_btc_rpc_username.clone(),
            self.config.l1_btc_rpc_password.clone(),
        )
        .await;
```

### 5.2 Withdrawals (L2 to Bitcoin)

Withdrawals are ordinary L2 transactions built with the SDK's `WithdrawBuilder`, whose recipient type was changed from an Ethereum address to `via_btc_client::types::BitcoinAddress`. Optional paymaster support is gated by `USE_PAYMASTER`:

```rust
// core/tests/via_loadnext/src/account/tx_command_executor.rs
    pub(super) async fn build_withdraw(&self, command: &TxCommand) -> Result<L2Tx, ClientError> {
        let wallet = self.eth_wallet.wallet.clone();

        let mut builder = wallet
            .start_withdraw()
            .to(command.to_btc.clone().unwrap())
            .amount(command.amount);

        let paymaster_approval = if self.config.use_paymaster {
            Some(get_approval_based_paymaster_input_for_estimation(
                self.paymaster_address,
                L2_BASE_TOKEN_ADDRESS,
                MIN_ALLOWANCE_FOR_PAYMASTER_ESTIMATE.into(),
            ))
        } else {
            None
        };
```

The builder's string-address variant also enforces regtest:

```rust
// core/tests/via_loadnext/src/sdk/operations/withdraw.rs
    pub fn str_to(mut self, to: impl AsRef<str>) -> Result<Self, ClientError> {
        let to: BitcoinAddress = BitcoinAddress::from_str(to.as_ref())
            .map_err(|_| ClientError::IncorrectAddress)?
            .require_network(Network::Regtest)
            .map_err(|_| ClientError::IncorrectAddress)?;
```

### 5.3 Network support

Regtest only. `BitcoinNetwork::Regtest` is hardcoded in `account_pool.rs` (key and address generation), `executor.rs`, `account/tx_command_executor.rs`, `account/btc_deposit.rs`, and `sdk/operations/withdraw.rs`. There is no `L1_BTC_NETWORK` configuration option.

## 6. Workflow

```rust
// core/tests/via_loadnext/src/executor.rs
    /// Inner representation of `start` function which returns a `Result`, so it can conveniently use `?`.
    async fn start_inner(&mut self) -> anyhow::Result<LoadtestResult> {
        tracing::info!("Initializing accounts");
        tracing::info!(
            "Running for MASTER {:?}",
            self.pool.eth_master_wallet.address()
        );
        self.check_btc_balance().await?;

        // Deposit BTC to the master account
        self.deposit_btc_to_master().await?;

        // Deposit BTC to the paymaster
        self.deposit_btc_to_paymaster().await?;

        // Distribute BTC on L2 to the accounts
        self.distribute_btc(self.config.accounts_amount).await?;

        let final_result = self.initial_tests().await?;
        Ok(final_result)
    }
```

Details verified in `executor.rs`:

- `check_btc_balance` requires the BTC master wallet to hold at least `accounts_amount * 0.1 + 1.0` BTC on L1 (the `+ 1.0` is for the paymaster) and records the balance in `LOADTEST_METRICS.master_account_balance`.
- `deposit_btc_to_master` sends that amount to the bridge address via `btc_deposit::deposit` with the ETH master wallet's address in the OP_RETURN, then sleeps 10 seconds waiting for L2 confirmation.
- `deposit_btc_to_paymaster` deposits 1 BTC to the address returned by `get_testnet_paymaster()` (panics with "No testnet paymaster is set" if absent).
- `distribute_btc` transfers `100_000_000_000_000_000` (0.1 of the 18-decimal L2 base token) per account on L2 using `L2_BASE_TOKEN_ADDRESS` transfers from the ETH master wallet.
- Then account lifespans run for `duration_sec` seconds, with at most `max_inflight_txs` unconfirmed transactions per account, and the report collector produces the final `LoadtestResult`.

## 7. Configuration

Configuration is read from environment variables via `envy` (field name uppercased = env var name). The full, real config struct:

```rust
// core/tests/via_loadnext/src/config.rs
#[derive(Debug, Clone, Deserialize)]
pub struct LoadtestConfig {
    /// Address of the Bitcoin RPC.
    #[serde(default = "default_l1_btc_rpc_address")]
    pub l1_btc_rpc_address: String,

    /// Username of the Bitcoin RPC.
    #[serde(default = "default_l1_btc_rpc_username")]
    pub l1_btc_rpc_username: String,

    /// Password of the Bitcoin RPC.
    #[serde(default = "default_l1_btc_rpc_password")]
    pub l1_btc_rpc_password: String,

    /// Ethereum private key of the wallet that has funds to perform a test.
    #[serde(default = "default_eth_master_wallet_pk")]
    pub eth_master_wallet_pk: String,

    /// Bitcoin private key of the wallet that has funds to perform a test.
    #[serde(default = "default_btc_master_wallet_pk")]
    pub btc_master_wallet_pk: String,

    /// Amount of accounts to be used in test.
    /// This option configures the "width" of the test:
    /// how many concurrent operation flows will be executed.
    /// The higher the value is, the more load will be put on the node.
    /// If testing the sequencer throughput, this number must be sufficiently high.
    #[serde(default = "default_accounts_amount")]
    pub accounts_amount: usize,

    /// Duration of the test. For proper results, this value should be at least 10 minutes.
    #[serde(default = "default_duration_sec")]
    pub duration_sec: u64,

    /// Limits the number of simultaneous API requests being performed at any moment of time.
    ///
    /// Setting it to:
    /// - 0 turns off API requests.
    /// - `accounts_amount` relieves the limit.
    #[serde(default = "default_sync_api_requests_limit")]
    pub sync_api_requests_limit: usize,

    /// Limits the number of simultaneously active PubSub subscriptions at any moment of time.
    ///
    /// Setting it to:
    /// - 0 turns off PubSub subscriptions.
    #[serde(default = "default_sync_pubsub_subscriptions_limit")]
    pub sync_pubsub_subscriptions_limit: usize,

    /// Time in seconds for a subscription to be active. Subscription will be closed after that time.
    #[serde(default = "default_single_subscription_time_secs")]
    pub single_subscription_time_secs: u64,

    /// Optional seed to be used in the test: normally you don't need to set the seed,
    /// but you can re-use seed from previous run to reproduce the sequence of operations locally.
    /// Seed must be represented as a hexadecimal string.
    ///
    /// Using the same seed doesn't guarantee reproducibility of API requests: unlike operations, these
    /// are generated in flight by multiple accounts in parallel.
    #[serde(default = "default_seed")]
    pub seed: Option<String>,

    /// Chain id of L2 node.
    #[serde(default = "default_l2_chain_id")]
    pub l2_chain_id: u64,

    /// RPC address of L2 node.
    #[serde(default = "default_l2_rpc_address")]
    pub l2_rpc_address: String,

    /// WS RPC address of L2 node.
    #[serde(default = "default_l2_ws_rpc_address")]
    pub l2_ws_rpc_address: String,

    /// The maximum number of transactions per account that can be sent without waiting for confirmation.
    /// Should not exceed the corresponding value in the L2 node configuration.
    #[serde(default = "default_max_inflight_txs")]
    pub max_inflight_txs: usize,

    /// All of test accounts get split into groups that share the
    /// deployed contract address. This helps to emulate the behavior of
    /// sending `Execute` to the same contract and reading its events by
    /// single a group. This value should be less than or equal to `ACCOUNTS_AMOUNT`.
    #[serde(default = "default_accounts_group_size")]
    pub accounts_group_size: usize,

    /// The expected number of the processed transactions during loadtest
    /// that should be compared to the actual result.
    /// If the value is `None`, the comparison is not performed.
    #[serde(default = "default_expected_tx_count")]
    pub expected_tx_count: Option<usize>,

    /// Label to use for results pushed to Prometheus.
    #[serde(default = "default_prometheus_label")]
    pub prometheus_label: String,

    /// Fail the load test immediately if a failure is encountered that would result
    /// in an eventual test failure anyway (e.g., a failure processing transactions).
    #[serde(default)]
    pub fail_fast: bool,

    /// use pay master to pay the transaction fee
    #[serde(default)]
    pub use_paymaster: bool,

    /// The via bridge address
    #[serde(default = "default_bridge_address")]
    pub bridge_address: String,
}
```

```rust
// core/tests/via_loadnext/src/config.rs
impl LoadtestConfig {
    pub fn from_env() -> envy::Result<Self> {
        envy::from_env()
    }
```

### 7.1 Environment variables and defaults

| Env var | Default | Source (config.rs) |
|---|---|---|
| `L1_BTC_RPC_ADDRESS` | `http://127.0.0.1:18443` | `default_l1_btc_rpc_address` |
| `L1_BTC_RPC_USERNAME` | `rpcuser` | `default_l1_btc_rpc_username` |
| `L1_BTC_RPC_PASSWORD` | `rpcpassword` | `default_l1_btc_rpc_password` |
| `ETH_MASTER_WALLET_PK` | `7726827caac94a7f9e1b160f7ea819f172f7b6f9d2a97f992c38edeab82d4110` (localhost-only, compromised key) | `default_eth_master_wallet_pk` |
| `BTC_MASTER_WALLET_PK` | `cVn486kDX5Mr9MimMyiRNMR4ZsKaLbLho3MHZgqVriB5q3S8FKKF` (regtest WIF, address `bcrt1q8tuqv885kehnzucdfskuw6mrhxcj7cjs4gfk5z`) | `default_btc_master_wallet_pk` |
| `ACCOUNTS_AMOUNT` | `10` | `default_accounts_amount` |
| `DURATION_SEC` | `60` | `default_duration_sec` |
| `ACCOUNTS_GROUP_SIZE` | `1` | `default_accounts_group_size` |
| `MAX_INFLIGHT_TXS` | `5` | `default_max_inflight_txs` |
| `SYNC_API_REQUESTS_LIMIT` | `20` | `default_sync_api_requests_limit` |
| `SYNC_PUBSUB_SUBSCRIPTIONS_LIMIT` | `150` | `default_sync_pubsub_subscriptions_limit` |
| `SINGLE_SUBSCRIPTION_TIME_SECS` | `30` | `default_single_subscription_time_secs` |
| `SEED` | none | `default_seed` |
| `L2_CHAIN_ID` | `L2ChainId::default()` | `default_l2_chain_id` |
| `L2_RPC_ADDRESS` | `http://127.0.0.1:3050` | `default_l2_rpc_address` |
| `L2_WS_RPC_ADDRESS` | `ws://127.0.0.1:3051` | `default_l2_ws_rpc_address` |
| `EXPECTED_TX_COUNT` | none | `default_expected_tx_count` |
| `PROMETHEUS_LABEL` | `unset` | `default_prometheus_label` |
| `FAIL_FAST` | `false` | serde default |
| `USE_PAYMASTER` | `false` | serde default |
| `BRIDGE_ADDRESS` | `bcrt1p3s7m76wp5seprjy4gdxuxrr8pjgd47q5s8lu9vefxmp0my2p4t9qh6s8kq` | `default_bridge_address` |

Additionally, `main.rs` reads `MISC_LOG_FORMAT`, `MISC_SENTRY_URL`, `CHAIN_ETH_NETWORK`, `CHAIN_ETH_ZKSYNC_NETWORK` for observability, and `PROMETHEUS_`-prefixed variables for the push-gateway exporter:

```rust
// core/tests/via_loadnext/src/main.rs
    let config = LoadtestConfig::from_env()
        .expect("Config parameters should be loaded from env or from default values");
    let execution_config = ExecutionConfig::from_env();
    let prometheus_config: Option<PrometheusConfig> = envy::prefixed("PROMETHEUS_").from_env().ok();

    TxType::initialize_weights(&execution_config.transaction_weights);
```

### 7.2 Transaction weights

Only three weights exist. There are no `l1_transactions`, `btc_deposit`, or `btc_withdrawal` weights:

```rust
// core/tests/via_loadnext/src/config.rs
#[derive(Debug, Clone, Deserialize)]
pub struct TransactionWeights {
    pub deposit: f32,
    pub withdrawal: f32,
    pub l2_transactions: f32,
}

impl TransactionWeights {
    pub fn from_env() -> Option<Self> {
        envy::prefixed("TRANSACTION_WEIGHTS_").from_env().ok()
    }
}

impl Default for TransactionWeights {
    fn default() -> Self {
        Self {
            deposit: 0.05,
            withdrawal: 0.5,
            l2_transactions: 1.0,
        }
    }
}
```

So the env vars are `TRANSACTION_WEIGHTS_DEPOSIT`, `TRANSACTION_WEIGHTS_WITHDRAWAL`, and `TRANSACTION_WEIGHTS_L2_TRANSACTIONS`. Note that `envy` deserializes the whole struct at once: if any of the three is missing, all fall back to the defaults.

### 7.3 Contract execution parameters

Read with the `CONTRACT_EXECUTION_PARAMS_` prefix into `zksync_test_contracts::LoadnextContractExecutionParams`:

```rust
// core/lib/test_contracts (documented in core/tests/via_loadnext/README.md)
pub struct LoadnextContractExecutionParams {
    pub reads: usize,
    pub initial_writes: usize,
    pub repeated_writes: usize,
    pub events: usize,
    pub hashes: usize,
    pub recursive_calls: usize,
    pub deploys: usize,
}
```

Env vars: `CONTRACT_EXECUTION_PARAMS_READS`, `CONTRACT_EXECUTION_PARAMS_INITIAL_WRITES`, `CONTRACT_EXECUTION_PARAMS_REPEATED_WRITES`, `CONTRACT_EXECUTION_PARAMS_EVENTS`, `CONTRACT_EXECUTION_PARAMS_HASHES`, `CONTRACT_EXECUTION_PARAMS_RECURSIVE_CALLS`, `CONTRACT_EXECUTION_PARAMS_DEPLOYS`. There is no `CONTRACT_EXECUTION_PARAMS_WRITES`; writes are split into initial and repeated.

### 7.4 Request limiters

```rust
// core/tests/via_loadnext/src/config.rs
#[derive(Debug)]
pub struct RequestLimiters {
    pub api_requests: Semaphore,
    pub subscriptions: Semaphore,
}

impl RequestLimiters {
    pub fn new(config: &LoadtestConfig) -> Self {
        Self {
            api_requests: Semaphore::new(config.sync_api_requests_limit),
            subscriptions: Semaphore::new(config.sync_pubsub_subscriptions_limit),
        }
    }
}
```

## 8. Running

Prerequisites: a running Via L2 node (default RPC `http://127.0.0.1:3050`), a Bitcoin regtest node (default `http://127.0.0.1:18443` with `rpcuser`/`rpcpassword`), a funded BTC master wallet (at least `accounts_amount * 0.1 + 1.0` BTC), and the correct bridge address.

Via the CLI:

```bash
via run loadtest
```

Or directly with cargo (all variables optional; defaults target the local regtest environment):

```bash
CONTRACT_EXECUTION_PARAMS_INITIAL_WRITES=2 \
CONTRACT_EXECUTION_PARAMS_REPEATED_WRITES=2 \
CONTRACT_EXECUTION_PARAMS_READS=6 \
CONTRACT_EXECUTION_PARAMS_EVENTS=2 \
CONTRACT_EXECUTION_PARAMS_HASHES=10 \
CONTRACT_EXECUTION_PARAMS_RECURSIVE_CALLS=0 \
CONTRACT_EXECUTION_PARAMS_DEPLOYS=0 \
ACCOUNTS_AMOUNT=10 \
ACCOUNTS_GROUP_SIZE=1 \
MAX_INFLIGHT_TXS=5 \
RUST_LOG="info,via_loadnext=debug" \
SYNC_API_REQUESTS_LIMIT=0 \
SYNC_PUBSUB_SUBSCRIPTIONS_LIMIT=0 \
TRANSACTION_WEIGHTS_DEPOSIT=0.05 \
TRANSACTION_WEIGHTS_WITHDRAWAL=0.5 \
TRANSACTION_WEIGHTS_L2_TRANSACTIONS=1 \
DURATION_SEC=600 \
ETH_MASTER_WALLET_PK="..." \
BTC_MASTER_WALLET_PK="..." \
L1_BTC_RPC_ADDRESS="http://127.0.0.1:18443" \
L1_BTC_RPC_USERNAME="rpcuser" \
L1_BTC_RPC_PASSWORD="rpcpassword" \
BRIDGE_ADDRESS="bcrt1p..." \
cargo run --release --bin via_loadnext
```

The unmodified upstream crate can still be run as `cargo run --bin loadnext` (and is wired to `zk run loadtest` in `infrastructure/zk/src/run.ts`), but its Ethereum L1 paths are not usable against Via.

## 9. Metrics and Reporting

- Average TPS and per-operation reports come from the inherited `ReportCollector` pipeline (`report_collector/mod.rs` includes a `PrometheusCollector`, "which exposes the ongoing loadtest results to grafana via prometheus").
- `LOADTEST_METRICS.master_account_balance` records the master BTC balance (in sats) at startup.
- If `PROMETHEUS_`-prefixed variables are set, `main.rs` starts a Prometheus push-gateway exporter; otherwise it logs "Starting without prometheus exporter".
- `EXPECTED_TX_COUNT`, when set, is compared against the actual processed transaction count.

## 10. Known Limitations

- Bitcoin network is hardcoded to regtest throughout; the tool cannot run against testnet or mainnet without code changes.
- The deposit fee is hardcoded to 0.0001 BTC in `btc_deposit.rs`.
- The master deposit flow waits a fixed 10 seconds for L2 confirmation instead of polling.
- The `README.md` inside `core/tests/via_loadnext` is still the unmodified ZKsync loadnext README; its example invocation (with `MASTER_WALLET_PK`, `MAIN_TOKEN`, `TRANSACTION_WEIGHTS_L1_TRANSACTIONS`, `--bin loadnext`) does not match the Via fork.
- The paymaster funding step calls `get_testnet_paymaster()` and panics if the node has no testnet paymaster configured.
