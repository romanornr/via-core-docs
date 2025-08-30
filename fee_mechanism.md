# Via L2 Bitcoin ZK-Rollup: Fee Mechanism

## 1. Introduction

The fee mechanism in Via L2 Bitcoin ZK-Rollup is responsible for calculating, collecting, and distributing transaction fees. This document provides a comprehensive overview of how fees work in the Via L2 system, including the fee model, calculation methods, and implementation details.

## 2. Fee Model Overview

Via L2 implements a fee model that accounts for both L2 computation costs and L1 data publication costs. The fee model has evolved through different versions to optimize for efficiency and fairness.

### 2.1 Fee Model Versions

The system supports two fee model versions:

1. **V1 (Legacy)**: In this model, the pubdata price is pegged to the L1 gas price. The fair L2 gas price only includes the cost of computation/proving and does not include overhead for closing batches.

2. **V2 (Current)**: In this model, the pubdata price can be independent of the L1 gas price. The fair L2 gas price includes both the cost of computation/proving and the overhead for closing batches.

### 2.2 Fee Components

Transaction fees in Via L2 consist of the following components:

1. **Computational Gas**: Covers the cost of executing the transaction on L2, including VM operations, storage access, and cryptographic operations.

2. **Pubdata Cost**: Covers the cost of publishing transaction data to L1 (Bitcoin). This is a significant component of the overall fee, as L1 data availability is a key security feature of the rollup.

3. **Batch Overhead**: Covers the fixed costs associated with processing a batch, such as verification and commitment.

## 3. Fee Calculation

### 3.1 Transaction Fee Structure

Each transaction in Via L2 includes a fee specification with the following parameters:

```rust
pub struct Fee {
    /// The limit of gas that are to be spent on the actual transaction.
    pub gas_limit: U256,
    /// ZKsync version of EIP1559 maxFeePerGas.
    pub max_fee_per_gas: U256,
    /// ZKsync version of EIP1559 maxPriorityFeePerGas.
    pub max_priority_fee_per_gas: U256,
    /// The maximal gas per pubdata byte the user agrees to.
    pub gas_per_pubdata_limit: U256,
}
```

These parameters determine the maximum fee a user is willing to pay for their transaction.

### 3.2 Batch Fee Input

The system calculates fees based on batch fee inputs, which can be one of two types:

1. **L1Pegged**: Used in the V1 fee model, where pubdata price is derived from the L1 gas price.

```rust
pub struct L1PeggedBatchFeeModelInput {
    /// Fair L2 gas price to provide
    pub fair_l2_gas_price: u64,
    /// The L1 gas price to provide to the VM.
    pub l1_gas_price: u64,
}
```

2. **PubdataIndependent**: Used in the V2 fee model, where pubdata price is independent of the L1 gas price.

```rust
pub struct PubdataIndependentBatchFeeModelInput {
    /// Fair L2 gas price to provide
    pub fair_l2_gas_price: u64,
    /// Fair pubdata price to provide.
    pub fair_pubdata_price: u64,
    /// The L1 gas price to provide to the VM.
    pub l1_gas_price: u64,
}
```

### 3.3 Fee Calculation Process

The fee calculation process involves several steps:

1. **Gas Price Determination**:
   - The base fee per gas is determined based on network conditions
   - The base fee has been increased to ensure economic security and prevent spam transactions
   - For V2 model, the pubdata price is determined separately

2. **Gas Usage Calculation**:
   - Computational gas used is tracked during transaction execution
   - Pubdata usage is tracked separately

3. **Total Fee Calculation**:
   - For V1 model: `total_fee = computational_gas * fair_l2_gas_price + pubdata_bytes * l1_gas_price * L1_GAS_PER_PUBDATA_BYTE`
   - For V2 model: `total_fee = computational_gas * fair_l2_gas_price + pubdata_bytes * fair_pubdata_price`

### 3.4 Fee Adjustment Factors

The fee model includes several adjustment factors to account for different network conditions:

1. **Compute Overhead Part**: Represents the possibility that a batch can be sealed due to computation resource limits.

2. **Pubdata Overhead Part**: Represents the possibility that a batch can be sealed due to pubdata limits.

3. **Batch Overhead L1 Gas**: The fixed amount of L1 gas used as overhead for each batch.

4. **Base Token Conversion Ratio**: Used for token conversion calculations with the base token (BTC).

## 4. Implementation Details

### 4.1 Core Components

The fee mechanism is implemented across several components:

1. **Fee Model Library** (`core/node/fee_model/`):
   - Defines the fee model and calculation methods
   - Provides batch fee input for transaction execution

2. **VM Implementation** (`core/lib/multivm/src/versions/vm_latest/`):
   - Tracks gas usage during transaction execution
   - Calculates refunds based on actual resource usage

3. **Bootloader** (`etc/multivm_bootloaders/`):
   - Deducts fees upfront before transaction execution
   - Handles refunds after transaction execution

4. **API Server** (`core/lib/zksync_core/src/api_server/`):
   - Provides fee estimation endpoints for users
   - Exposes fee parameters to clients

### 4.2 Fee Collection Process

The fee collection process occurs in the following steps:

1. **Fee Deduction**:
   - Fees are deducted upfront from the user's account before transaction execution
   - The bootloader handles this deduction as part of transaction processing

2. **Execution and Tracking**:
   - During execution, the VM tracks actual resource usage (computational gas and pubdata)
   - The `RefundsTracer` monitors resource usage for refund calculation

3. **Refund Calculation**:
   - After execution, the system calculates refunds based on the difference between pre-paid fees and actual usage
   - The `tx_body_refund` method in `RefundsTracer` calculates the appropriate refund

4. **Refund Processing**:
   - Refunds are processed by the bootloader and credited back to the user's account
   - The refund amount is stored in the transaction receipt

### 4.3 Fee Estimation

The system provides fee estimation through the API:

1. **Estimation Endpoints**:
   - `zks_estimateFee`: Estimates the fee for a transaction
   - `zks_getFeeParams`: Returns the current fee parameters
   - `zks_getBatchFeeInput`: Returns the current batch fee input
   - `via_estimateDepositFee`: Estimates the fee for a Bitcoin deposit

2. **Estimation Process**:
   - The system simulates the transaction execution to determine gas usage
   - It applies the current fee parameters to calculate the estimated fee
   - A safety margin is added to account for potential fluctuations

3. **Deposit Fee Estimation**:
   - For Bitcoin deposits, the system estimates both the L1 (Bitcoin) transaction fee and the L2 processing fee
   - The gas limit for deposits has been reduced to optimize costs while ensuring reliable execution
   - The gas price for deposits has been standardized to 120,000,000 wei to ensure consistent fee estimation

```typescript
// infrastructure/via/src/token.ts
async function estimateGasFee(l2RpcUrl: string, amount: number, receiverL2Address: string): Promise<bigint> {
    const l2Provider = new ethers.JsonRpcProvider(l2RpcUrl);
    
    // Calculate gas cost for the deposit transaction
    const gasCost = await l2Provider.estimateGas({
        from: receiverL2Address,
        to: receiverL2Address,
        value: 0,
        data: "0x"
    });
    
    // Hardcode the gas price to avoid issues during the Priority Id verification
    // const gasPrice = await l2Provider.getGasPrice();
    const gasPrice = BigInt(120_000_000);
    
    return (gasCost * gasPrice) / BigInt(10_000_000_000);
}
```

This approach ensures that:
- Deposit fees are predictable and consistent
- Gas parameters are optimized for the specific requirements of deposit transactions
- Users can accurately estimate the total cost of a deposit before initiating it

### 4.4 Fee Distribution

Collected fees are distributed as follows:

1. **Sequencer Fees**:
   - The sequencer (operator) receives fees as compensation for processing transactions and producing blocks
   - These fees cover the computational costs of running the sequencer

2. **L1 Publication Costs**:
   - A portion of the fees covers the cost of publishing data to Bitcoin through inscriptions
   - This ensures data availability and security of the rollup

## 5. Key Files and Modules

The fee mechanism is implemented across several files and modules:

1. **Fee Model Definition**:
   - `core/lib/types/src/fee_model.rs`: Defines the fee model types and structures
   - `core/lib/types/src/fee.rs`: Defines the fee structure for transactions

2. **Fee Calculation**:
   - `core/node/fee_model/src/lib.rs`: Implements the fee model and calculation logic
   - `core/lib/multivm/src/versions/vm_latest/tracers/refunds.rs`: Handles refund calculation

3. **Fee Collection**:
   - `core/lib/multivm/src/versions/vm_latest/bootloader_state/state.rs`: Manages transaction execution and fee collection
   - `core/lib/multivm/src/versions/vm_latest/bootloader_state/tx.rs`: Defines transaction representation for the bootloader

4. **Fee Estimation**:
   - API endpoints in the RPC layer provide fee estimation functionality

## 6. Configuration Parameters

The fee mechanism can be configured through several parameters:

1. **Fee Model Version**: Determines which fee model to use (V1 or V2)

2. **Base Fee**: The base fee per gas used for all transactions, which has been increased to ensure economic security

3. **Minimal L2 Gas Price**: The minimum acceptable gas price for L2 computation

4. **Compute Overhead Part**: Factor for computation overhead in batch sealing (0.0 to 1.0)

4. **Pubdata Overhead Part**: Factor for pubdata overhead in batch sealing (0.0 to 1.0)

5. **Batch Overhead L1 Gas**: Fixed L1 gas overhead for each batch

6. **Max Gas Per Batch**: Maximum gas limit per batch

7. **Max Pubdata Per Batch**: Maximum pubdata bytes per batch

## 6. Withdrawal Fee Strategy for Bitcoin Withdrawals

This release introduces a fee strategy dedicated to Bitcoin withdrawal transactions. The strategy ensures:
- Total transaction fee is estimated accurately and made evenly divisible across user outputs
- Fees are distributed equally among all withdrawal outputs
- Withdrawals that cannot cover their fair share of the fee are filtered out before signing
- Transactions are built with a fee rate that matches current network conditions within a strict tolerance

### 6.1 Strategy Pattern and Implementation

The fee logic is implemented using a Strategy pattern:

- Trait definition with default size-based fee estimation and an interface for applying fees:
  - [via_verifier/lib/via_musig2/src/fee.rs](via_verifier/lib/via_musig2/src/fee.rs:9)

- Withdrawal-specific strategy type and implementation:
  - [WithdrawalFeeStrategy](via_verifier/lib/via_musig2/src/fee.rs:44)
  - [impl FeeStrategy for WithdrawalFeeStrategy](via_verifier/lib/via_musig2/src/fee.rs:52)

- Transaction builder integrates the strategy when constructing withdrawal transactions with OP_RETURN:
  - [TransactionBuilder::build_transaction_with_op_return()](via_verifier/lib/via_musig2/src/transaction_builder.rs:61)

- Fee per user helper and empty-transaction guard:
  - [UnsignedBridgeTx::get_fee_per_user()](via_verifier/lib/via_verifier_types/src/transaction.rs:17)
  - [UnsignedBridgeTx::is_empty()](via_verifier/lib/via_verifier_types/src/transaction.rs:25)

- Constant OP_RETURN prefix used for withdrawal references:
  - [OP_RETURN_WITHDRAW_PREFIX](via_verifier/node/via_verifier_coordinator/src/sessions/withdrawal.rs:21)

### 6.2 Fee Estimation and Equal Distribution

- The strategy estimates the total fee as a function of inputs, outputs and fee rate (sat/vB) with a correction to ensure the fee is evenly divisible by the number of withdrawal outputs:
  - Remainder-safe calculation: Round up to the next multiple of the output count:
    - Let `r = base_fee % output_count`. If `r == 0` then `total_fee = base_fee`; else `total_fee = base_fee + (output_count - r)`. See [rust FeeStrategy::estimate_fee_sats()](https://github.com/vianetwork/via-core/blob/main/via_verifier/lib/via_musig2/src/fee.rs#L22).
  - Tests: divisibility and edge cases are covered by [rust test_estimate_fee_multiple_cases](https://github.com/vianetwork/via-core/blob/main/via_verifier/lib/via_musig2/src/fee.rs#L40).

- After total fee is determined, it is split equally across all user withdrawal outputs:
  - [impl FeeStrategy for WithdrawalFeeStrategy](https://github.com/vianetwork/via-core/blob/main/via_verifier/lib/via_musig2/src/fee.rs#L52)
  - Fixed fee per user calculation: [rust UnsignedBridgeTx::get_fee_per_user()](https://github.com/vianetwork/via-core/blob/main/via_verifier/lib/via_verifier_types/src/transaction.rs#L96)
  - Handles edge case where there are no withdrawal outputs (only change and OP_RETURN)
  - Empty/no-op transactions are not inserted into the UTXO manager: [rust TransactionBuilder::build_transaction_with_op_return()](https://github.com/vianetwork/via-core/blob/main/via_verifier/lib/via_musig2/src/transaction_builder.rs#L82)

### 6.3 Iterative Filtering of Uneconomical Withdrawals

- Algorithm outline (pseudocode):
  1. Estimate total fee for the current output set
  2. Compute fee_per_user = total_fee / outputs_count
  3. Filter out any outputs where output.value < fee_per_user
  4. If outputs were removed, repeat from step 1 with the smaller set
  5. When all remaining outputs can pay their fee share, subtract fee_per_user from each output value and finalize

- Code path:
  - [impl FeeStrategy for WithdrawalFeeStrategy](https://github.com/vianetwork/via-core/blob/main/via_verifier/lib/via_musig2/src/fee.rs#L52)

- Validations covered by tests:
  - Equal split across outputs: [test_fee_applied_equally_to_outputs()](https://github.com/vianetwork/via-core/blob/main/via_verifier/lib/via_musig2/src/fee.rs#L107)
  - Removal of too-small outputs: [test_small_output_is_removed()](https://github.com/vianetwork/via-core/blob/main/via_verifier/lib/via_musig2/src/fee.rs#L135)
  - No viable outputs edge case: [test_no_outputs_can_pay_fee()](https://github.com/vianetwork/via-core/blob/main/via_verifier/lib/via_musig2/src/fee.rs#L163)

### 6.4 Network Fee Rate Validation

Withdrawal sessions validate the fee rate used to construct the unsigned transaction against the current network fee rate:
- The used fee rate must be within ±1 sat/vB of the network's current estimate
- If the difference exceeds the tolerance, the session is rejected and not signed
- Withdrawal validation ensures each user receives the correct amount after fee deduction

Reference:
- Fee rate check: [via_verifier/node/via_verifier_coordinator/src/sessions/withdrawal.rs](https://github.com/vianetwork/via-core/blob/main/via_verifier/node/via_verifier_coordinator/src/sessions/withdrawal.rs#L369)
- User amount validation with clear variable naming: `user_should_receive = amount - fee_per_user`

### 6.5 Behavior with Insufficient Withdrawals

- Withdrawals with amount < fee_per_user are filtered out prior to signing
- If all withdrawals are filtered out, the resulting transaction will be empty (only OP_RETURN and change outputs). Such transactions are not broadcast, and the session is treated as a no-op:
  - [UnsignedBridgeTx::is_empty()](https://github.com/vianetwork/via-core/blob/main/via_verifier/lib/via_verifier_types/src/transaction.rs#L25)
- Empty transactions are not inserted into the UTXO manager to prevent tracking of non-existent UTXOs
  - Transaction builder checks if the bridge transaction is empty before UTXO insertion: [TransactionBuilder::build_transaction_with_op_return()](https://github.com/vianetwork/via-core/blob/main/via_verifier/lib/via_musig2/src/transaction_builder.rs#L82)

### 6.6 Integration into the Bridge Flow

- Coordinator builds withdrawal transactions with the fee strategy and OP_RETURN metadata for the proof reference:
  - [TransactionBuilder::build_transaction_with_op_return()](https://github.com/vianetwork/via-core/blob/main/via_verifier/lib/via_musig2/src/transaction_builder.rs#L61)

- For indexing, validation of origin and storage of bridge withdrawal metadata see:
  - Bridge docs: [bridge.md](bridge.md)

## 7. Advanced Fee Management Features

### 7.1 Fee Rate Limits per Network

Via L2 implements network-specific fee rate limits to prevent excessive Bitcoin transaction fees and ensure predictable costs across different network environments.

#### 7.1.1 FeeRateLimits Structure

The system uses a `FeeRateLimits` structure that caps Bitcoin transaction fee rates based on the network type:

```rust
pub struct FeeRateLimits {
    /// Maximum fee rate for mainnet (satoshis per vbyte)
    pub mainnet_max_fee_rate: u64,
    /// Maximum fee rate for testnet (satoshis per vbyte)
    pub testnet_max_fee_rate: u64,
    /// Maximum fee rate for regtest (satoshis per vbyte)
    pub regtest_max_fee_rate: u64,
}
```

#### 7.1.2 Network-Specific Limits

Different networks have different fee rate limits to account for their specific characteristics:

- **Mainnet**: Conservative limits to prevent excessive costs during high congestion
- **Testnet**: Higher limits to allow for testing scenarios
- **Regtest**: Very high limits for development and testing purposes

#### 7.1.3 Configuration

Fee rate limits are configured through the `via_btc_client.toml` configuration file:

```toml
[fee_strategy]
# Maximum fee rates per network (satoshis per vbyte)
mainnet_max_fee_rate = 100
testnet_max_fee_rate = 200
regtest_max_fee_rate = 1000

# Enable external API fallback for fee estimation
enable_external_api_fallback = true

# Mempool integration settings
use_mempool_info = true
mempool_fee_multiplier = 1.2
```

### 7.2 Enhanced Fee Estimation

The fee estimation system has been significantly enhanced to provide more accurate and reliable fee calculations.

#### 7.2.1 Multi-Source Fee Estimation

The system now uses multiple sources for fee estimation:

1. **Primary RPC-based estimation**: Uses Bitcoin node's `estimatesmartfee` RPC
2. **Mempool-based estimation**: Considers current mempool state and minimum fees
3. **External API fallback**: Falls back to external services when RPC estimation fails

#### 7.2.2 Network-Aware Fee Calculation

Fee estimation now considers network-specific limits and applies appropriate caps:

```rust
impl FeeEstimator {
    pub fn estimate_fee_rate(&self, network: Network, target_blocks: u32) -> Result<u64> {
        // Get base estimate from RPC
        let base_estimate = self.rpc_estimate_fee(target_blocks)?;
        
        // Apply network-specific limits
        let max_rate = self.get_network_max_rate(network);
        let capped_rate = std::cmp::min(base_estimate, max_rate);
        
        // Consider mempool state
        let mempool_adjusted = self.adjust_for_mempool(capped_rate)?;
        
        Ok(mempool_adjusted)
    }
}
```

#### 7.2.3 Fallback Mechanism

When primary RPC-based estimation fails or is disabled, the system automatically falls back to external APIs:

1. **Detection**: System detects RPC estimation failure or unavailability
2. **Fallback Activation**: External API estimation is triggered
3. **Validation**: External estimates are validated against network limits
4. **Application**: Validated estimates are applied with appropriate safety margins

### 7.3 External API Fallback

The Bitcoin client now supports fallback to external APIs for fee rate estimation, providing redundancy and reliability.

#### 7.3.1 Supported External APIs

The system can integrate with various external fee estimation APIs:

- **Mempool.space API**: Provides real-time fee estimates based on mempool analysis
- **BlockCypher API**: Offers fee estimation with different confirmation targets
- **Custom APIs**: Support for additional fee estimation services

#### 7.3.2 Fallback Configuration

External API fallback is configured in the `via_btc_client.toml` file:

```toml
[fee_strategy.external_apis]
# Enable external API fallback
enabled = true

# Primary external API endpoint
primary_api_url = "https://mempool.space/api/v1/fees/recommended"

# Secondary fallback API
secondary_api_url = "https://api.blockcypher.com/v1/btc/main"

# API timeout settings
timeout_seconds = 10
retry_attempts = 3

# Fallback behavior
fallback_on_rpc_failure = true
fallback_on_high_fees = true
```

#### 7.3.3 API Integration Process

The external API integration follows this process:

1. **Primary Estimation**: Attempt RPC-based fee estimation
2. **Failure Detection**: Detect if RPC estimation fails or returns unreasonable values
3. **API Selection**: Choose appropriate external API based on configuration
4. **Request Processing**: Make API request with proper error handling
5. **Validation**: Validate external estimates against network limits
6. **Application**: Apply validated estimates with safety margins

### 7.4 Mempool Integration

The system now integrates with Bitcoin mempool data to provide more accurate fee estimation and dynamic adjustment.

#### 7.4.1 BitcoinRpc::get_mempool_info() Method

A new trait method has been added to fetch Bitcoin mempool information:

```rust
pub trait BitcoinRpc {
    /// Get current mempool information
    async fn get_mempool_info(&self) -> Result<MempoolInfo, BitcoinRpcError>;
    
    // ... other methods
}

pub struct MempoolInfo {
    /// Current mempool size in bytes
    pub size: u64,
    /// Number of transactions in mempool
    pub tx_count: u32,
    /// Minimum fee rate in mempool (sat/vB)
    pub min_fee_rate: f64,
    /// Maximum fee rate in mempool (sat/vB)
    pub max_fee_rate: f64,
    /// Average fee rate in mempool (sat/vB)
    pub avg_fee_rate: f64,
}
```

#### 7.4.2 Dynamic Fee Adjustment

Fee calculation now considers current mempool state for dynamic adjustment:

```rust
impl FeeCalculator {
    pub fn calculate_dynamic_fee(&self, mempool_info: &MempoolInfo, target_blocks: u32) -> u64 {
        let base_fee = self.estimate_base_fee(target_blocks);
        
        // Adjust based on mempool congestion
        let congestion_multiplier = self.calculate_congestion_multiplier(mempool_info);
        let adjusted_fee = (base_fee as f64 * congestion_multiplier) as u64;
        
        // Ensure minimum fee requirements
        let min_fee = std::cmp::max(mempool_info.min_fee_rate as u64, 1);
        
        std::cmp::max(adjusted_fee, min_fee)
    }
}
```

#### 7.4.3 Mempool-Based Fee Strategies

The system supports different mempool-based fee strategies:

1. **Conservative**: Uses higher percentiles of mempool fee distribution
2. **Standard**: Uses median mempool fees with moderate adjustments
3. **Aggressive**: Uses lower percentiles for faster confirmation

### 7.5 Configuration Updates

#### 7.5.1 Enhanced via_btc_client.toml Configuration

The configuration file now includes comprehensive fee strategy settings:

```toml
[fee_strategy]
# Fee estimation strategy: "rpc", "mempool", "external", "hybrid"
strategy = "hybrid"

# Network-specific fee rate limits (sat/vB)
[fee_strategy.limits]
mainnet_max_fee_rate = 100
testnet_max_fee_rate = 200
regtest_max_fee_rate = 1000

# RPC-based estimation settings
[fee_strategy.rpc]
enabled = true
default_target_blocks = 6
max_target_blocks = 144
min_fee_rate = 1

# Mempool integration settings
[fee_strategy.mempool]
enabled = true
update_interval_seconds = 30
congestion_threshold = 0.8
fee_multiplier = 1.2

# External API settings
[fee_strategy.external_api]
enabled = true
primary_url = "https://mempool.space/api/v1/fees/recommended"
secondary_url = "https://api.blockcypher.com/v1/btc/main"
timeout_seconds = 10
retry_attempts = 3

# Fallback behavior
[fee_strategy.fallback]
on_rpc_failure = true
on_high_fees = true
on_api_timeout = true
```

#### 7.5.2 Environment-Specific Configuration

Different environments can have tailored fee configurations:

**Development Environment:**
```toml
[fee_strategy]
strategy = "rpc"
regtest_max_fee_rate = 10000
enable_external_api_fallback = false
```

**Production Environment:**
```toml
[fee_strategy]
strategy = "hybrid"
mainnet_max_fee_rate = 50
enable_external_api_fallback = true
mempool_integration = true
```

### 7.6 Behavioral Changes

#### 7.6.1 Stricter Fee Validation

The system now implements stricter fee validation and capping mechanisms:

1. **Pre-transaction Validation**: Fees are validated before transaction creation
2. **Network-Aware Capping**: Fees are capped based on network-specific limits
3. **Reasonableness Checks**: Extreme fee rates are rejected with appropriate errors

#### 7.6.2 Robust Fee Estimation

Fee estimation is now more robust with multiple fallback options:

1. **Primary Method**: RPC-based estimation with mempool consideration
2. **Secondary Method**: External API-based estimation
3. **Tertiary Method**: Static fallback based on network defaults
4. **Error Handling**: Comprehensive error handling with user-friendly messages

#### 7.6.3 Adaptive Fee Strategies

The system adapts fee strategies based on network conditions:

- **Low Congestion**: Uses conservative estimates with minimal premiums
- **Medium Congestion**: Applies moderate fee multipliers
- **High Congestion**: Uses aggressive strategies with external API fallback

## 8. Best Practices for Fee Configuration

### 8.1 Network-Specific Recommendations

#### 8.1.1 Mainnet Configuration
```toml
[fee_strategy]
strategy = "hybrid"
mainnet_max_fee_rate = 100  # Conservative limit
enable_external_api_fallback = true
mempool_integration = true
```

#### 8.1.2 Testnet Configuration
```toml
[fee_strategy]
strategy = "mempool"
testnet_max_fee_rate = 200  # Higher limit for testing
enable_external_api_fallback = true
```

#### 8.1.3 Development Configuration
```toml
[fee_strategy]
strategy = "rpc"
regtest_max_fee_rate = 1000  # Very high limit for development
enable_external_api_fallback = false
```

### 8.2 Monitoring and Alerting

#### 8.2.1 Fee Rate Monitoring

Monitor fee rates to ensure they remain within acceptable ranges:

```bash
# Check current fee estimates
curl -s "https://mempool.space/api/v1/fees/recommended"

# Monitor mempool size
bitcoin-cli getmempoolinfo
```

#### 8.2.2 Alert Thresholds

Set up alerts for:
- Fee rates exceeding network limits
- External API failures
- Mempool congestion levels
- Fee estimation errors

### 8.3 Performance Optimization

#### 8.3.1 Caching Strategies

Implement caching for fee estimates to reduce API calls:

```toml
[fee_strategy.caching]
enabled = true
cache_duration_seconds = 60
max_cache_entries = 1000
```

#### 8.3.2 Batch Processing

Process multiple fee estimations in batches to improve efficiency:

```toml
[fee_strategy.batching]
enabled = true
batch_size = 10
batch_timeout_ms = 500
```

## 9. Troubleshooting Fee-Related Issues

### 9.1 Common Issues and Solutions

#### 9.1.1 High Fee Rates

**Problem**: Fee rates are unexpectedly high
**Solutions**:
1. Check network congestion levels
2. Verify fee rate limits in configuration
3. Enable external API fallback
4. Adjust mempool fee multipliers

#### 9.1.2 Fee Estimation Failures

**Problem**: Fee estimation consistently fails
**Solutions**:
1. Verify Bitcoin node connectivity
2. Check external API endpoints
3. Review configuration parameters
4. Enable debug logging for fee estimation

#### 9.1.3 Mempool Integration Issues

**Problem**: Mempool data is not being used effectively
**Solutions**:
1. Verify mempool RPC access
2. Check mempool update intervals
3. Adjust congestion thresholds
4. Review mempool fee multipliers

### 9.2 Debugging Tools

#### 9.2.1 Fee Estimation Debug Commands

```bash
# Test fee estimation
via-btc-client estimate-fee --target-blocks 6 --network mainnet

# Check mempool status
via-btc-client mempool-info --network mainnet

# Test external API fallback
via-btc-client test-external-api --api-url "https://mempool.space/api/v1/fees/recommended"
```

#### 9.2.2 Configuration Validation

```bash
# Validate fee configuration
via-btc-client validate-config --config-file via_btc_client.toml

# Test fee limits
via-btc-client test-fee-limits --network mainnet --fee-rate 150
```

### 9.3 Logging and Monitoring

#### 9.3.1 Fee-Related Log Messages

Enable detailed logging for fee-related operations:

```toml
[logging]
level = "debug"
modules = ["fee_estimation", "mempool_integration", "external_api"]
```

#### 9.3.2 Metrics Collection

Monitor key fee-related metrics:
- Fee estimation success rates
- External API response times
- Mempool update frequencies
- Fee rate distributions

## 10. Conclusion

The enhanced fee mechanism in Via L2 Bitcoin ZK-Rollup provides a robust, flexible, and reliable system for managing transaction fees. The new features include:

1. **Network-Specific Fee Rate Limits**: Prevent excessive fees across different Bitcoin networks
2. **Enhanced Fee Estimation**: Multi-source estimation with intelligent fallback mechanisms
3. **External API Integration**: Reliable fallback to external fee estimation services
4. **Mempool Integration**: Dynamic fee adjustment based on real-time network conditions
5. **Comprehensive Configuration**: Flexible configuration options for different deployment scenarios

### 10.1 Key Benefits

The enhanced fee mechanism provides several key benefits:

1. **Predictable Costs**: Network-specific limits ensure fees remain within acceptable ranges
2. **High Reliability**: Multiple fallback mechanisms prevent fee estimation failures
3. **Dynamic Adaptation**: Real-time adjustment based on network conditions
4. **Operational Flexibility**: Comprehensive configuration options for different use cases
5. **Improved User Experience**: More accurate fee estimates and better cost predictability

### 10.2 Future Enhancements

The fee mechanism continues to evolve with planned improvements including:

1. **Machine Learning Integration**: AI-powered fee prediction based on historical data
2. **Cross-Chain Fee Optimization**: Coordination with other blockchain networks
3. **Advanced Mempool Analytics**: More sophisticated mempool analysis and prediction
4. **User-Customizable Strategies**: Allow users to define custom fee strategies
5. **Real-Time Fee Adjustment**: Dynamic fee adjustment during transaction processing

The fee model continues to balance efficiency, fairness, and economic security while providing the flexibility needed for different network conditions and use cases.