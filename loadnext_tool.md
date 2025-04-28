# Via L2 Bitcoin ZK-Rollup: Loadnext Tool Documentation

## 1. Introduction

The Via Loadnext tool is a comprehensive load testing utility designed for stress-testing the Via L2 Bitcoin ZK-Rollup protocol. Adapted from the ZKsync loadnext tool, it has been modified to work with Via's Bitcoin integration, enabling thorough testing of the protocol's performance, reliability, and robustness under various conditions.

## 2. Purpose and Functionality

The Loadnext tool serves several critical purposes in the Via ecosystem:

1. **Performance Testing**: Measures the system's ability to handle high transaction volumes and calculates metrics like transactions per second (TPS)
2. **Stress Testing**: Pushes the system to its limits by simulating many concurrent users and transactions
3. **Reliability Testing**: Verifies the system's ability to handle both valid and invalid transactions appropriately
4. **Integration Testing**: Tests the complete flow from L1 (Bitcoin) to L2 (Via) and back, including deposits and withdrawals
5. **Regression Testing**: Ensures new protocol changes don't negatively impact performance or functionality

## 3. Key Features

The Loadnext tool offers a robust set of features:

- **Resilient Operation**: Continues testing even if the server becomes unresponsive
- **Diverse Operations**: Performs a unique set of operations for each test account
- **Multi-layer Testing**: Tests both L1-to-L2 and L2-to-L1 operations
- **Error Simulation**: Deliberately sends incorrect transactions to test error handling
- **Extensible Architecture**: Provides an easy-to-extend command system for adding new test scenarios
- **Comprehensive Reporting**: Collects and analyzes detailed performance metrics

## 4. Architecture and Components

### 4.1 Core Components

The Loadnext tool is structured around several key components:

#### 4.1.1 Account Management
- `AccountPool`: Manages a collection of test wallets with both L2 and Bitcoin capabilities
- `TestWallet`: Represents a test user with Ethereum wallet capabilities
- `BtcAccount`: Represents a Bitcoin account for testing Bitcoin operations
- `AccountLifespan`: Manages the lifecycle of test accounts during the test, including Bitcoin wallet operations

#### 4.1.2 Transaction System
- `TxCommand`: Represents a transaction to be executed
- `ApiRequest`: Represents an API request to be made
- `SubscriptionType`: Represents a subscription to events
- Various transaction builders for different operation types

#### 4.1.3 Reporting and Metrics
- `ReportBuilder`: Collects and formats test results
- `MetricsCollector`: Gathers performance metrics
- `TimeHistogram`: Tracks execution times for different operations
- `OperationResultsCollector`: Analyzes success/failure rates

#### 4.1.4 Execution Engine
- `Executor`: Orchestrates the entire test process
- `RequestLimiters`: Controls concurrency to prevent overwhelming the system

### 4.2 Bitcoin Integration

The tool includes specialized components for Bitcoin operations:

- Bitcoin wallet management with private key handling
- Bitcoin deposit and withdrawal functionality
- Bitcoin address handling and validation
- Bitcoin transaction creation and signing
- Bitcoin UTXO management
- Bitcoin fee estimation

The Bitcoin integration has been significantly enhanced to support:

- Direct Bitcoin deposits to L2 via the bridge address
- Bitcoin withdrawals from L2 to specified Bitcoin addresses
- Bitcoin transaction monitoring and confirmation tracking
- Bitcoin wallet balance management

## 5. Workflow

The general flow of a Loadnext test is as follows:

1. **Initialization**:
   - Bitcoin wallets are created for each test account
   - The master account performs an initial deposit to L2
   - The paymaster on L2 is funded if necessary
   - The L2 master account distributes funds to participating accounts
   - Bitcoin funds are distributed to test accounts if Bitcoin operations are enabled

2. **Test Execution**:
   - Each account continuously sends transactions as configured
   - Transactions can include L2 operations, Bitcoin deposits, and Bitcoin withdrawals
   - At any given time, there are no more than `max_inflight_txs` transactions in flight for each account
   - The test runs for the configured duration (`duration_sec`)

3. **Finalization**:
   - After the test is finished, the master account withdraws remaining funds
   - Bitcoin balances are consolidated if configured
   - The average TPS and other metrics are reported

## 6. Configuration Options

The Loadnext tool is highly configurable through environment variables:

### 6.1 General Configuration
- `ACCOUNTS_AMOUNT`: Number of accounts to use in the test
- `ACCOUNTS_GROUP_SIZE`: Number of accounts sharing a contract address
- `MAX_INFLIGHT_TXS`: Maximum number of in-flight transactions per account
- `DURATION_SEC`: Test duration in seconds
- `SEED`: Random seed for reproducible tests

### 6.2 Network Configuration
- `L2_RPC_ADDRESS`: Address of the L2 RPC endpoint
- `L2_WS_RPC_ADDRESS`: Address of the L2 WebSocket RPC endpoint
- `L1_BTC_RPC_ADDRESS`: Address of the Bitcoin RPC endpoint
- `L1_BTC_RPC_USER`: Username for Bitcoin RPC authentication
- `L1_BTC_RPC_PASSWORD`: Password for Bitcoin RPC authentication
- `L1_BTC_NETWORK`: Bitcoin network to use (mainnet, testnet, regtest)
- `L2_CHAIN_ID`: Chain ID of the L2 network

### 6.3 Contract Execution Parameters
- `CONTRACT_EXECUTION_PARAMS_READS`: Number of storage reads per transaction
- `CONTRACT_EXECUTION_PARAMS_WRITES`: Number of storage writes per transaction
- `CONTRACT_EXECUTION_PARAMS_EVENTS`: Number of events emitted per transaction
- `CONTRACT_EXECUTION_PARAMS_HASHES`: Number of hash operations per transaction
- `CONTRACT_EXECUTION_PARAMS_RECURSIVE_CALLS`: Number of recursive calls per transaction
- `CONTRACT_EXECUTION_PARAMS_DEPLOYS`: Number of contract deployments per transaction

### 6.4 Transaction Weights
- `TRANSACTION_WEIGHTS_DEPOSIT`: Weight for deposit transactions
- `TRANSACTION_WEIGHTS_WITHDRAWAL`: Weight for withdrawal transactions
- `TRANSACTION_WEIGHTS_L2_TRANSACTIONS`: Weight for L2 transactions
- `TRANSACTION_WEIGHTS_BTC_DEPOSIT`: Weight for Bitcoin deposit transactions
- `TRANSACTION_WEIGHTS_BTC_WITHDRAWAL`: Weight for Bitcoin withdrawal transactions

### 6.5 Bitcoin Configuration
- `BTC_MASTER_WALLET_PK`: Private key for the Bitcoin master wallet
- `BTC_ACCOUNTS_AMOUNT`: Number of Bitcoin accounts to use in the test
- `BTC_MIN_CONFIRMATIONS`: Minimum confirmations required for Bitcoin transactions
- `BTC_FEE_RATE`: Fee rate for Bitcoin transactions (in satoshis per byte)
- `BTC_DEPOSIT_AMOUNT`: Amount to deposit in each Bitcoin deposit transaction
- `BTC_WITHDRAWAL_AMOUNT`: Amount to withdraw in each Bitcoin withdrawal transaction

## 7. Example Usage

To run a load test with parameters similar to mainnet:

```bash
cargo build

CONTRACT_EXECUTION_PARAMS_WRITES=2 \
CONTRACT_EXECUTION_PARAMS_READS=6 \
CONTRACT_EXECUTION_PARAMS_EVENTS=2 \
CONTRACT_EXECUTION_PARAMS_HASHES=10 \
CONTRACT_EXECUTION_PARAMS_RECURSIVE_CALLS=0 \
CONTRACT_EXECUTION_PARAMS_DEPLOYS=0 \
ACCOUNTS_AMOUNT=300 \
ACCOUNTS_GROUP_SIZE=300 \
MAX_INFLIGHT_TXS=20 \
RUST_LOG="info,loadnext=debug" \
SYNC_API_REQUESTS_LIMIT=0 \
SYNC_PUBSUB_SUBSCRIPTIONS_LIMIT=0 \
TRANSACTION_WEIGHTS_DEPOSIT=0 \
TRANSACTION_WEIGHTS_WITHDRAWAL=0 \
TRANSACTION_WEIGHTS_L1_TRANSACTIONS=0 \
TRANSACTION_WEIGHTS_L2_TRANSACTIONS=1 \
TRANSACTION_WEIGHTS_BTC_DEPOSIT=0.1 \
TRANSACTION_WEIGHTS_BTC_WITHDRAWAL=0.1 \
DURATION_SEC=1200 \
MASTER_WALLET_PK="..." \
BTC_MASTER_WALLET_PK="..." \
L1_BTC_RPC_USER="rpcuser" \
L1_BTC_RPC_PASSWORD="rpcpassword" \
L1_BTC_NETWORK="regtest" \
cargo run --bin via_loadnext
```

## 8. Implementation Details

### 8.1 Transaction Types

The tool supports various transaction types:

- **Transfer**: Sends funds from one L2 account to another
- **Withdraw**: Withdraws funds from L2 to L1 (Bitcoin)
- **Deposit**: Deposits funds from L1 (Bitcoin) to L2
- **DeployContract**: Deploys a test contract to L2
- **L2Execute**: Executes a function on a deployed contract
- **BtcDeposit**: Deposits Bitcoin directly to L2 via the bridge address
- **BtcWithdraw**: Withdraws funds from L2 to a Bitcoin address
- **BtcTransfer**: Transfers Bitcoin between Bitcoin addresses

### 8.2 API Request Types

The tool can also test various API endpoints:

- **BlockWithTxs**: Retrieves block information with transactions
- **Balance**: Checks account balances
- **GetLogs**: Retrieves event logs from contracts

### 8.3 Error Simulation

To test error handling, the tool can deliberately create invalid transactions:

- **ZeroFee**: Creates transactions with zero fee
- **IncorrectSignature**: Creates transactions with invalid signatures

## 9. Metrics and Reporting

The tool collects various metrics during testing:

- **Transaction Throughput**: Measures transactions per second (TPS)
- **Response Times**: Tracks how long operations take to complete
- **Success Rates**: Monitors the percentage of successful operations
- **Error Distribution**: Analyzes the types of errors encountered
- **Bitcoin Transaction Metrics**: Tracks Bitcoin-specific metrics like confirmation times and fee rates

These metrics are reported both in the console output and can be exported to Prometheus for visualization.

## 11. Bitcoin-Specific Features

The Via Loadnext tool includes several Bitcoin-specific features:

### 11.1 Bitcoin Wallet Management

- Automatic creation of Bitcoin wallets for test accounts
- Private key management for Bitcoin transactions
- UTXO tracking and management
- Balance monitoring and consolidation

### 11.2 Bitcoin Transaction Types

- **Direct Deposits**: Sends Bitcoin directly to the bridge address with L2 recipient information
- **L2 Withdrawals**: Initiates withdrawals from L2 to specified Bitcoin addresses
- **Bitcoin Transfers**: Moves Bitcoin between test accounts for UTXO management

### 11.3 Bitcoin Network Compatibility

The tool supports different Bitcoin networks:
- **Mainnet**: For production testing
- **Testnet**: For staging environment testing
- **Regtest**: For local development testing

Each network has different parameters for fee estimation, confirmation times, and address formats.

## 10. Conclusion

The Via Loadnext tool is an essential component of the Via L2 Bitcoin ZK-Rollup ecosystem, providing comprehensive load testing capabilities to ensure the protocol can handle real-world usage scenarios. By simulating various user behaviors and transaction patterns, it helps identify performance bottlenecks and reliability issues before they affect production systems.

The tool's adaptation from ZKsync to Via represents a significant shift from Ethereum-based operations to Bitcoin-based operations, reflecting the Via protocol's focus on Bitcoin as the L1 layer rather than Ethereum.