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

## 7. Conclusion

The fee mechanism in Via L2 Bitcoin ZK-Rollup is designed to balance efficiency, fairness, and economic security. By separating computational costs from data publication costs and implementing a flexible fee model, the system can adapt to different network conditions and optimize for both user experience and operator economics.

### 7.1 Recent Fee Adjustments

The base fee has been increased to enhance economic security and prevent spam transactions. This adjustment ensures that:

1. **Transaction Costs Reflect Value**: The fee structure better aligns the cost of transactions with the value they provide to the network
2. **Spam Prevention**: Higher base fees discourage spam transactions that could congest the network
3. **Operator Sustainability**: Fees adequately compensate operators for their services, ensuring long-term sustainability
4. **Network Security**: Proper economic incentives contribute to the overall security of the network

The fee model continues to evolve to better align incentives between users, operators, and the overall network security. Future improvements may include more dynamic fee adjustment mechanisms and optimizations for specific transaction types.