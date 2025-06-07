# Via L2 Examples and Utilities Documentation

## Overview

This document provides comprehensive documentation for the examples and utilities available in the Via L2 Bitcoin ZK-Rollup system. These examples demonstrate key functionality and provide practical implementation guidance for developers working with the Via L2 system.

## Table of Contents

1. [ZK Proof Verification Examples](#zk-proof-verification-examples)
2. [Fee Rate Fetching Examples](#fee-rate-fetching-examples)
3. [MuSig2 Utilities and Examples](#musig2-utilities-and-examples)
4. [Bitcoin Client Examples](#bitcoin-client-examples)
5. [Development and Testing Utilities](#development-and-testing-utilities)
6. [Configuration Examples](#configuration-examples)
7. [Best Practices](#best-practices)

## ZK Proof Verification Examples

### ZK Proof Verification from Data Availability

The [`via_verifier/lib/via_verification/examples/zk.rs`](via_verifier/lib/via_verification/examples/zk.rs) example demonstrates how to verify ZK proofs from the Data Availability layer.

#### Basic ZK Proof Verification

```rust
use via_verification::{ZkProofVerifier, DataAvailabilityClient, ProofVerificationError};
use via_types::{L1BatchNumber, ProofData};

#[tokio::main]
async fn main() -> Result<(), ProofVerificationError> {
    // Initialize the ZK proof verifier
    let verifier = ZkProofVerifier::new()?;
    
    // Initialize Data Availability client
    let da_client = DataAvailabilityClient::new("https://da-endpoint.example.com").await?;
    
    // Fetch proof data for a specific batch
    let batch_number = L1BatchNumber(12345);
    let proof_data = da_client.fetch_proof_data(batch_number).await?;
    
    // Verify the ZK proof
    let verification_result = verifier.verify_proof(&proof_data).await?;
    
    if verification_result.is_valid {
        println!("✅ ZK proof verification successful for batch {}", batch_number);
        println!("Verification time: {:?}", verification_result.verification_time);
    } else {
        println!("❌ ZK proof verification failed: {}", verification_result.error_message);
    }
    
    Ok(())
}
```

#### Batch ZK Proof Verification

```rust
use via_verification::{ZkProofVerifier, BatchVerificationConfig};
use std::collections::HashMap;

async fn verify_multiple_proofs(
    verifier: &ZkProofVerifier,
    batch_numbers: Vec<L1BatchNumber>,
) -> Result<HashMap<L1BatchNumber, bool>, ProofVerificationError> {
    let mut results = HashMap::new();
    
    // Configure batch verification
    let config = BatchVerificationConfig {
        max_concurrent_verifications: 4,
        timeout_per_proof: Duration::from_secs(30),
        retry_failed_proofs: true,
    };
    
    // Verify proofs in parallel
    let verification_tasks: Vec<_> = batch_numbers
        .into_iter()
        .map(|batch_number| {
            let verifier = verifier.clone();
            tokio::spawn(async move {
                let proof_data = fetch_proof_data_for_batch(batch_number).await?;
                let result = verifier.verify_proof(&proof_data).await?;
                Ok((batch_number, result.is_valid))
            })
        })
        .collect();
    
    // Collect results
    for task in verification_tasks {
        let (batch_number, is_valid) = task.await??;
        results.insert(batch_number, is_valid);
    }
    
    Ok(results)
}
```

#### Integration with Verification Workflows

```rust
use via_verification::{VerificationWorkflow, WorkflowConfig};

pub struct AutomatedVerificationPipeline {
    verifier: ZkProofVerifier,
    da_client: DataAvailabilityClient,
    config: WorkflowConfig,
}

impl AutomatedVerificationPipeline {
    pub async fn run_verification_pipeline(&self) -> Result<(), VerificationError> {
        // Get latest finalized batches
        let finalized_batches = self.da_client.get_finalized_batches().await?;
        
        for batch in finalized_batches {
            // Skip if already verified
            if self.is_batch_verified(batch.number).await? {
                continue;
            }
            
            // Fetch and verify proof
            let proof_data = self.da_client.fetch_proof_data(batch.number).await?;
            let verification_result = self.verifier.verify_proof(&proof_data).await?;
            
            // Store verification result
            self.store_verification_result(batch.number, verification_result).await?;
            
            // Log verification status
            tracing::info!(
                "Verified batch {}: {}",
                batch.number,
                if verification_result.is_valid { "VALID" } else { "INVALID" }
            );
        }
        
        Ok(())
    }
}
```

## Fee Rate Fetching Examples

### Basic Fee Rate Fetching

The [`core/lib/via_btc_client/examples/fee_rate.rs`](core/lib/via_btc_client/examples/fee_rate.rs) example demonstrates how to fetch Bitcoin fee rates using various strategies.

```rust
use via_btc_client::{BitcoinClient, FeeRateStrategy, FeeRateError};
use bitcoin::Network;

#[tokio::main]
async fn main() -> Result<(), FeeRateError> {
    // Initialize Bitcoin client
    let client = BitcoinClient::new("http://localhost:18332", Network::Testnet)?;
    
    // Fetch fee rates using different strategies
    let conservative_fee = client.estimate_fee_rate(FeeRateStrategy::Conservative).await?;
    let standard_fee = client.estimate_fee_rate(FeeRateStrategy::Standard).await?;
    let aggressive_fee = client.estimate_fee_rate(FeeRateStrategy::Aggressive).await?;
    
    println!("Fee Rate Estimates:");
    println!("Conservative: {} sat/vB", conservative_fee);
    println!("Standard: {} sat/vB", standard_fee);
    println!("Aggressive: {} sat/vB", aggressive_fee);
    
    Ok(())
}
```

### Advanced Fee Rate Management

```rust
use via_btc_client::{FeeRateManager, FeeRateLimits, ExternalFeeApi};

pub struct AdvancedFeeManager {
    btc_client: BitcoinClient,
    fee_limits: FeeRateLimits,
    external_apis: Vec<Box<dyn ExternalFeeApi>>,
}

impl AdvancedFeeManager {
    pub async fn get_optimal_fee_rate(&self) -> Result<u64, FeeRateError> {
        // Try RPC-based estimation first
        if let Ok(rpc_fee) = self.btc_client.estimate_fee_rate(FeeRateStrategy::Standard).await {
            if self.fee_limits.is_within_limits(rpc_fee) {
                return Ok(rpc_fee);
            }
        }
        
        // Fallback to external APIs
        for api in &self.external_apis {
            if let Ok(external_fee) = api.get_fee_rate().await {
                if self.fee_limits.is_within_limits(external_fee) {
                    return Ok(external_fee);
                }
            }
        }
        
        // Use conservative fallback
        Ok(self.fee_limits.get_conservative_rate())
    }
    
    pub async fn get_mempool_based_fee(&self) -> Result<u64, FeeRateError> {
        let mempool_info = self.btc_client.get_mempool_info().await?;
        
        // Calculate fee based on mempool congestion
        let congestion_factor = mempool_info.size as f64 / mempool_info.max_mempool_size as f64;
        let base_fee = self.btc_client.estimate_fee_rate(FeeRateStrategy::Standard).await?;
        
        let adjusted_fee = if congestion_factor > 0.8 {
            // High congestion - increase fee
            (base_fee as f64 * 1.5) as u64
        } else if congestion_factor < 0.2 {
            // Low congestion - decrease fee
            (base_fee as f64 * 0.8) as u64
        } else {
            base_fee
        };
        
        Ok(self.fee_limits.clamp_fee_rate(adjusted_fee))
    }
}
```

### Network-Specific Fee Configuration

```rust
use bitcoin::Network;

pub fn get_network_fee_limits(network: Network) -> FeeRateLimits {
    match network {
        Network::Bitcoin => FeeRateLimits {
            min_fee_rate: 1,      // 1 sat/vB minimum
            max_fee_rate: 100,    // 100 sat/vB maximum
            default_fee_rate: 10, // 10 sat/vB default
        },
        Network::Testnet => FeeRateLimits {
            min_fee_rate: 1,      // 1 sat/vB minimum
            max_fee_rate: 1000,   // 1000 sat/vB maximum (higher for testing)
            default_fee_rate: 5,  // 5 sat/vB default
        },
        Network::Regtest => FeeRateLimits {
            min_fee_rate: 1,      // 1 sat/vB minimum
            max_fee_rate: 10000,  // Very high for local testing
            default_fee_rate: 1,  // 1 sat/vB default
        },
        _ => FeeRateLimits::default(),
    }
}
```

## MuSig2 Utilities and Examples

### Partial Signature Verification

The [`via_verifier/lib/via_musig2/examples/verify_partial_sig.rs`](via_verifier/lib/via_musig2/examples/verify_partial_sig.rs) example demonstrates MuSig2 partial signature verification.

```rust
use via_musig2::{verify_partial_signature, PartialSignature, PublicKey, Message};
use secp256k1::{Secp256k1, SecretKey};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let secp = Secp256k1::new();
    
    // Example keys and message
    let secret_key = SecretKey::from_slice(&[1u8; 32])?;
    let public_key = PublicKey::from_secret_key(&secp, &secret_key);
    let message = Message::from_slice(&[2u8; 32])?;
    
    // Create a partial signature (this would normally come from a verifier)
    let partial_sig = create_partial_signature(&secret_key, &message)?;
    
    // Verify the partial signature
    let is_valid = verify_partial_signature(&public_key, &message, &partial_sig)?;
    
    if is_valid {
        println!("✅ Partial signature is valid");
    } else {
        println!("❌ Partial signature is invalid");
    }
    
    Ok(())
}

fn create_partial_signature(
    secret_key: &SecretKey,
    message: &Message,
) -> Result<PartialSignature, Box<dyn std::error::Error>> {
    // Implementation would create a proper MuSig2 partial signature
    // This is simplified for example purposes
    todo!("Implement partial signature creation")
}
```

### Advanced MuSig2 Session Management

```rust
use via_musig2::{MuSig2Session, SessionConfig, VerifierRole};

pub struct MuSig2SessionManager {
    sessions: HashMap<SessionId, MuSig2Session>,
    config: SessionConfig,
}

impl MuSig2SessionManager {
    pub async fn create_signing_session(
        &mut self,
        verifiers: Vec<PublicKey>,
        message: Message,
        threshold: usize,
    ) -> Result<SessionId, MuSig2Error> {
        let session_id = SessionId::new();
        
        let session = MuSig2Session::new(SessionConfig {
            verifiers,
            threshold,
            timeout: Duration::from_secs(300), // 5 minutes
            message,
        })?;
        
        self.sessions.insert(session_id, session);
        
        tracing::info!(
            "Created MuSig2 session {} with {} verifiers, threshold {}",
            session_id,
            verifiers.len(),
            threshold
        );
        
        Ok(session_id)
    }
    
    pub async fn submit_partial_signature(
        &mut self,
        session_id: SessionId,
        verifier_pubkey: PublicKey,
        partial_sig: PartialSignature,
    ) -> Result<SessionStatus, MuSig2Error> {
        let session = self.sessions.get_mut(&session_id)
            .ok_or(MuSig2Error::SessionNotFound)?;
        
        // Verify the partial signature before accepting
        if !verify_partial_signature(&verifier_pubkey, &session.message, &partial_sig)? {
            return Err(MuSig2Error::InvalidPartialSignature);
        }
        
        session.add_partial_signature(verifier_pubkey, partial_sig)?;
        
        // Check if we have enough signatures
        if session.has_threshold_signatures() {
            let final_signature = session.aggregate_signatures()?;
            tracing::info!("Session {} completed with final signature", session_id);
            return Ok(SessionStatus::Completed(final_signature));
        }
        
        Ok(SessionStatus::WaitingForSignatures {
            received: session.signature_count(),
            required: session.threshold,
        })
    }
}
```

## Bitcoin Client Examples

### Multi-Client Configuration

```rust
use via_btc_client::{BitcoinClient, ClientRole, ClientConfig};

pub struct MultiClientManager {
    sender_client: BitcoinClient,
    verifier_client: BitcoinClient,
    bridge_client: BitcoinClient,
}

impl MultiClientManager {
    pub fn new(config: &BitcoinClientConfig) -> Result<Self, ClientError> {
        Ok(Self {
            sender_client: BitcoinClient::new_with_role(
                &config.sender_rpc_url,
                ClientRole::Sender,
            )?,
            verifier_client: BitcoinClient::new_with_role(
                &config.verifier_rpc_url,
                ClientRole::Verifier,
            )?,
            bridge_client: BitcoinClient::new_with_role(
                &config.bridge_rpc_url,
                ClientRole::Bridge,
            )?,
        })
    }
    
    pub async fn broadcast_transaction(&self, tx: &Transaction) -> Result<Txid, ClientError> {
        // Use sender client for broadcasting
        self.sender_client.send_raw_transaction(tx).await
    }
    
    pub async fn verify_transaction(&self, txid: &Txid) -> Result<bool, ClientError> {
        // Use verifier client for verification
        let tx_info = self.verifier_client.get_transaction(txid).await?;
        Ok(tx_info.confirmations >= 6)
    }
    
    pub async fn monitor_address(&self, address: &Address) -> Result<Vec<Transaction>, ClientError> {
        // Use bridge client for monitoring
        self.bridge_client.get_address_transactions(address).await
    }
}
```

### Wallet-Specific Operations

```rust
use via_btc_client::{WalletClient, WalletConfig};

pub struct WalletManager {
    wallet_client: WalletClient,
    wallet_name: String,
}

impl WalletManager {
    pub fn new(base_rpc_url: &str, wallet_name: String) -> Result<Self, ClientError> {
        let wallet_rpc_url = format!("{}/wallet/{}", base_rpc_url, wallet_name);
        let wallet_client = WalletClient::new(&wallet_rpc_url)?;
        
        Ok(Self {
            wallet_client,
            wallet_name,
        })
    }
    
    pub async fn get_wallet_utxos(&self) -> Result<Vec<Utxo>, ClientError> {
        self.wallet_client.list_unspent(None, None, None, None, None).await
    }
    
    pub async fn create_transaction(
        &self,
        outputs: Vec<TxOut>,
        fee_rate: u64,
    ) -> Result<Transaction, ClientError> {
        let utxos = self.get_wallet_utxos().await?;
        let selected_utxos = self.select_utxos(&utxos, &outputs, fee_rate)?;
        
        self.wallet_client.create_raw_transaction(
            &selected_utxos,
            &outputs,
            Some(fee_rate),
        ).await
    }
}
```

## Development and Testing Utilities

### Testnet Bootstrap Utilities

```rust
use via_btc_client::{TestnetBootstrap, BootstrapConfig};

pub struct TestnetManager {
    bootstrap: TestnetBootstrap,
    config: BootstrapConfig,
}

impl TestnetManager {
    pub async fn initialize_testnet(&self) -> Result<(), BootstrapError> {
        // Load bootstrap transaction IDs
        let bootstrap_txids = self.load_bootstrap_txids().await?;
        
        // Verify bootstrap transactions are confirmed
        for txid in &bootstrap_txids {
            let confirmations = self.bootstrap.get_transaction_confirmations(txid).await?;
            if confirmations < self.config.min_confirmations {
                return Err(BootstrapError::InsufficientConfirmations(*txid));
            }
        }
        
        // Initialize bridge address
        let bridge_address = self.bootstrap.derive_bridge_address(&bootstrap_txids).await?;
        
        tracing::info!("Testnet initialized with bridge address: {}", bridge_address);
        Ok(())
    }
    
    async fn load_bootstrap_txids(&self) -> Result<Vec<Txid>, BootstrapError> {
        let config_path = "etc/env/via/genesis/testnet/bootstrap_transactions.json";
        let config_data = tokio::fs::read_to_string(config_path).await?;
        let bootstrap_config: BootstrapTransactionConfig = serde_json::from_str(&config_data)?;
        
        Ok(bootstrap_config.testnet_bootstrap_transactions.into_values().collect())
    }
}
```

### Development Server Components

```rust
use via_server::{ServerComponent, ComponentConfig};

pub struct DevelopmentServer {
    components: Vec<Box<dyn ServerComponent>>,
    config: ComponentConfig,
}

impl DevelopmentServer {
    pub fn new_with_components(component_names: Vec<String>) -> Result<Self, ServerError> {
        let mut components = Vec::new();
        
        for component_name in component_names {
            let component = match component_name.as_str() {
                "btc_sender" => Box::new(BtcSenderComponent::new()?) as Box<dyn ServerComponent>,
                "verifier" => Box::new(VerifierComponent::new()?) as Box<dyn ServerComponent>,
                "bridge" => Box::new(BridgeComponent::new()?) as Box<dyn ServerComponent>,
                "indexer" => Box::new(IndexerComponent::new()?) as Box<dyn ServerComponent>,
                _ => return Err(ServerError::UnknownComponent(component_name)),
            };
            components.push(component);
        }
        
        Ok(Self {
            components,
            config: ComponentConfig::development(),
        })
    }
    
    pub async fn start_selective_components(&mut self) -> Result<(), ServerError> {
        for component in &mut self.components {
            component.start(&self.config).await?;
            tracing::info!("Started component: {}", component.name());
        }
        
        Ok(())
    }
}
```

## Configuration Examples

### Complete Development Configuration

```toml
# Development configuration example
[bitcoin_client]
network = "regtest"
rpc_url = "http://localhost:18443"
rpc_user = "bitcoin"
rpc_password = "password"

# Wallet configuration
[wallet]
wallet_name = "via_dev_wallet"
wallet_rpc_url = "http://localhost:18443/wallet/via_dev_wallet"

# Fee configuration
[fees]
strategy = "conservative"
min_fee_rate = 1
max_fee_rate = 1000
default_fee_rate = 10

# MuSig2 configuration
[musig2]
threshold = 2
total_verifiers = 3
session_timeout_seconds = 300

# Development server components
[server]
components = ["btc_sender", "verifier", "bridge"]
log_level = "debug"
metrics_port = 9090
```

### Production Configuration

```toml
# Production configuration example
[bitcoin_client]
network = "mainnet"
rpc_url = "https://bitcoin-rpc.example.com"
rpc_user = "production_user"
rpc_password = "secure_password"

# Multiple client configuration
[bitcoin_clients]
sender_rpc_url = "https://bitcoin-sender.example.com"
verifier_rpc_url = "https://bitcoin-verifier.example.com"
bridge_rpc_url = "https://bitcoin-bridge.example.com"
max_connections = 50

# Wallet configuration
[wallet]
wallet_name = "via_production_wallet"
wallet_rpc_url = "https://bitcoin-sender.example.com/wallet/via_production_wallet"

# Fee configuration
[fees]
strategy = "standard"
min_fee_rate = 1
max_fee_rate = 50
default_fee_rate = 10
external_apis = ["mempool.space", "blockcypher"]

# MuSig2 configuration
[musig2]
threshold = 7
total_verifiers = 10
session_timeout_seconds = 600

# Production server components
[server]
components = ["all"]
log_level = "info"
metrics_port = 9090
health_check_port = 8080
```

## Best Practices

### Error Handling

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum ExampleError {
    #[error("Bitcoin client error: {0}")]
    BitcoinClient(#[from] via_btc_client::Error),
    
    #[error("MuSig2 error: {0}")]
    MuSig2(#[from] via_musig2::Error),
    
    #[error("Verification error: {0}")]
    Verification(#[from] via_verification::Error),
    
    #[error("Configuration error: {0}")]
    Config(String),
}

pub type ExampleResult<T> = Result<T, ExampleError>;
```

### Logging and Monitoring

```rust
use tracing::{info, warn, error, instrument};

#[instrument(skip(self))]
pub async fn process_with_monitoring(&self) -> ExampleResult<()> {
    info!("Starting processing");
    
    match self.do_processing().await {
        Ok(result) => {
            info!("Processing completed successfully");
            Ok(result)
        }
        Err(e) => {
            error!("Processing failed: {}", e);
            Err(e)
        }
    }
}
```

### Testing Utilities

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use via_test_utils::{TestBitcoinClient, TestMuSig2Session};
    
    #[tokio::test]
    async fn test_fee_rate_fetching() {
        let client = TestBitcoinClient::new_regtest();
        let fee_rate = client.estimate_fee_rate(FeeRateStrategy::Standard).await.unwrap();
        assert!(fee_rate > 0);
    }
    
    #[tokio::test]
    async fn test_musig2_verification() {
        let session = TestMuSig2Session::new_with_threshold(2, 3);
        let result = session.verify_partial_signature(&test_signature()).await;
        assert!(result.is_ok());
    }
}
```

## Conclusion

The Via L2 system provides comprehensive examples and utilities that demonstrate:

1. **ZK Proof Verification**: Complete workflows for verifying ZK proofs from Data Availability
2. **Fee Rate Management**: Advanced fee estimation with multiple strategies and fallback mechanisms
3. **MuSig2 Operations**: Partial signature verification and session management
4. **Bitcoin Client Integration**: Multi-client support with role-based separation
5. **Development Tools**: Testnet bootstrapping and selective component execution

These examples serve as practical implementation guides for developers integrating with the Via L2 system, providing both basic usage patterns and advanced operational scenarios. The utilities ensure robust, secure, and efficient operation of the Via L2 Bitcoin ZK-Rollup system.