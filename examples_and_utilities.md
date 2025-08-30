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

## Via test utils crate

A helper crate provides reusable building blocks for end-to-end and integration tests:
- Location: core/lib/via_test_utils
- Utilities:
  - Test Bitcoin client and sequencer inscriber helpers
  - Bootstrap state mock with sequencer vote set
  - Chained inscription generator that emits a sequence of L1BatchDAReference followed by ProofDAReference, returning both parsed messages and the last batch hash
  - Test indexer constructor using the test client and parser

Typical usage in watcher tests:
- Create N chained batches starting from k, optionally seeding prev_l1_batch_hash
- Feed all messages in order to the watcher’s VerifierMessageProcessor
- Assert DB side-effects: existence for batches 1..N, canonical head and first not-finalized/verified queries

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

### Tip: Handling empty withdrawal transactions (MuSig2)
- When all withdrawals fall below their fair fee share after filtering, the builder produces an unsigned transaction that is effectively empty. In this case, [rust TransactionBuilder::build_transaction_with_op_return()](via_verifier/lib/via_musig2/src/transaction_builder.rs:58) will not insert it into the UTXO manager, and [rust UnsignedBridgeTx::is_empty()](via_verifier/lib/via_verifier_types/src/transaction.rs:25) can be used by downstream code to detect and skip broadcast or session progression.
- Recommended handling:
  - Check empties early and return a no-op result to the coordinator
  - Log a structured reason (e.g., “all withdrawals below fee_per_user”) for auditability
  - Consider notifying upstream components to adjust minimum withdrawal thresholds

### CLI Debug: Batch deposits and withdrawals (commits 0062, 0067)

- Debug CLI commands for high-volume local testing were added:
  - [infrastructure/via/src/debug.ts](https://github.com/vianetwork/via-core/blob/main/infrastructure/via/src/debug.ts)
  - Shared defaults/constants: [infrastructure/via/src/constants.ts](https://github.com/vianetwork/via-core/blob/main/infrastructure/via/src/constants.ts)
  - Wallet helpers (BIP39/BIP32 P2WPKH): [infrastructure/via/src/helpers.ts](https://github.com/vianetwork/via-core/blob/main/infrastructure/via/src/helpers.ts)
  - Token ops (exported helpers): [infrastructure/via/src/token.ts](https://github.com/vianetwork/via-core/blob/main/infrastructure/via/src/token.ts)

Commands:
- Batch deposits to L2:
  ```bash
  via debug deposit-many \
    --amount 0.10 \
    --receiver-l2-address 0xYourL2Address \
    --count 100 \
    [--sender-private-key <WIF>] \
    [--network regtest|testnet|bitcoin] \
    [--l1-rpc-url http://0.0.0.0:18443] \
    [--l2-rpc-url http://0.0.0.0:3050] \
    [--rpc-username rpcuser] \
    [--rpc-password rpcpassword]
  ```
  - Splits total amount evenly across count deposits using deposit() from [infrastructure/via/src/token.ts](https://github.com/vianetwork/via-core/blob/main/infrastructure/via/src/token.ts).

- Batch withdrawals to random L1 addresses:
  ```bash
  via debug withdraw-many \
    --amount 1.0 \
    --count 10 \
    [--user-private-key 0x...] \
    [--rpc-url http://0.0.0.0:3050] \
    [--network regtest|testnet|bitcoin]
  ```
  - Uses generateBitcoinWallet() to derive fresh P2WPKH addresses; logs the receiver per withdrawal.
  - Reuses a single L2 wallet and sets sequential nonces to avoid clashes.

Notes:
- getWallet() and withdraw(..., nonce?) are exported for programmatic use; withdraw-many auto-increments nonce starting from the current account nonce.
- Defaults for keys, RPC URLs and network are taken from [infrastructure/via/src/constants.ts](https://github.com/vianetwork/via-core/blob/main/infrastructure/via/src/constants.ts).

### Transactions generator: direct host mode

- You can now generate random regtest transactions either inside the compose container (default) or directly on the host, and override RPC parameters.
- Command:
  ```bash
  via transactions \
    --rpc-connect bitcoind \
    --rpc-username rpcuser \
    --rpc-password rpcpassword \
    --rpc-wallet Alice \
    --address mqdofsXHpePPGBFXuwwypAqCcXi48Xhb2f \
    --skip-container   # run bitcoin-cli locally on the host
  ```
- Implementation: [`typescript generateRandomTransactions(options)`](infrastructure/via/src/transactions.ts) builds bitcoin-cli calls with a docker prefix unless --skip-container is set. Bootstrap env handling tolerates empty-string values. See [`typescript updateBootstrapTxidsEnv()`](infrastructure/via/src/bootstrap.ts).

### Multisig subcommands for upgrade and system-wallet executions

- The “multisig” CLI can construct, sign, finalize, and broadcast OP_RETURN governance execution transactions for:
  - Protocol upgrade execution ("VIA_PROTOCOL:UPGRADE")
  - Sequencer update ("VIA_PROTOCOL:SEQ")
  - Governance update ("VIA_PROTOCOL:GOV")
  - Bridge update execution ("VIA_PROTOCOL:BRI" referencing an UpdateBridgeProposal txid)

Core helpers:
- Compute multisig (P2WSH):
  ```bash
  via multisig compute-multisig \
    --pubkeys <hexPubkey1,hexPubkey2,hexPubkey3> \
    --minimumSigners 2 \
    --out ./upgrade_tx_exec.json
  ```

- Create unsigned upgrade PSBT (OP_RETURN + change to P2WSH):
  ```bash
  via multisig create-upgrade-tx \
    --inputTxId <tx_id> \
    --inputVout <vout> \
    --inputAmount <sats> \
    --upgradeProposalTxId <proposal_txid> \
    --fee 500 \
    --out ./upgrade_tx_exec.json
  ```

- Create unsigned sequencer update PSBT:
  ```bash
  via multisig create-update-sequencer \
    --inputTxId <tx_id> \
    --inputVout <vout> \
    --inputAmount <sats> \
    --fee 500 \
    --sequencerAddress <bcrt1...> \
    --out ./upgrade_tx_exec.json
  ```

- Create unsigned governance update PSBT:
  ```bash
  via multisig create-update-governance \
    --inputTxId <tx_id> \
    --inputVout <vout> \
    --inputAmount <sats> \
    --fee 500 \
    --governanceAddress <bcrt1...> \
    --out ./upgrade_tx_exec.json
  ```

- Create unsigned bridge update execution PSBT:
  ```bash
  via multisig create-update-bridge \
    --inputTxId <tx_id> \
    --inputVout <vout> \
    --inputAmount <sats> \
    --proposalTxid <proposal_txid> \
    --fee 500 \
    --out ./upgrade_tx_exec.json
  ```

- Sign and finalize (renamed):
  ```bash
  via multisig sign-tx --privateKey <wif> --out ./upgrade_tx_exec.json
  via multisig finalize-tx --out ./upgrade_tx_exec.json
  via multisig broadcast-tx --rpcUrl http://0.0.0.0:18443 --rpcUser rpcuser --rpcPass rpcpassword
  ```

Constants (OP_RETURN prefixes) exposed for tooling:
- VIA_PROTOCOL:WITHDRAWAL
- VIA_PROTOCOL:UPGRADE
- VIA_PROTOCOL:SEQ
- VIA_PROTOCOL:BRI
- VIA_PROTOCOL:GOV

Sources:
- [`typescript multisig.ts`](infrastructure/via/src/multisig.ts)
- Helpers: [`typescript getNetwork()`](infrastructure/via/src/helpers.ts), [`typescript readJsonFile()`](infrastructure/via/src/helpers.ts), [`typescript writeJsonFile()`](infrastructure/via/src/helpers.ts)
- Guides:
  - Upgrade: [`markdown upgrade.md`](docs/via_guides/upgrade.md)
  - MuSig2 wallet/system control: [`markdown musig2.md`](docs/via_guides/musig2.md)

### Deposit helper requires bridge address

- The debug deposit and example deposit scripts require an explicit bridge address:
  ```bash
  via token deposit \
    --amount 10 \
    --receiver-l2-address 0xYourL2 \
    --bridge-address bcrt1p...
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