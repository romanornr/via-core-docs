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
// core/lib/types/src/fee_model.rs
/// Pubdata price may be independent from L1 gas price.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub struct PubdataIndependentBatchFeeModelInput {
    /// Fair L2 gas price to provide
    pub fair_l2_gas_price: u64,
    /// Fair pubdata price to provide.
    pub fair_pubdata_price: u64,
    /// The L1 gas price to provide to the VM. Even if some of the VM versions may not use this value, it is still maintained for backward compatibility.
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

1. **Via Fee Model** (`core/node/via_fee_model/`):
   - The Via-specific batch fee input providers: `ViaMainNodeFeeInputProvider` (sequencer) and `ViaApiFeeInputProvider` (API server)
   - Derives the L2 gas price from the Bitcoin fee rate. The core formula (`lib.rs`):

   ```rust
   // core/node/via_fee_model/src/lib.rs
    // Todo: rename "batch_overhead_l1_gas" to "total_inscription_gas_vbyte"
    let inscriptions_cost_satoshi = config.batch_overhead_l1_gas * l1_gas_price;
    // Scale the inscriptions_cost_satoshi to 18 decimals
    let gas_price_satoshi = inscriptions_cost_satoshi * 10_000_000_000 / config.max_gas_per_batch;
    // The "minimal_l2_gas_price" calculated from the operational cost to publish and verify block.
    let fair_l2_gas_price = gas_price_satoshi + config.minimal_l2_gas_price;
   ```

   Here `l1_gas_price` is the Bitcoin fee rate (sat/vB) supplied by the gas adjuster in `src/l1_gas_price/gas_adjuster/`, so L2 pricing tracks real inscription costs. `batch_overhead_l1_gas` is (despite the inherited name) the estimated inscription vbytes per batch.

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
   - ~~`via_estimateDepositFee`~~: **Does not exist** in the codebase (fabricated by doc generator)

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

The fee mechanism is configured through fields on `StateKeeperConfig`. These are the fee-related fields, verbatim from the struct definition (the struct also contains sealing and VM fields not shown here):

```rust
// core/lib/config/src/configs/chain.rs (StateKeeperConfig, fee-related fields)
    /// The minimal acceptable L2 gas price, i.e. the price that should include the cost of computation/proving as well
    /// as potentially premium for congestion.
    pub minimal_l2_gas_price: u64,
    /// The constant that represents the possibility that a batch can be sealed because of overuse of computation resources.
    /// It has range from 0 to 1. If it is 0, the compute will not depend on the cost for closing the batch.
    /// If it is 1, the gas limit per batch will have to cover the entire cost of closing the batch.
    pub compute_overhead_part: f64,
    /// The constant that represents the possibility that a batch can be sealed because of overuse of pubdata.
    /// It has range from 0 to 1. If it is 0, the pubdata will not depend on the cost for closing the batch.
    /// If it is 1, the pubdata limit per batch will have to cover the entire cost of closing the batch.
    pub pubdata_overhead_part: f64,
    /// The constant amount of L1 gas that is used as the overhead for the batch. It includes the price for batch verification, etc.
    pub batch_overhead_l1_gas: u64,
    /// The maximum amount of gas that can be used by the batch. This value is derived from the circuits limitation per batch.
    pub max_gas_per_batch: u64,
    /// The maximum amount of pubdata that can be used by the batch.
    /// This variable should not exceed:
    /// - 128kb for calldata-based rollups
    /// - 120kb * n, where `n` is a number of blobs for blob-based rollups
    /// - the DA layer's blob size limit for the DA layer-based validiums
    /// - 100 MB for the object store-based or no-da validiums
    pub max_pubdata_per_batch: u64,

    /// The version of the fee model to use.
    pub fee_model_version: FeeModelVersion,
```

For reference, the values used by `StateKeeperConfig::for_tests()` in the same file (they mirror the localhost environment): `compute_overhead_part: 0.0`, `pubdata_overhead_part: 1.0`, `batch_overhead_l1_gas: 800_000`, `max_gas_per_batch: 200_000_000`, `max_pubdata_per_batch: 100_000`, `minimal_l2_gas_price: 100000000`, `fee_model_version: FeeModelVersion::V2`.

In summary:

1. **Fee Model Version** (`fee_model_version`): Determines which fee model to use (V1 or V2)

2. **Minimal L2 Gas Price** (`minimal_l2_gas_price`): The minimum acceptable gas price for L2 computation

3. **Compute Overhead Part** (`compute_overhead_part`): Factor for computation overhead in batch sealing (0.0 to 1.0)

4. **Pubdata Overhead Part** (`pubdata_overhead_part`): Factor for pubdata overhead in batch sealing (0.0 to 1.0)

5. **Batch Overhead L1 Gas** (`batch_overhead_l1_gas`): Fixed L1 gas overhead for each batch (for Via, the estimated inscription vbytes per batch, see section 4.1)

6. **Max Gas Per Batch** (`max_gas_per_batch`): Maximum gas limit per batch

7. **Max Pubdata Per Batch** (`max_pubdata_per_batch`): Maximum pubdata bytes per batch

## 6. Withdrawal Fee Strategy for Bitcoin Withdrawals

This release introduces a fee strategy dedicated to Bitcoin withdrawal transactions. The strategy ensures:
- Total transaction fee is estimated accurately and made evenly divisible across user outputs
- Fees are distributed equally among all withdrawal outputs
- Withdrawals that cannot cover their fair share of the fee are filtered out before signing
- Transactions are built with a fee rate that matches current network conditions within a strict tolerance

### 6.1 Strategy Pattern and Implementation

The fee logic is implemented using a Strategy pattern:

- Trait definition with default size-based fee estimation and an interface for applying fees: `via_verifier/lib/via_musig2/src/fee.rs`
- Withdrawal-specific strategy type and implementation: `WithdrawalFeeStrategy` and `impl FeeStrategy for WithdrawalFeeStrategy` in the same file
- Transaction builder integrates the strategy when constructing withdrawal transactions with OP_RETURN: `TransactionBuilder::build_transaction_with_op_return()` (`via_verifier/lib/via_musig2/src/transaction_builder.rs`)
- Fee per user helper and empty-transaction guard: `UnsignedBridgeTx::get_fee_per_user()` and `UnsignedBridgeTx::is_empty()` (`via_verifier/lib/via_verifier_types/src/transaction.rs`)
- Constant OP_RETURN prefix used for withdrawal references: `const OP_RETURN_WITHDRAW_PREFIX: &[u8] = b"VIA_WI";` (`via_verifier/node/via_verifier_coordinator/src/sessions/withdrawal.rs`, also duplicated in `core/lib/via_btc_client/src/indexer/parser.rs`)

The trait, including its default `estimate_fee` implementation, and the strategy type:

```rust
// via_verifier/lib/via_musig2/src/fee.rs
pub trait FeeStrategy: Send + Sync {
    fn estimate_fee(
        &self,
        input_count: u32,
        output_count: u32,
        fee_rate: u64,
    ) -> anyhow::Result<Amount> {
        let input_size =
            (WITNESS_OVERHEAD + INPUT_BASE_SIZE + INPUT_WITNESS_SIZE) * u64::from(input_count);
        // approximate size per output +2 (+1 for potential change)
        let output_size = OUTPUT_SIZE * u64::from(output_count + 1);

        let total_size = TX_OVERHEAD + input_size + output_size + OP_RETURN_SIZE;
        let fee = fee_rate * total_size;

        // Ensure fee is divisible by output_count to avoid decimals when splitting
        let output_count_u64 = std::cmp::max(output_count, 1) as u64;
        let remainder = fee % output_count_u64;
        let adjusted_fee = if remainder == 0 {
            fee
        } else {
            fee + (output_count_u64 - remainder)
        };

        Ok(Amount::from_sat(adjusted_fee))
    }

    fn apply_fee_to_outputs(
        &self,
        outputs: Vec<TransactionOutput>,
        input_count: u32,
        fee_rate: u64,
    ) -> anyhow::Result<TransactionWithFee>;
}

pub struct WithdrawalFeeStrategy {}

impl WithdrawalFeeStrategy {
    pub fn new() -> Self {
        Self {}
    }
}
```

The size constants used by `estimate_fee`:

```rust
// via_verifier/lib/via_musig2/src/constants.rs
/// approximate size per input
///
/// base_size: 41 bytes
/// Previous txid: 32 bytes
/// Previous output index: 4 bytes
/// Script length: 1 byte (0x00)
/// ScriptSig: 0 bytes (empty)
/// Sequence: 4 bytes
pub const INPUT_BASE_SIZE: u64 = 41_u64;

/// Witness
/// MuSig2 signature: 65 bytes
/// Signature 64 bytes
/// Signature type 1 Byte
pub const INPUT_WITNESS_SIZE: u64 = 65_u64;
pub const OUTPUT_SIZE: u64 = 34_u64;
pub const OP_RETURN_SIZE: u64 = 68_u64;

// Transaction overhead (version + input_count + output_count + locktime)
pub const TX_OVERHEAD: u64 = 10;
// Witness overhead (marker + flag)
pub const WITNESS_OVERHEAD: u64 = 2;
```

The strategy is injected into the builder through `TransactionBuilderConfig`:

```rust
// via_verifier/lib/via_musig2/src/types.rs
#[derive(Clone)]
pub struct TransactionBuilderConfig {
    /// The fee strategy
    pub fee_strategy: Arc<dyn FeeStrategy>,
    /// The max tx weight
    pub max_tx_weight: u64,
    /// The max number of output to include in each transaction
    pub max_output_per_tx: usize,
    /// The OP_RETURN prefix
    pub op_return_prefix: Vec<u8>,
    /// Bridge address
    pub bridge_address: Address,
    /// The fee rate.
    pub default_fee_rate_opt: Option<u64>,
    /// The transaction fee rate
    pub default_available_utxos_opt: Option<Vec<(OutPoint, TxOut)>>,
    /// Unique data thta will be inscribed in all the bridge txs
    pub op_return_data_input_opt: Option<Vec<u8>>,
}
```

### 6.2 Fee Estimation and Equal Distribution

- The strategy estimates the total fee as a function of inputs, outputs and fee rate (sat/vB) with a correction to ensure the fee is evenly divisible by the number of withdrawal outputs. The method is `FeeStrategy::estimate_fee` (a default trait method, full body shown in section 6.1):
  - Remainder-safe calculation: round up to the next multiple of the output count. Let `remainder = fee % output_count`. If `remainder == 0` then the fee is unchanged; else `adjusted_fee = fee + (output_count - remainder)`.
  - Tests: divisibility and edge cases are covered by `test_estimate_fee_multiple_cases` in `via_verifier/lib/via_musig2/src/fee.rs`.

- After total fee is determined, it is split equally across all user withdrawal outputs inside `apply_fee_to_outputs` (full body in section 6.3). The `UnsignedBridgeTx` helper exposes the per-user fee after the transaction is built. Note the `- 2` accounts for the OP_RETURN output and the change output, so `withdrawals_count` is the number of user outputs:

```rust
// via_verifier/lib/via_verifier_types/src/transaction.rs
impl UnsignedBridgeTx {
    pub fn get_fee_per_user(&self) -> Amount {
        let withdrawals_count = self.tx.output.len() as u64 - 2;
        if withdrawals_count == 0 {
            return self.fee;
        }
        Amount::from_sat(self.fee.to_sat() / withdrawals_count)
    }

    pub fn is_empty(&self) -> bool {
        self.tx.output.len() as u64 - 2 == 0
    }

    pub fn to_vec(unsigned_bridge_txs_bytes: Vec<Vec<u8>>) -> Vec<UnsignedBridgeTx> {
        let mut unsigned_bridge_txs = vec![];
        for unsigned_bridge_tx in unsigned_bridge_txs_bytes {
            unsigned_bridge_txs.push(UnsignedBridgeTx::from_bytes(&unsigned_bridge_tx))
        }
        unsigned_bridge_txs
    }
}
```

- `get_fee_per_user` handles the edge case where there are no withdrawal outputs (only change and OP_RETURN) by returning the whole fee.

### 6.3 Iterative Filtering of Uneconomical Withdrawals

The algorithm loops: estimate the fee for the current output set, compute the per-user share, drop any output that cannot cover its share, and re-estimate with the smaller set until every remaining output can pay. This is the full `apply_fee_to_outputs` body:

```rust
// via_verifier/lib/via_musig2/src/fee.rs
impl FeeStrategy for WithdrawalFeeStrategy {
    fn apply_fee_to_outputs(
        &self,
        mut outputs: Vec<TransactionOutput>,
        input_count: u32,
        fee_rate: u64,
    ) -> anyhow::Result<TransactionWithFee> {
        loop {
            let fee = self.estimate_fee(input_count, outputs.len() as u32, fee_rate)?;
            if outputs.is_empty() {
                return Ok(TransactionWithFee {
                    outputs_with_fees: vec![],
                    fee,
                    total_value_needed: Amount::ZERO,
                });
            }

            let fee_per_user = Amount::from_sat(fee.to_sat() / outputs.len() as u64);
            let mut total_value_needed = Amount::ZERO;
            let mut valid_outputs_count = 0;

            for output in &outputs {
                if output.output.value >= fee_per_user {
                    valid_outputs_count += 1;
                    total_value_needed += output.output.value - fee_per_user;
                }
            }

            if valid_outputs_count == outputs.len() {
                let mut new_outputs_with_fee = Vec::with_capacity(outputs.len());

                for output in outputs {
                    let output_with_fee = TransactionOutput {
                        output: TxOut {
                            script_pubkey: output.output.script_pubkey,
                            value: output.output.value - fee_per_user,
                        },
                        op_return_data: output.op_return_data,
                    };
                    new_outputs_with_fee.push(output_with_fee);
                }

                return Ok(TransactionWithFee {
                    outputs_with_fees: new_outputs_with_fee,
                    fee,
                    total_value_needed,
                });
            }

            outputs.retain(|output| output.output.value >= fee_per_user);
        }
    }
}
```

- Validations covered by tests in the same file (`via_verifier/lib/via_musig2/src/fee.rs`):
  - Equal split across outputs: `test_fee_applied_equally_to_outputs()`
  - Removal of too-small outputs: `test_small_output_is_removed()`
  - No viable outputs edge case: `test_no_outputs_can_pay_fee()`
  - Zero fee rate passthrough: `test_fee_zero_rate()`

### 6.4 Network Fee Rate Validation

Withdrawal sessions validate the fee rate used to construct the unsigned transaction against the current network fee rate:
- The used fee rate must be within ±1 sat/vB of the network's current estimate
- If the difference exceeds the tolerance, the session verification returns `false` and the transaction is not signed
- The same method then re-parses the withdrawals from the unsigned transaction and rejects the session if any of them is already processed or unknown

The check lives in `_verify_withdrawals` in the withdrawal session:

```rust
// via_verifier/node/via_verifier_coordinator/src/sessions/withdrawal.rs
    async fn _verify_withdrawals(&self, session_op: &SessionOperation) -> anyhow::Result<bool> {
        // Verify the fee used to build the withdrawal transaction.
        let fee_rate = self
            .transaction_builder
            .utxo_manager
            .get_btc_client()
            .get_fee_rate(1)
            .await?;

        let used_fee_rate = session_op.get_unsigned_bridge_tx().fee_rate;

        // Acceptable if difference is within ±1 sat/vbyte
        if (used_fee_rate as i32 - fee_rate as i32).abs() > 1 {
            tracing::error!("Fee mismatch: used={}, network={}", used_fee_rate, fee_rate);
            return Ok(false);
        }

        let l1_withdrawals = self
            .parse_bridge_withdrawal(session_op.get_unsigned_bridge_tx().tx.clone())
            .await?;

        let mut storage = self
            .master_connection_pool
            .connection_tagged("verifier task")
            .await?;

        for w in l1_withdrawals {
            let not_processed = storage
                .via_withdrawal_dal()
                .check_if_withdrawal_exists_unprocessed(&w.clone().into())
                .await?;

            if !not_processed {
                tracing::error!(
                    "Withdrawal already processed or not exists was found in this session, tx hash id: {:?}",
                    w.l2_meta.l2_id
                );
                return Ok(false);
            }
        }

        Ok(true)
    }
```

### 6.5 Behavior with Insufficient Withdrawals

- Before any transaction is built, the withdrawal session only loads withdrawals above a minimum value: `let min_value = 660;` sats in `via_verifier/node/via_verifier_coordinator/src/sessions/withdrawal.rs`, passed to `list_no_processed_withdrawals(min_value, WITHDRAWAL_LIMIT)` (with `const WITHDRAWAL_LIMIT: u32 = 7;`). If nothing qualifies, no session is created
- Withdrawals with `output.value < fee_per_user` are filtered out by the `apply_fee_to_outputs` loop (section 6.3) prior to signing
- If all withdrawals are filtered out, `apply_fee_to_outputs` returns an empty `outputs_with_fees` set, so the resulting transaction would contain only the OP_RETURN and change outputs. `UnsignedBridgeTx::is_empty()` (verbatim in section 6.2) detects exactly this state: it returns true when `tx.output.len() - 2 == 0`
- The UTXO manager is only updated with the bridge transaction after a successful broadcast: `after_broadcast_final_transaction` in the withdrawal session calls `self.transaction_builder.utxo_manager_insert_transaction(session_op.get_unsigned_bridge_tx().tx.clone())` after marking the withdrawals as processed, so transactions that never reach broadcast never pollute UTXO tracking

### 6.6 Integration into the Bridge Flow

The withdrawal session wires `WithdrawalFeeStrategy` into the builder config and calls the entry point:

```rust
// via_verifier/node/via_verifier_coordinator/src/sessions/withdrawal.rs
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
```

The entry point syncs the UTXO context, resolves the fee rate, then chunks the outputs into bridge transactions:

```rust
// via_verifier/lib/via_musig2/src/transaction_builder.rs
    /// Builds transactions with OP_RETURN data from the provided outputs
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

The fee rate the builder uses either comes from the config override or from the Bitcoin client (section 7), floored at 1 sat/vB:

```rust
// via_verifier/lib/via_musig2/src/transaction_builder.rs
    async fn get_fee_rate(&self, config: &TransactionBuilderConfig) -> Result<u64> {
        if let Some(fee_rate) = config.default_fee_rate_opt {
            Ok(fee_rate)
        } else {
            let network_fee = self.utxo_manager.get_btc_client().get_fee_rate(1).await?;
            Ok(std::cmp::max(network_fee, 1))
        }
    }
```

The fee strategy itself is invoked per transaction inside `prepare_build_transaction`, after UTXO selection (so `input_count` reflects the actually selected inputs), and a zero fee is treated as an error:

```rust
// via_verifier/lib/via_musig2/src/transaction_builder.rs
    /// Prepares transaction by selecting UTXOs and calculating fees
    pub async fn prepare_build_transaction(
        &self,
        outputs: Vec<TransactionOutput>,
        available_utxos: &[(OutPoint, TxOut)],
        fee_rate: u64,
        fee_strategy: Arc<dyn FeeStrategy>,
    ) -> Result<(TransactionWithFee, Vec<(OutPoint, TxOut)>)> {
        let total_needed = self.calculate_total_output_value(&outputs);
        let selected_utxos = self.select_utxos(available_utxos, total_needed).await?;

        tracing::debug!("Selected UTXOs {:?}", &selected_utxos);

        let tx_fee =
            fee_strategy.apply_fee_to_outputs(outputs, selected_utxos.len() as u32, fee_rate)?;

        if tx_fee.fee == Amount::ZERO {
            anyhow::bail!("Error to prepare build transaction, fee=0");
        }

        Ok((tx_fee, selected_utxos))
    }
```

- For indexing, validation of origin and storage of bridge withdrawal metadata see:
  - Bridge docs: [bridge.md](bridge.md)

## 7. Bitcoin Fee Rate Estimation (`via_btc_client`)

The Bitcoin client owns the L1 fee rate estimation used by the inscriber, the withdrawal builder, and the gas adjuster. The whole mechanism is configured by four fields:

```rust
// core/lib/config/src/configs/via_btc_client.rs
pub struct ViaBtcClientConfig {
    /// Name of the used Bitcoin network
    pub network: String,
    /// External fee APIs
    pub external_apis: Vec<String>,
    /// Fee strategies
    pub fee_strategies: Vec<String>,
    /// Use the RPC node as main fee rate provider.
    pub use_rpc_for_fee_rate: Option<bool>,
}
```

### 7.1 Estimation Flow

`BitcoinClient::get_fee_rate(conf_target)` (`core/lib/via_btc_client/src/client/mod.rs`) works in two stages (RPC first, external APIs as fallback), then applies a small buffer, a mempool minimum fee floor, and the per-network clamp:

```rust
// core/lib/via_btc_client/src/client/mod.rs
    #[instrument(skip(self), target = "bitcoin_client")]
    async fn get_fee_rate(&self, conf_target: u16) -> BitcoinClientResult<u64> {
        debug!("Estimating fee rate");
        let mut fee_rate_sat_kb: Option<u64> = None;

        if self.config.use_rpc_for_fee_rate() {
            let estimation_result = self
                .rpc
                .estimate_smart_fee(conf_target, Some(EstimateMode::Economical))
                .await;

            fee_rate_sat_kb = match estimation_result {
                Ok(estimation) => {
                    if let Some(fee_rate) = estimation.fee_rate {
                        Some(fee_rate.to_sat())
                    } else {
                        error!(
                            "RPC fee estimate missing value: {}",
                            estimation
                                .errors
                                .map(|e| e.join(", "))
                                .unwrap_or_else(|| "Unknown error".to_string())
                        );
                        None
                    }
                }
                Err(err) => {
                    METRICS.rpc_errors[&RpcMethodLabel {
                        method: "rpc_estimate_smart_fee".into(),
                    }]
                        .inc();
                    error!("Failed to estimate smart fee via RPC: {:?}", err);
                    None
                }
            };
        }

        // Fallback to external APIs if RPC failed or returned no fee
        if fee_rate_sat_kb.is_none() {
            let client = reqwest::Client::new();

            for (api_url, fee_target_key) in self
                .config
                .external_apis
                .iter()
                .zip(self.config.fee_strategies.iter())
            {
                match client.get(api_url).send().await {
                    Ok(resp) => match resp.json::<serde_json::Value>().await {
                        Ok(json) => {
                            if let Some(fee_rate_vb) =
                                json.get(fee_target_key).and_then(|v| v.as_f64())
                            {
                                fee_rate_sat_kb = Some((fee_rate_vb * 1000.0).round() as u64);
                                debug!(
                                    "Fee rate from API {} [{}]: {} sat/vB",
                                    api_url, fee_target_key, fee_rate_vb
                                );
                                break;
                            } else {
                                error!("Missing '{}' in response from {}", fee_target_key, api_url);
                            }
                        }
                        Err(e) => {
                            error!("Failed to parse JSON from {}: {:?}", api_url, e);
                        }
                    },
                    Err(e) => {
                        METRICS.rpc_errors[&RpcMethodLabel {
                            method: "estimate_smart_fee".into(),
                        }]
                            .inc();
                        error!("Failed to fetch from {}: {:?}", api_url, e);
                    }
                }
            }
        }

        // If no fee estimate was obtained
        let mut fee_rate_sat_kb = fee_rate_sat_kb.ok_or_else(|| {
            BitcoinError::FeeEstimationFailed("All fee estimation methods failed".into())
        })?;

        // Add a small buffer to avoid precision loss
        fee_rate_sat_kb += 1000;

        // Get mempool minimum fee
        let mempool_info = self.rpc.get_mempool_info().await?;
        let mempool_min_fee_sat_kb = mempool_info.mempool_min_fee.to_sat();
        fee_rate_sat_kb = std::cmp::max(fee_rate_sat_kb, mempool_min_fee_sat_kb);

        // Convert to sat/vB
        let fee_rate_sat_vb = fee_rate_sat_kb.checked_div(1000).ok_or_else(|| {
            BitcoinError::FeeEstimationFailed("Failed to convert sat/kB to sat/vB".into())
        })?;

        // Apply network-specific caps
        let limits = FeeRateLimits::from_network(self.config.network());
        let capped = std::cmp::min(fee_rate_sat_vb, limits.max_fee_rate());
        let final_rate = std::cmp::max(capped, limits.min_fee_rate());

        debug!("Final fee rate used: {} sat/vB", final_rate);
        Ok(final_rate)
    }
```

Note two details that only appear in the full body: a flat 1000 sat/kB buffer is added before conversion to avoid precision loss, and the estimate is floored by the node's `mempool_min_fee` from `get_mempool_info` before the `FeeRateLimits` clamp.

Each entry in `external_apis` is paired positionally with a JSON key in `fee_strategies`; for example `https://mempool.space/api/v1/fees/recommended` with key `fastestFee`. There is no separate `[fee_strategy]` TOML section or dedicated fee CLI; these four config fields are the entire surface.

### 7.2 Network Fee Rate Limits

Estimates are clamped by hardcoded per-network limits that protect against estimator spikes (notably unreliable testnet estimates):

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
            BitcoinNetwork::Testnet => Self {
                min_fee_rate: 1,
                max_fee_rate: 100,
            },
            _ => Self {
                min_fee_rate: 1,
                max_fee_rate: 10,
            },
        }
    }

    /// Get the minimum fee rate for the network (in sat/vB)
    pub fn min_fee_rate(&self) -> u64 {
        self.min_fee_rate
    }

    /// Get the maximum fee rate for the network (in sat/vB)
    pub fn max_fee_rate(&self) -> u64 {
        self.max_fee_rate
    }
}
```

These are code constants derived from the network, not configuration values.

### 7.3 Mempool Info

The RPC trait exposes the node's mempool state as a plain passthrough of Bitcoin Core's `getmempoolinfo`:

```rust
// core/lib/via_btc_client/src/traits.rs
async fn get_mempool_info(&self) -> BitcoinRpcResult<GetMempoolInfoResult>;
```

There is no congestion-multiplier or percentile-strategy machinery on top of it; fee dynamics come from `estimate_smart_fee`, the external APIs, and the clamp above. A runnable demonstration lives at `core/lib/via_btc_client/examples/fee_rate.rs`.

## 8. Conclusion

Via's fee mechanism spans three layers, each with a distinct job:

1. **L1 fee rate** (`via_btc_client`): RPC estimation with external-API fallback, clamped per network
2. **L2 gas price** (`via_fee_model`): converts the batch's expected inscription cost (vbytes times sat/vB) into an 18-decimal L2 gas price, so users pay real settlement costs
3. **Withdrawal fees** (`via_musig2` fee strategy): distributes the bridge transaction fee equally across withdrawal outputs, filters uneconomical withdrawals, and enforces the ±1 sat/vB fee-rate tolerance at signing time

The inscription-side incentives (20% commit-to-reveal fee shift, +5% reveal incentive, 5% bump per pending inscription) are covered in the Bitcoin sender documentation.