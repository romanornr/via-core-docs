# Via L2 Bitcoin ZK-Rollup: RPC/API Layer Documentation

## 1. Overview

The RPC/API Layer in Via L2 is the primary interface for users and applications to interact with the rollup. It provides a set of JSON-RPC endpoints that allow clients to query the state of the blockchain, submit transactions, and subscribe to events. The API layer is designed to be compatible with Ethereum's JSON-RPC interface, making it easy for existing Ethereum tools and libraries to work with Via L2.

## 2. Architecture

The RPC/API Layer consists of several key components:

### 2.1 Core Components

1. **API Server**: The main component that handles HTTP and WebSocket requests, routes them to the appropriate handlers, and returns responses.
2. **Namespaces**: Logical groupings of related API methods (e.g., `eth`, `zks`, `via`, `web3`, `net`).
3. **Transaction Sender**: Handles transaction submission and validation.
4. **State Access**: Provides access to the blockchain state through database connections.
5. **Filters and Subscriptions**: Manages event filters and WebSocket subscriptions.

### 2.2 Role in Via L2 Architecture

The RPC/API Layer serves as the primary interface between external users/applications and the Via L2 system. It:

- Receives transaction submissions from users and forwards them to the Mempool
- Provides query capabilities for blockchain state, transactions, blocks, and events
- Offers WebSocket subscriptions for real-time updates
- Exposes ZKSync-specific functionality through custom API methods
- Provides Via-specific endpoints for Bitcoin integration

## 3. API Standards and Namespaces

Via L2 implements several API namespaces, each providing a set of related methods:

### 3.1 Standard Ethereum JSON-RPC Namespaces

#### `eth` Namespace

Implements standard Ethereum JSON-RPC methods:

- **State Access**: `eth_getBalance`, `eth_getStorageAt`, `eth_getCode`, `eth_getTransactionCount`
- **Block Information**: `eth_blockNumber`, `eth_getBlockByNumber`, `eth_getBlockByHash`, `eth_getBlockTransactionCountByNumber`, `eth_getBlockTransactionCountByHash`, `eth_getBlockReceipts`
- **Transaction Operations**: `eth_sendRawTransaction`, `eth_getTransactionByHash`, `eth_getTransactionReceipt`, `eth_estimateGas`, `eth_call`
- **Event Filtering**: `eth_getLogs`, `eth_newFilter`, `eth_newBlockFilter`, `eth_newPendingTransactionFilter`, `eth_getFilterChanges`, `eth_getFilterLogs`, `eth_uninstallFilter`
- **Chain Information**: `eth_chainId`, `eth_gasPrice`, `eth_feeHistory`, `eth_syncing`
- **Other Methods**: `eth_accounts`, `eth_protocolVersion`

#### `net` Namespace

Network-related methods:

- `net_version`: Returns the current network ID
- `net_listening`: Returns if client is actively listening for network connections
- `net_peerCount`: Returns number of peers currently connected to the client

#### `web3` Namespace

Utility methods:

- `web3_clientVersion`: Returns the current client version
- `web3_sha3`: Returns Keccak-256 hash of the given data

### 3.2 Via and ZKSync-Specific Namespaces

#### `via` Namespace

Via-specific methods for Bitcoin integration:

- **Bitcoin Information**: `via_getBitcoinBlockNumber`, `via_getBitcoinBlock`, `via_getBitcoinTransaction`
- **Bitcoin Mempool**: `via_getBitcoinMempoolInfo` - Retrieves Bitcoin mempool information for fee estimation and network monitoring
- **Bridge Information**: `via_getBridgeAddress`, `via_getVerifierAddresses`
- **Protocol Information**: `via_getProtocolVersion`, `via_getCurrentProtocolVersion`
- **Block and Transaction Details**: `via_getBlockDetails`, `via_getTransactionDetails`, `via_getRawBlockTransactions`
- **Deposit and Withdrawal**: `via_estimateDepositFee`, `via_getWithdrawalStatus`
- **Coordinator API**: Enhanced signature validation endpoints with cryptographic verification

#### `zks` Namespace

ZKSync-specific methods:

- **Fee Estimation**: `zks_estimateFee`, `zks_estimateGasL1ToL2`, `zks_getFeeParams`, `zks_getBatchFeeInput`
- **Contract Information**: `zks_getMainContract`, `zks_getBridgehubContract`, `zks_getTestnetPaymaster`, `zks_getBridgeContracts`, `zks_getBaseTokenL1Address`
- **Chain Information**: `zks_L1ChainId`, `zks_L1BatchNumber`, `zks_getL1BatchBlockRange`, `zks_getProtocolVersion`
- **Token Information**: `zks_getConfirmedTokens`, `zks_getAllAccountBalances`
- **Block and Transaction Details**: `zks_getBlockDetails`, `zks_getTransactionDetails`, `zks_getRawBlockTransactions`, `zks_getL1BatchDetails`, `zks_getBytecodeByHash`
- **Proof Generation**: `zks_getL2ToL1MsgProof`, `zks_getL2ToL1LogProof`, `zks_getProof`
- **Transaction Submission**: `zks_sendRawTransactionWithDetailedOutput`

#### `en` Namespace

External Node specific methods:

- Methods for synchronization and state management specific to external nodes

#### `debug` Namespace

Debug and tracing methods:

- Methods for transaction tracing and debugging

#### `snapshots` Namespace

Snapshot-related methods:

- Methods for managing and accessing blockchain snapshots

### 3.3 WebSocket Subscriptions

Via L2 supports WebSocket subscriptions through the `eth_subscribe` method, allowing clients to receive real-time updates for:

- New blocks
- New pending transactions
- Logs matching specific criteria

## 4. Implementation Details

### 4.1 Enhanced Bitcoin RPC URL Support

Via L2 now supports enhanced Bitcoin RPC URL configurations with wallet-specific paths:

#### Wallet-Specific RPC URLs

Bitcoin RPC URLs can now include wallet-specific paths for targeted operations:

```
# Standard RPC URL
http://user:password@localhost:8332

# Wallet-specific RPC URL
http://user:password@localhost:8332/wallet/wallet_name
http://user:password@localhost:8332/wallet/bc1qxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

#### Configuration Examples

```toml
# Standard configuration
[bitcoin_rpc]
url = "http://user:password@localhost:8332"

# Wallet-specific configuration
[bitcoin_rpc]
url = "http://user:password@localhost:8332/wallet/main_wallet"
enable_wallet_operations = true

# Multiple wallet support
[bitcoin_rpc.wallets]
default = "http://user:password@localhost:8332/wallet/default"
bridge = "http://user:password@localhost:8332/wallet/bridge_wallet"
```

#### Enhanced RPC Client Features

- **Wallet-Specific UTXO Fetching**: Retrieve UTXOs for specific wallet addresses
- **Enhanced RPC Client Configuration**: Improved connection handling for wallet operations
- **Automatic Wallet Path Detection**: Smart routing based on wallet address patterns

### 4.2 New BitcoinRpc Trait Methods

#### `get_mempool_info()` Method

The new `BitcoinRpc::get_mempool_info()` trait method provides comprehensive Bitcoin mempool information:

```rust
pub trait BitcoinRpc {
    async fn get_mempool_info(&self) -> Result<MempoolInfo, RpcError>;
}

#[derive(Debug, Serialize, Deserialize)]
pub struct MempoolInfo {
    pub loaded: bool,
    pub size: u64,
    pub bytes: u64,
    pub usage: u64,
    pub total_fee: f64,
    pub max_mempool: u64,
    pub mempool_min_fee: f64,
    pub min_relay_tx_fee: f64,
    pub unbroadcast_count: u64,
}
```

#### API Integration

```json
// Request
{
  "jsonrpc": "2.0",
  "method": "via_getBitcoinMempoolInfo",
  "params": [],
  "id": 1
}

// Response
{
  "jsonrpc": "2.0",
  "result": {
    "loaded": true,
    "size": 1234,
    "bytes": 567890,
    "usage": 1024000,
    "total_fee": 0.00123456,
    "max_mempool": 300000000,
    "mempool_min_fee": 0.00001000,
    "min_relay_tx_fee": 0.00001000,
    "unbroadcast_count": 5
  },
  "id": 1
}
```

### 4.3 Server Configuration

The API server is configured and initialized in the node builder:

- **HTTP Server**: Handles standard JSON-RPC requests
- **WebSocket Server**: Handles WebSocket connections for subscriptions
- **Server Components**: New `--components` CLI flag for selective component execution
- **Configuration Parameters**:
  - Ports for HTTP and WebSocket servers
  - API namespaces to enable
  - Limits for filters, subscriptions, and batch requests
  - Response size limits
  - Rate limiting for WebSocket requests

#### Server Component API

The new `--components` CLI flag allows selective server component execution:

```bash
# Start server with specific components
via_server --components api,mempool,state_keeper

# Start server with all components (default)
via_server --components all

# Start server with minimal components
via_server --components api
```

**Available Components:**
- `api`: RPC/API server
- `mempool`: Transaction mempool
- `state_keeper`: State management
- `eth_sender`: Ethereum transaction sender
- `btc_sender`: Bitcoin transaction sender
- `tree`: Merkle tree management
- `proof_data_handler`: Proof data processing

### 4.4 Request Routing

1. Incoming requests are received by the HTTP or WebSocket server
2. Requests are parsed and validated
3. The appropriate namespace and method handler is identified
4. The method handler processes the request, accessing the blockchain state as needed
5. Enhanced validation is applied for security-critical endpoints
6. The response is formatted and returned to the client

### 4.5 State Access

The API server accesses blockchain state through:

- **Connection Pool**: Manages database connections for efficient state access
- **Storage DAL (Data Access Layer)**: Provides methods for querying blockchain data
- **Enhanced Database Queries**: Improved L1 batch details queries with better data availability
- **Caching**: Implements caching for frequently accessed data to improve performance

#### Database Query Improvements

**Enhanced L1 Batch Details Query:**
- Fetches batches even if commit/proof transactions are unconfirmed
- Improved data availability for batch information
- Enhanced query reliability and data consistency
- Better handling of pending transaction states

```sql
-- Enhanced batch query with improved reliability
SELECT
    l1_batches.*,
    commit_tx.hash as commit_tx_hash,
    prove_tx.hash as prove_tx_hash,
    execute_tx.hash as execute_tx_hash
FROM l1_batches
LEFT JOIN l1_txs commit_tx ON l1_batches.eth_commit_tx_id = commit_tx.id
LEFT JOIN l1_txs prove_tx ON l1_batches.eth_prove_tx_id = prove_tx.id
LEFT JOIN l1_txs execute_tx ON l1_batches.eth_execute_tx_id = execute_tx.id
WHERE l1_batches.number = $1
-- Fetch even if transactions are unconfirmed
```

### 4.6 WebSocket Handling

WebSocket connections are managed by:

- **Subscription Manager**: Tracks active subscriptions
- **Event Notifiers**: Monitor for new blocks, transactions, and logs
- **Notification Dispatcher**: Sends notifications to subscribed clients

## 5. Enhanced API Security and Validation

### 5.1 Coordinator API Validation Enhancements

The coordinator API now implements stricter validation on critical endpoints:

#### `/submit_partial_signature` Endpoint

Enhanced security measures for partial signature submission:

```json
// Request with enhanced validation
{
  "jsonrpc": "2.0",
  "method": "coordinator_submitPartialSignature",
  "params": {
    "batch_id": "0x123...",
    "signature": {
      "r": "0x...",
      "s": "0x...",
      "v": 27
    },
    "signer_address": "0x...",
    "timestamp": 1640995200,
    "nonce": "0x..."
  },
  "id": 1
}
```

**Validation Steps:**
1. **Cryptographic Signature Verification**: Validates signature against expected signer
2. **Timestamp Validation**: Ensures signature is within acceptable time window
3. **Nonce Verification**: Prevents replay attacks
4. **Batch State Validation**: Confirms batch is in correct state for signature submission
5. **Signer Authorization**: Verifies signer is authorized for the specific batch

**Error Responses:**

```json
// Invalid signature
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32602,
    "message": "Invalid signature",
    "data": {
      "reason": "Signature verification failed",
      "expected_signer": "0x...",
      "provided_signature": "0x..."
    }
  },
  "id": 1
}

// Unauthorized signer
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32603,
    "message": "Unauthorized signer",
    "data": {
      "reason": "Signer not authorized for batch",
      "batch_id": "0x123...",
      "signer": "0x..."
    }
  },
  "id": 1
}
```

### 5.2 Enhanced Authentication Requirements

**API Key Authentication:**
```http
POST /api/v1/coordinator/submit_partial_signature
Authorization: Bearer <api_key>
Content-Type: application/json
X-Signature: <request_signature>
X-Timestamp: <unix_timestamp>
```

**Request Signing:**
All coordinator API requests must be signed with the following format:
```
signature = sign(method + url + timestamp + body_hash, private_key)
```

## 6. Interactions with Other Components

### 6.1 Mempool

- The API server submits transactions to the Mempool via the `tx_sender` component
- The Mempool validates and stores pending transactions
- The API server can query pending transactions from the Mempool
- Enhanced mempool integration with Bitcoin mempool information

### 6.2 State Management

- The API server queries blockchain state from the database
- State queries include account balances, storage, code, and transaction data
- The API server can execute view calls against the current state
- Enhanced database queries with improved reliability for batch information

### 6.3 Block Information

- The API server retrieves block information from the database
- Block queries include block headers, transactions, receipts, and logs
- The API server tracks the latest sealed block number for consistent responses
- Improved batch details retrieval with better handling of unconfirmed transactions

### 6.4 Bitcoin Integration

- Enhanced Bitcoin RPC client with wallet-specific operations
- Mempool information integration for fee estimation
- Improved UTXO management with wallet-specific queries
- Better error handling and connection management for Bitcoin operations

## 7. API Method Examples

### 7.1 Enhanced Bitcoin RPC Examples

#### Wallet-Specific UTXO Query

```json
// Request
{
  "jsonrpc": "2.0",
  "method": "via_getWalletUTXOs",
  "params": {
    "wallet_address": "bc1qxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "min_confirmations": 1
  },
  "id": 1
}

// Response
{
  "jsonrpc": "2.0",
  "result": {
    "utxos": [
      {
        "txid": "abc123...",
        "vout": 0,
        "amount": 0.001,
        "confirmations": 6,
        "spendable": true,
        "address": "bc1qxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
      }
    ],
    "total_amount": 0.001,
    "total_count": 1
  },
  "id": 1
}
```

#### Bitcoin Mempool Information

```json
// Request
{
  "jsonrpc": "2.0",
  "method": "via_getBitcoinMempoolInfo",
  "params": [],
  "id": 1
}

// Response
{
  "jsonrpc": "2.0",
  "result": {
    "loaded": true,
    "size": 1234,
    "bytes": 567890,
    "usage": 1024000,
    "total_fee": 0.00123456,
    "max_mempool": 300000000,
    "mempool_min_fee": 0.00001000,
    "min_relay_tx_fee": 0.00001000,
    "unbroadcast_count": 5,
    "fee_histogram": [
      {"fee_rate": 1.0, "count": 100},
      {"fee_rate": 5.0, "count": 200},
      {"fee_rate": 10.0, "count": 150}
    ]
  },
  "id": 1
}
```

### 7.2 Enhanced L1 Batch Details

```json
// Request
{
  "jsonrpc": "2.0",
  "method": "zks_getL1BatchDetails",
  "params": [123],
  "id": 1
}

// Response with enhanced data availability
{
  "jsonrpc": "2.0",
  "result": {
    "number": 123,
    "timestamp": 1640995200,
    "l1_tx_count": 10,
    "l2_tx_count": 100,
    "root_hash": "0x...",
    "status": "verified",
    "commit_tx_hash": "0x...",
    "commit_tx_status": "confirmed",
    "prove_tx_hash": "0x...",
    "prove_tx_status": "pending",
    "execute_tx_hash": null,
    "execute_tx_status": "not_submitted",
    "data_availability": {
      "commit_data_available": true,
      "proof_data_available": true,
      "execute_data_available": false
    }
  },
  "id": 1
}
```

### 7.3 Coordinator API Examples

#### Submit Partial Signature with Enhanced Validation

```json
// Request
{
  "jsonrpc": "2.0",
  "method": "coordinator_submitPartialSignature",
  "params": {
    "batch_id": "0x123...",
    "signature": {
      "r": "0x1234567890abcdef...",
      "s": "0xfedcba0987654321...",
      "v": 27
    },
    "signer_address": "0xabcdef1234567890...",
    "timestamp": 1640995200,
    "nonce": "0x1",
    "metadata": {
      "signature_type": "ecdsa",
      "hash_algorithm": "keccak256"
    }
  },
  "id": 1
}

// Successful Response
{
  "jsonrpc": "2.0",
  "result": {
    "accepted": true,
    "signature_id": "0x...",
    "batch_status": "partially_signed",
    "required_signatures": 5,
    "received_signatures": 3
  },
  "id": 1
}
```

### 7.4 Transaction Submission

```json
// Request
{
  "jsonrpc": "2.0",
  "method": "eth_sendRawTransaction",
  "params": ["0x..."],
  "id": 1
}

// Response
{
  "jsonrpc": "2.0",
  "result": "0x...",
  "id": 1
}
```

### 7.5 State Query

```json
// Request
{
  "jsonrpc": "2.0",
  "method": "eth_getBalance",
  "params": ["0x...", "latest"],
  "id": 1
}

// Response
{
  "jsonrpc": "2.0",
  "result": "0x...",
  "id": 1
}
```

### 7.6 ZKSync-Specific Method

```json
// Request
{
  "jsonrpc": "2.0",
  "method": "zks_estimateFee",
  "params": [{
    "from": "0x...",
    "to": "0x...",
    "value": "0x...",
    "data": "0x..."
  }],
  "id": 1
}

// Response
{
  "jsonrpc": "2.0",
  "result": {
    "gas_limit": "0x...",
    "gas_per_pubdata_limit": "0x...",
    "max_fee_per_gas": "0x...",
    "max_priority_fee_per_gas": "0x..."
  },
  "id": 1
}
```

### 7.7 Via-Specific Method

```json
// Request
{
  "jsonrpc": "2.0",
  "method": "via_getProtocolVersion",
  "params": [],
  "id": 1
}

// Response
{
  "jsonrpc": "2.0",
  "result": {
    "major": 0,
    "minor": 26,
    "patch": 0
  },
  "id": 1
}
```

```json
// Request
{
  "jsonrpc": "2.0",
  "method": "via_getBridgeAddress",
  "params": [],
  "id": 1
}

// Response
{
  "jsonrpc": "2.0",
  "result": "bc1qxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "id": 1
}
```

## 8. Configuration Parameters

The API server can be configured with the following parameters:

### 8.1 Basic Configuration

- **HTTP Port**: Port for the HTTP JSON-RPC server
- **WebSocket Port**: Port for the WebSocket server
- **API Namespaces**: List of enabled API namespaces (eth, zks, via, web3, net, debug)
- **Filters Limit**: Maximum number of filters per client
- **Subscriptions Limit**: Maximum number of subscriptions per client
- **Batch Request Size Limit**: Maximum number of requests in a batch
- **Response Body Size Limit**: Maximum size of response bodies
- **WebSocket Requests Per Minute Limit**: Rate limiting for WebSocket requests
- **VM Concurrency Limit**: Maximum number of concurrent VM executions
- **Cache Sizes**: Sizes for various caches (factory deps, initial writes, latest values)

### 8.2 Enhanced Bitcoin RPC Configuration

```toml
[bitcoin_rpc]
# Standard RPC configuration
url = "http://user:password@localhost:8332"
timeout = 30
max_retries = 3

# Wallet-specific configuration
[bitcoin_rpc.wallets]
default = "http://user:password@localhost:8332/wallet/default"
bridge = "http://user:password@localhost:8332/wallet/bridge"
coordinator = "http://user:password@localhost:8332/wallet/coordinator"

# Enhanced features
enable_wallet_operations = true
enable_mempool_monitoring = true
mempool_refresh_interval = 10  # seconds
utxo_cache_ttl = 300  # seconds
```

### 8.3 Server Components Configuration

```toml
[server.components]
# Enable specific components
enabled = ["api", "mempool", "state_keeper", "btc_sender"]

# Component-specific settings
[server.components.api]
enable_debug_namespace = false
enable_coordinator_api = true

[server.components.btc_sender]
enable_wallet_operations = true
batch_size = 10

[server.components.mempool]
max_pending_transactions = 10000
cleanup_interval = 60
```

### 8.4 Security Configuration

```toml
[security]
# API authentication
require_api_key = true
api_key_header = "X-API-Key"

# Request signing
require_request_signing = true
signature_header = "X-Signature"
timestamp_header = "X-Timestamp"
max_timestamp_drift = 300  # seconds

# Rate limiting
requests_per_minute = 1000
burst_limit = 100

# Coordinator API security
[security.coordinator]
require_signature_validation = true
max_signature_age = 600  # seconds
require_nonce = true
```

## 9. Security Considerations

The RPC/API Layer implements several enhanced security measures:

### 9.1 Standard Security Measures

- **CORS**: Cross-Origin Resource Sharing configuration for HTTP requests
- **Rate Limiting**: Limits on request rates to prevent DoS attacks
- **Size Limits**: Limits on request and response sizes
- **Concurrency Limits**: Limits on concurrent VM executions
- **Filters and Subscriptions Limits**: Limits on the number of filters and subscriptions per client

### 9.2 Enhanced Security Features

#### Cryptographic Validation
- **Signature Verification**: All coordinator API requests require cryptographic signatures
- **Nonce Management**: Prevents replay attacks through nonce validation
- **Timestamp Validation**: Ensures requests are within acceptable time windows

#### Authentication and Authorization
- **API Key Management**: Secure API key-based authentication
- **Role-Based Access Control**: Different access levels for different API endpoints
- **Signer Authorization**: Validates that signers are authorized for specific operations

#### Request Security
- **Request Signing**: All sensitive requests must be cryptographically signed
- **Payload Validation**: Enhanced validation of request payloads
- **Input Sanitization**: Comprehensive input validation and sanitization

#### Network Security
- **TLS/SSL Enforcement**: Secure communication channels
- **IP Whitelisting**: Optional IP-based access control
- **DDoS Protection**: Enhanced protection against distributed denial-of-service attacks

### 9.3 Bitcoin Integration Security

- **Wallet Isolation**: Separate wallet operations for enhanced security
- **UTXO Validation**: Cryptographic validation of UTXO ownership
- **Transaction Verification**: Enhanced Bitcoin transaction validation
- **Private Key Management**: Secure handling of Bitcoin private keys

## 10. Error Handling and Response Formats

### 10.1 Standard JSON-RPC Errors

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32602,
    "message": "Invalid params",
    "data": {
      "reason": "Missing required parameter",
      "parameter": "wallet_address"
    }
  },
  "id": 1
}
```

### 10.2 Bitcoin RPC Specific Errors

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32001,
    "message": "Bitcoin RPC Error",
    "data": {
      "bitcoin_error_code": -5,
      "bitcoin_error_message": "Invalid or non-wallet transaction id",
      "wallet_path": "/wallet/main"
    }
  },
  "id": 1
}
```

### 10.3 Coordinator API Errors

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32003,
    "message": "Signature validation failed",
    "data": {
      "validation_errors": [
        "Invalid signature format",
        "Timestamp too old"
      ],
      "retry_after": 60
    }
  },
  "id": 1
}
```

## 11. Integration Examples

### 11.1 Wallet Integration with Enhanced Bitcoin Support

```javascript
// Initialize client with wallet-specific RPC
const client = new ViaRpcClient({
  url: 'http://localhost:3050',
  bitcoinWallet: 'main_wallet'
});

// Get wallet-specific UTXOs
const utxos = await client.via_getWalletUTXOs({
  wallet_address: 'bc1q...',
  min_confirmations: 1
});

// Monitor mempool for fee estimation
const mempoolInfo = await client.via_getBitcoinMempoolInfo();
const recommendedFee = calculateOptimalFee(mempoolInfo);
```

### 11.2 Coordinator Integration with Enhanced Validation

```javascript
// Submit partial signature with enhanced validation
const signature = await signBatch(batchData, privateKey);

const result = await client.coordinator_submitPartialSignature({
  batch_id: batchId,
  signature: signature,
  signer_address: signerAddress,
  timestamp: Math.floor(Date.now() / 1000),
  nonce: await getNonce(signerAddress)
});
```

### 11.3 Server Component Management

```bash
# Start server with specific components
via_server --components api,mempool,btc_sender \
  --bitcoin-wallet-path /wallet/main \
  --enable-coordinator-api \
  --require-signature-validation
```

## 12. Conclusion

The enhanced RPC/API Layer in Via L2 provides a comprehensive and secure interface for interacting with the rollup. The recent improvements include:

**Enhanced Bitcoin Integration:**
- Wallet-specific RPC URL support for targeted Bitcoin operations
- New mempool information API for better fee estimation
- Improved UTXO management with wallet-specific queries

**Security Enhancements:**
- Cryptographic validation for coordinator API endpoints
- Enhanced signature verification and authentication
- Improved request validation and error handling

**Operational Improvements:**
- Enhanced database queries with better data availability
- Selective server component execution through CLI flags
- Improved error handling and response formats

**Developer Experience:**
- Comprehensive configuration options for different use cases
- Clear integration examples and usage guidelines
- Enhanced documentation with practical examples

The layer maintains full compatibility with standard Ethereum JSON-RPC methods while providing advanced Bitcoin-specific functionality essential for the Via L2 Bitcoin ZK-Rollup. The enhanced security measures ensure safe operation in production environments, while the improved configuration options provide flexibility for different deployment scenarios.