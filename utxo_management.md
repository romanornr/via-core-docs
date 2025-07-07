# UTXO Management Documentation

## Overview

The Via Core system implements sophisticated UTXO (Unspent Transaction Output) management for efficient Bitcoin transaction construction and bridge operations. The UTXO management system handles selection, sorting, consolidation, and optimization of Bitcoin UTXOs to minimize transaction fees and maximize operational efficiency.

## Core Components

### UTXO Selector

The `UtxoSelector` is the primary component responsible for selecting optimal UTXOs for transaction construction.

```rust
pub struct UtxoSelector {
    selection_strategy: UtxoSelectionStrategy,
    dust_threshold: Amount,
    fee_estimator: Arc<FeeEstimator>,
    network: Network,
}

#[derive(Debug, Clone)]
pub enum UtxoSelectionStrategy {
    LargestFirst,
    SmallestFirst,
    BranchAndBound,
    Random,
    Optimal,
}
```

### Enhanced UTXO Selection Algorithm

The system implements multiple selection strategies with the default being the enhanced largest-first algorithm:

```rust
impl UtxoSelector {
    pub async fn select_utxos_for_amount(
        &self,
        target_amount: u64,
        available_utxos: &[Utxo],
        max_inputs: Option<usize>,
    ) -> Result<UtxoSelection, UtxoSelectionError> {
        match self.selection_strategy {
            UtxoSelectionStrategy::LargestFirst => {
                self.select_largest_first(target_amount, available_utxos, max_inputs).await
            }
            UtxoSelectionStrategy::SmallestFirst => {
                self.select_smallest_first(target_amount, available_utxos, max_inputs).await
            }
            UtxoSelectionStrategy::BranchAndBound => {
                self.select_branch_and_bound(target_amount, available_utxos, max_inputs).await
            }
            UtxoSelectionStrategy::Optimal => {
                self.select_optimal(target_amount, available_utxos, max_inputs).await
            }
            UtxoSelectionStrategy::Random => {
                self.select_random(target_amount, available_utxos, max_inputs).await
            }
        }
    }
    
    async fn select_largest_first(
        &self,
        target_amount: u64,
        available_utxos: &[Utxo],
        max_inputs: Option<usize>,
    ) -> Result<UtxoSelection, UtxoSelectionError> {
        // Sort UTXOs by value in descending order for efficient selection
        let mut sorted_utxos = available_utxos
            .iter()
            .filter(|utxo| utxo.value >= self.dust_threshold.to_sat())
            .cloned()
            .collect::<Vec<_>>();
        
        // Enhanced sorting: primary by value (descending), secondary by age (older first)
        sorted_utxos.sort_by(|a, b| {
            match b.value.cmp(&a.value) {
                std::cmp::Ordering::Equal => a.block_height.cmp(&b.block_height),
                other => other,
            }
        });
        
        let mut selected_utxos = Vec::new();
        let mut selected_value = 0u64;
        let max_inputs = max_inputs.unwrap_or(100); // Default max inputs
        
        for utxo in sorted_utxos {
            if selected_utxos.len() >= max_inputs {
                break;
            }
            
            selected_utxos.push(utxo);
            selected_value += utxo.value;
            
            // Check if we have enough value including estimated fees
            let estimated_fee = self.estimate_transaction_fee(selected_utxos.len(), 2).await?;
            if selected_value >= target_amount + estimated_fee {
                break;
            }
        }
        
        let final_fee = self.estimate_transaction_fee(selected_utxos.len(), 2).await?;
        let total_required = target_amount + final_fee;
        
        if selected_value < total_required {
            return Err(UtxoSelectionError::InsufficientFunds {
                required: total_required,
                available: selected_value,
            });
        }
        
        Ok(UtxoSelection {
            utxos: selected_utxos,
            total_value: selected_value,
            target_amount,
            estimated_fee: final_fee,
            change_amount: selected_value.saturating_sub(total_required),
        })
    }
}
```

## UTXO Data Structures

### UTXO Representation

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Utxo {
    pub txid: Txid,
    pub vout: u32,
    pub value: u64,
    pub script_pubkey: ScriptBuf,
    pub address: Option<Address>,
    pub block_height: u32,
    pub confirmations: u32,
    pub spendable: bool,
    pub solvable: bool,
    pub safe: bool,
    pub label: Option<String>,
    pub witness_utxo: Option<TxOut>,
}

impl Utxo {
    pub fn is_dust(&self, dust_threshold: Amount) -> bool {
        self.value < dust_threshold.to_sat()
    }
    
    pub fn is_mature(&self, current_height: u32, maturity_blocks: u32) -> bool {
        current_height.saturating_sub(self.block_height) >= maturity_blocks
    }
    
    pub fn effective_value(&self, fee_rate: u64) -> i64 {
        let input_size = self.input_size();
        let input_fee = (input_size as u64) * fee_rate;
        (self.value as i64) - (input_fee as i64)
    }
    
    fn input_size(&self) -> usize {
        // Estimate input size based on script type
        match self.script_pubkey.len() {
            25 => 148, // P2PKH
            23 => 148, // P2SH
            22 => 68,  // P2WPKH
            34 => 104, // P2WSH
            _ => 148,  // Default to P2PKH size
        }
    }
}
```

### UTXO Selection Result

```rust
#[derive(Debug, Clone)]
pub struct UtxoSelection {
    pub utxos: Vec<Utxo>,
    pub total_value: u64,
    pub target_amount: u64,
    pub estimated_fee: u64,
    pub change_amount: u64,
}

impl UtxoSelection {
    pub fn input_count(&self) -> usize {
        self.utxos.len()
    }
    
    pub fn requires_change(&self) -> bool {
        self.change_amount > 0
    }
    
    pub fn efficiency_score(&self) -> f64 {
        // Calculate efficiency as ratio of target amount to total input value
        if self.total_value == 0 {
            0.0
        } else {
            (self.target_amount as f64) / (self.total_value as f64)
        }
    }
    
    pub fn fee_rate(&self) -> f64 {
        let estimated_tx_size = self.estimated_transaction_size();
        if estimated_tx_size == 0 {
            0.0
        } else {
            (self.estimated_fee as f64) / (estimated_tx_size as f64)
        }
    }
    
    fn estimated_transaction_size(&self) -> usize {
        // Base transaction size + inputs + outputs
        let base_size = 10; // Version (4) + input count (1) + output count (1) + locktime (4)
        let input_size: usize = self.utxos.iter().map(|u| u.input_size()).sum();
        let output_size = 34 * 2; // Assume 2 outputs (recipient + change)
        base_size + input_size + output_size
    }
}
```

## Advanced Selection Strategies

### Branch and Bound Algorithm

Implements the Branch and Bound algorithm for optimal UTXO selection:

```rust
impl UtxoSelector {
    async fn select_branch_and_bound(
        &self,
        target_amount: u64,
        available_utxos: &[Utxo],
        max_inputs: Option<usize>,
    ) -> Result<UtxoSelection, UtxoSelectionError> {
        let fee_rate = self.fee_estimator.get_fee_rate().await?;
        let max_inputs = max_inputs.unwrap_or(100);
        
        // Calculate effective values for all UTXOs
        let mut effective_utxos: Vec<(Utxo, i64)> = available_utxos
            .iter()
            .filter(|utxo| utxo.value >= self.dust_threshold.to_sat())
            .map(|utxo| {
                let effective_value = utxo.effective_value(fee_rate);
                (utxo.clone(), effective_value)
            })
            .filter(|(_, effective_value)| *effective_value > 0)
            .collect();
        
        // Sort by effective value descending
        effective_utxos.sort_by(|a, b| b.1.cmp(&a.1));
        
        let target_effective = target_amount as i64;
        let mut best_selection: Option<Vec<Utxo>> = None;
        let mut best_waste = i64::MAX;
        
        // Branch and bound search
        self.branch_and_bound_search(
            &effective_utxos,
            target_effective,
            0,
            0,
            Vec::new(),
            &mut best_selection,
            &mut best_waste,
            max_inputs,
        );
        
        match best_selection {
            Some(utxos) => {
                let total_value = utxos.iter().map(|u| u.value).sum();
                let estimated_fee = self.estimate_transaction_fee(utxos.len(), 2).await?;
                let change_amount = total_value.saturating_sub(target_amount + estimated_fee);
                
                Ok(UtxoSelection {
                    utxos,
                    total_value,
                    target_amount,
                    estimated_fee,
                    change_amount,
                })
            }
            None => Err(UtxoSelectionError::NoSolutionFound),
        }
    }
    
    fn branch_and_bound_search(
        &self,
        utxos: &[(Utxo, i64)],
        target: i64,
        current_index: usize,
        current_value: i64,
        current_selection: Vec<Utxo>,
        best_selection: &mut Option<Vec<Utxo>>,
        best_waste: &mut i64,
        max_inputs: usize,
    ) {
        // Pruning conditions
        if current_selection.len() > max_inputs {
            return;
        }
        
        if current_value >= target {
            let waste = current_value - target;
            if waste < *best_waste {
                *best_waste = waste;
                *best_selection = Some(current_selection);
            }
            return;
        }
        
        if current_index >= utxos.len() {
            return;
        }
        
        // Upper bound check
        let remaining_value: i64 = utxos[current_index..].iter().map(|(_, v)| v).sum();
        if current_value + remaining_value < target {
            return;
        }
        
        // Try including current UTXO
        let mut new_selection = current_selection.clone();
        new_selection.push(utxos[current_index].0.clone());
        self.branch_and_bound_search(
            utxos,
            target,
            current_index + 1,
            current_value + utxos[current_index].1,
            new_selection,
            best_selection,
            best_waste,
            max_inputs,
        );
        
        // Try excluding current UTXO
        self.branch_and_bound_search(
            utxos,
            target,
            current_index + 1,
            current_value,
            current_selection,
            best_selection,
            best_waste,
            max_inputs,
        );
    }
}
```

## UTXO Consolidation

### Consolidation Strategy

The system implements intelligent UTXO consolidation to reduce fragmentation:

```rust
pub struct UtxoConsolidator {
    dust_threshold: Amount,
    consolidation_threshold: usize,
    max_consolidation_inputs: usize,
    fee_estimator: Arc<FeeEstimator>,
}

impl UtxoConsolidator {
    pub async fn should_consolidate(&self, utxos: &[Utxo]) -> bool {
        let small_utxos = utxos
            .iter()
            .filter(|u| u.value < self.dust_threshold.to_sat() * 10)
            .count();
        
        small_utxos >= self.consolidation_threshold
    }
    
    pub async fn create_consolidation_transaction(
        &self,
        utxos: &[Utxo],
        destination_address: &Address,
    ) -> Result<ConsolidationTransaction, ConsolidationError> {
        // Select UTXOs for consolidation (prioritize smallest first)
        let mut consolidation_utxos = utxos
            .iter()
            .filter(|u| u.value >= self.dust_threshold.to_sat())
            .cloned()
            .collect::<Vec<_>>();
        
        // Sort by value ascending (smallest first for consolidation)
        consolidation_utxos.sort_by(|a, b| a.value.cmp(&b.value));
        
        // Limit number of inputs
        consolidation_utxos.truncate(self.max_consolidation_inputs);
        
        let total_input_value: u64 = consolidation_utxos.iter().map(|u| u.value).sum();
        let estimated_fee = self.estimate_consolidation_fee(consolidation_utxos.len()).await?;
        
        if total_input_value <= estimated_fee {
            return Err(ConsolidationError::InsufficientValue);
        }
        
        let output_value = total_input_value - estimated_fee;
        
        Ok(ConsolidationTransaction {
            inputs: consolidation_utxos,
            output_value,
            destination_address: destination_address.clone(),
            estimated_fee,
        })
    }
    
    async fn estimate_consolidation_fee(&self, input_count: usize) -> Result<u64, FeeEstimationError> {
        let fee_rate = self.fee_estimator.get_fee_rate().await?;
        
        // Estimate transaction size: inputs (148 bytes each) + 1 output (34 bytes) + overhead (10 bytes)
        let estimated_size = (input_count * 148) + 34 + 10;
        Ok((estimated_size as u64) * fee_rate)
    }
}
```

## Fee Estimation

### Dynamic Fee Estimation

The system implements dynamic fee estimation based on network conditions:

```rust
pub struct FeeEstimator {
    btc_client: Arc<BitcoinClient>,
    fee_cache: Arc<RwLock<FeeCache>>,
    network: Network,
}

impl FeeEstimator {
    pub async fn get_fee_rate(&self) -> Result<u64, FeeEstimationError> {
        // Check cache first
        if let Some(cached_rate) = self.get_cached_fee_rate().await {
            return Ok(cached_rate);
        }
        
        // Fetch from Bitcoin node
        let fee_rate = match self.network {
            Network::Bitcoin => self.estimate_mainnet_fee().await?,
            Network::Testnet => self.estimate_testnet_fee().await?,
            Network::Regtest => self.get_regtest_fee().await?,
            _ => return Err(FeeEstimationError::UnsupportedNetwork),
        };
        
        // Cache the result
        self.cache_fee_rate(fee_rate).await;
        
        Ok(fee_rate)
    }
    
    async fn estimate_mainnet_fee(&self) -> Result<u64, FeeEstimationError> {
        // Use estimatesmartfee for mainnet
        let estimate = self.btc_client.estimate_smart_fee(6).await?; // 6 blocks target
        let fee_rate = (estimate.fee_rate * 100_000_000.0) as u64; // Convert BTC/kB to sat/byte
        
        // Apply network-specific limits
        let min_fee = 1; // 1 sat/byte minimum
        let max_fee = 1000; // 1000 sat/byte maximum
        
        Ok(fee_rate.clamp(min_fee, max_fee))
    }
    
    async fn estimate_testnet_fee(&self) -> Result<u64, FeeEstimationError> {
        // More conservative for testnet
        let estimate = self.btc_client.estimate_smart_fee(3).await?;
        let fee_rate = (estimate.fee_rate * 100_000_000.0) as u64;
        
        let min_fee = 1;
        let max_fee = 100; // Lower max for testnet
        
        Ok(fee_rate.clamp(min_fee, max_fee))
    }
    
    async fn get_regtest_fee(&self) -> Result<u64, FeeEstimationError> {
        // Fixed low fee for regtest
        Ok(1) // 1 sat/byte
    }
    
    pub async fn estimate_transaction_fee(
        &self,
        input_count: usize,
        output_count: usize,
    ) -> Result<u64, FeeEstimationError> {
        let fee_rate = self.get_fee_rate().await?;
        
        // Calculate transaction size
        let base_size = 10; // Version + input count + output count + locktime
        let input_size = input_count * 148; // Average input size
        let output_size = output_count * 34; // Average output size
        let total_size = base_size + input_size + output_size;
        
        Ok((total_size as u64) * fee_rate)
    }
}
```

## UTXO Monitoring and Maintenance

### UTXO Set Management

```rust
pub struct UtxoManager {
    btc_client: Arc<BitcoinClient>,
    database: Arc<Database>,
    consolidator: UtxoConsolidator,
    selector: UtxoSelector,
}

impl UtxoManager {
    pub async fn refresh_utxo_set(&mut self) -> Result<(), UtxoManagerError> {
        // Fetch current UTXOs from Bitcoin node
        let current_utxos = self.btc_client.list_unspent(None, None, None).await?;
        
        // Update database with current UTXO set
        self.database.update_utxo_set(&current_utxos).await?;
        
        // Check if consolidation is needed
        if self.consolidator.should_consolidate(&current_utxos).await {
            tracing::info!("UTXO consolidation recommended: {} small UTXOs detected", 
                          current_utxos.len());
        }
        
        Ok(())
    }
    
    pub async fn get_optimal_utxos_for_amount(
        &self,
        amount: u64,
        max_inputs: Option<usize>,
    ) -> Result<UtxoSelection, UtxoManagerError> {
        let available_utxos = self.database.get_spendable_utxos().await?;
        
        self.selector
            .select_utxos_for_amount(amount, &available_utxos, max_inputs)
            .await
            .map_err(UtxoManagerError::SelectionError)
    }
    
    pub async fn mark_utxos_as_spent(
        &self,
        utxos: &[Utxo],
        spending_txid: Txid,
    ) -> Result<(), UtxoManagerError> {
        self.database.mark_utxos_spent(utxos, spending_txid).await?;
        Ok(())
    }
    
    pub async fn get_utxo_statistics(&self) -> Result<UtxoStatistics, UtxoManagerError> {
        let utxos = self.database.get_all_utxos().await?;
        
        let total_count = utxos.len();
        let total_value: u64 = utxos.iter().map(|u| u.value).sum();
        let dust_count = utxos.iter().filter(|u| u.is_dust(Amount::from_sat(546))).count();
        let average_value = if total_count > 0 { total_value / (total_count as u64) } else { 0 };
        
        // Calculate value distribution
        let mut value_buckets = [0usize; 10];
        for utxo in &utxos {
            let bucket_index = match utxo.value {
                0..=1000 => 0,
                1001..=10000 => 1,
                10001..=100000 => 2,
                100001..=1000000 => 3,
                1000001..=10000000 => 4,
                10000001..=100000000 => 5,
                100000001..=1000000000 => 6,
                1000000001..=10000000000 => 7,
                10000000001..=100000000000 => 8,
                _ => 9,
            };
            value_buckets[bucket_index] += 1;
        }
        
        Ok(UtxoStatistics {
            total_count,
            total_value,
            dust_count,
            average_value,
            value_distribution: value_buckets,
        })
    }
}
```

## Configuration and Tuning

### UTXO Management Configuration

```toml
# etc/env/base/via_utxo_management.toml

[utxo_selection]
strategy = "largest_first"  # largest_first, smallest_first, branch_and_bound, optimal
dust_threshold = 546        # satoshis
max_inputs_per_tx = 100
selection_timeout = 30      # seconds

[consolidation]
enabled = true
threshold = 50              # consolidate when more than 50 small UTXOs
max_inputs = 200           # maximum inputs in consolidation tx
small_utxo_multiplier = 10 # UTXOs smaller than dust_threshold * multiplier

[fee_estimation]
cache_duration = 300       # seconds
mainnet_max_fee_rate = 1000 # sat/byte
testnet_max_fee_rate = 100  # sat/byte
regtest_fee_rate = 1        # sat/byte
confirmation_target = 6     # blocks

[monitoring]
refresh_interval = 60      # seconds
statistics_interval = 300  # seconds
alert_dust_threshold = 100 # alert when more than 100 dust UTXOs
```

### Environment Variables

```bash
# UTXO Selection Configuration
export VIA_UTXO_SELECTION_STRATEGY="largest_first"
export VIA_UTXO_DUST_THRESHOLD=546
export VIA_UTXO_MAX_INPUTS=100

# Consolidation Configuration
export VIA_UTXO_CONSOLIDATION_ENABLED=true
export VIA_UTXO_CONSOLIDATION_THRESHOLD=50
export VIA_UTXO_CONSOLIDATION_MAX_INPUTS=200

# Fee Estimation Configuration
export VIA_FEE_CACHE_DURATION=300
export VIA_MAINNET_MAX_FEE_RATE=1000
export VIA_TESTNET_MAX_FEE_RATE=100
export VIA_REGTEST_FEE_RATE=1
```

## Performance Optimization

### Optimization Strategies

1. **UTXO Caching**: Cache UTXO sets to reduce Bitcoin node queries
2. **Parallel Processing**: Process UTXO operations in parallel where possible
3. **Database Indexing**: Optimize database queries with proper indexing
4. **Fee Rate Caching**: Cache fee rate estimates to reduce API calls
5. **Batch Operations**: Batch UTXO updates and database operations

### Monitoring and Metrics

Key metrics to monitor for UTXO management:

- **UTXO Set Size**: Total number of UTXOs
- **UTXO Value Distribution**: Distribution of UTXO values
- **Dust UTXO Count**: Number of dust UTXOs
- **Selection Efficiency**: Average efficiency of UTXO selections
- **Consolidation Frequency**: How often consolidation is performed
- **Fee Rate Accuracy**: Accuracy of fee rate estimates
- **Selection Time**: Time taken for UTXO selection operations

This comprehensive UTXO management system ensures efficient Bitcoin transaction construction while minimizing fees and maintaining optimal UTXO set health.