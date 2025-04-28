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
- **Bridge Information**: `via_getBridgeAddress`, `via_getVerifierAddresses`
- **Protocol Information**: `via_getProtocolVersion`, `via_getCurrentProtocolVersion`
- **Block and Transaction Details**: `via_getBlockDetails`, `via_getTransactionDetails`, `via_getRawBlockTransactions`
- **Deposit and Withdrawal**: `via_estimateDepositFee`, `via_getWithdrawalStatus`

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

### 4.1 Server Configuration

The API server is configured and initialized in the node builder:

- **HTTP Server**: Handles standard JSON-RPC requests
- **WebSocket Server**: Handles WebSocket connections for subscriptions
- **Configuration Parameters**:
  - Ports for HTTP and WebSocket servers
  - API namespaces to enable
  - Limits for filters, subscriptions, and batch requests
  - Response size limits
  - Rate limiting for WebSocket requests

### 4.2 Request Routing

1. Incoming requests are received by the HTTP or WebSocket server
2. Requests are parsed and validated
3. The appropriate namespace and method handler is identified
4. The method handler processes the request, accessing the blockchain state as needed
5. The response is formatted and returned to the client

### 4.3 State Access

The API server accesses blockchain state through:

- **Connection Pool**: Manages database connections for efficient state access
- **Storage DAL (Data Access Layer)**: Provides methods for querying blockchain data
- **Caching**: Implements caching for frequently accessed data to improve performance

### 4.4 WebSocket Handling

WebSocket connections are managed by:

- **Subscription Manager**: Tracks active subscriptions
- **Event Notifiers**: Monitor for new blocks, transactions, and logs
- **Notification Dispatcher**: Sends notifications to subscribed clients

## 5. Interactions with Other Components

### 5.1 Mempool

- The API server submits transactions to the Mempool via the `tx_sender` component
- The Mempool validates and stores pending transactions
- The API server can query pending transactions from the Mempool

### 5.2 State Management

- The API server queries blockchain state from the database
- State queries include account balances, storage, code, and transaction data
- The API server can execute view calls against the current state

### 5.3 Block Information

- The API server retrieves block information from the database
- Block queries include block headers, transactions, receipts, and logs
- The API server tracks the latest sealed block number for consistent responses

## 6. API Method Examples

### 6.1 Transaction Submission

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

### 6.2 State Query

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

### 6.3 ZKSync-Specific Method

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

### 6.4 Via-Specific Method

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

## 7. Configuration Parameters

The API server can be configured with the following parameters:

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

## 8. Security Considerations

The RPC/API Layer implements several security measures:

- **CORS**: Cross-Origin Resource Sharing configuration for HTTP requests
- **Rate Limiting**: Limits on request rates to prevent DoS attacks
- **Size Limits**: Limits on request and response sizes
- **Concurrency Limits**: Limits on concurrent VM executions
- **Filters and Subscriptions Limits**: Limits on the number of filters and subscriptions per client

## 9. Conclusion

The RPC/API Layer in Via L2 provides a comprehensive interface for interacting with the rollup. It implements standard Ethereum JSON-RPC methods for compatibility with existing tools and libraries, while also offering ZKSync-specific and Via-specific methods for advanced functionality. The layer is designed for performance, security, and reliability, with features like connection pooling, caching, and rate limiting.

The addition of the `via` namespace provides Bitcoin-specific functionality that is essential for interacting with the Via L2 Bitcoin ZK-Rollup, including methods for querying Bitcoin blocks and transactions, retrieving bridge information, and managing deposits and withdrawals.