# Via L2 Bitcoin ZK-Rollup: Key Management

This document describes how cryptographic keys are managed, stored, and used within the Via L2 Bitcoin ZK-Rollup system.

## Table of Contents

1. [Overview of Key Types](#overview-of-key-types)
2. [Verifier Network Keys](#verifier-network-keys)
3. [MuSig2 Bridge Keys](#musig2-bridge-keys)
4. [Prover and Verifier Keys](#prover-and-verifier-keys)
5. [Sequencer Keys](#sequencer-keys)
6. [Celestia Keys](#celestia-keys)
7. [Key Storage and Security](#key-storage-and-security)
8. [Key Rotation and Management](#key-rotation-and-management)
9. [Current Limitations and Future Improvements](#current-limitations-and-future-improvements)

## Overview of Key Types

The Via L2 system uses several types of cryptographic keys for different purposes:

| Key Type | Purpose | Format | Storage |
|----------|---------|--------|---------|
| Verifier Private Keys | Signing attestations and participating in MuSig2 | Bitcoin ECDSA private key (WIF format) | Environment variables, secure storage |
| MuSig2 Aggregated Key | Securing the bridge address | Bitcoin Taproot key | Derived from Verifier public keys |
| Prover Verification Keys | Verifying ZK proofs | Circuit-specific verification keys | Files, database |
| Sequencer (BTC sender) Keys | Signing Bitcoin inscription transactions (batch commitments, proof references) | Bitcoin ECDSA private key (WIF format) | Environment variables, secure storage |
| Celestia Keys | Authenticating with Celestia | Celestia account key | Key files |

## Verifier Network Keys

### Key Generation and Distribution

Each Verifier node in the network has its own private key used for:
1. Signing attestations that are inscribed on Bitcoin
2. Participating in MuSig2 multi-signature transactions for withdrawals
3. Authenticating with the Coordinator node

```rust
// Key generation example from via_verifier/lib/via_musig2/examples/key_generation_setup.rs
fn generate_keypair() -> (SecretKey, PublicKey) {
    let mut rng = OsRng;
    let secp = Secp256k1::new();
    let secret_key = SecretKey::new(&mut rng);
    let public_key = PublicKey::from_secret_key(&secp, &secret_key);
    (secret_key, public_key)
}
```

### Key Configuration

Verifier keys are configured through environment variables and configuration files:

```rust
// via_verifier/node/via_verifier_coordinator/src/verifier/mod.rs
fn create_request_headers(&self) -> anyhow::Result<header::HeaderMap> {
    let mut headers = header::HeaderMap::new();
    let timestamp = chrono::Utc::now().timestamp().to_string();
    let signer = get_signer_with_merkle_root(
        &self.wallet.private_key,
        self.via_bridge_config.verifiers_pub_keys.clone(),
        self.verifier_config.bridge_address_merkle_root(),
    )?;
    let verifier_index = signer.signer_index().to_string();
    let sequencer_version = get_sequencer_version().to_string();

    let private_key = bitcoin::PrivateKey::from_wif(&self.wallet.private_key)?;
    let secret_key = private_key.inner;

    // Sign timestamp + verifier_index + sequencer_version as a JSON object
    let payload = serde_json::json!({
        "timestamp": timestamp,
        "verifier_index": verifier_index,
        "sequencer_version": sequencer_version
    });
    let signature = crate::auth::sign_request(&payload, &secret_key)?;

    headers.insert("X-Timestamp", header::HeaderValue::from_str(&timestamp)?);
    headers.insert(
        "X-Verifier-Index",
        header::HeaderValue::from_str(&verifier_index)?,
    );
    headers.insert("X-Signature", header::HeaderValue::from_str(&signature)?);
    headers.insert(
        "X-Sequencer-Version",
        header::HeaderValue::from_str(&sequencer_version)?,
    );

    Ok(headers)
}
```

Note the current shape: the private key comes from a `ViaWallet` (not a config field), the verifier's index is derived by matching its public key against `via_bridge_config.verifiers_pub_keys`, and requests also carry the sequencer version. `sign_request` itself is a SHA-256-then-ECDSA signature, base64-encoded (`via_verifier/node/via_verifier_coordinator/src/auth.rs`):

```rust
/// Signs a request payload using the verifier's private key.
pub fn sign_request<T: Serialize>(payload: &T, secret_key: &SecretKey) -> anyhow::Result<String> {
    let secp = Secp256k1::new();

    // Serialize and hash the payload.
    let payload_bytes =
        serde_json::to_vec(payload).with_context(|| "Failed to serialize payload")?;
    let hash = Sha256::digest(&payload_bytes);
    let message =
        Message::from_digest_slice(hash.as_ref()).with_context(|| "Hash is not 32 bytes")?;

    // Sign the message.
    let sig = secp.sign_ecdsa(&message, secret_key);
    // Encode the compact 64-byte signature in base64.
    let sig_bytes = sig.serialize_compact();
    Ok(base64::engine::general_purpose::STANDARD.encode(sig_bytes))
}
```

Key configuration parameters (real env variable names):
- `VIA_VERIFIER_PRIVATE_KEY` and `VIA_VERIFIER_WALLET_ADDRESS`: the verifier's WIF private key and address, loaded into `ViaWallets.vote_operator` (`core/lib/env_config/src/via_wallets.rs`)
- `VIA_BRIDGE_VERIFIERS_PUB_KEYS` and `VIA_BRIDGE_BRIDGE_ADDRESS`: the verifier public key set and bridge address, loaded into `ViaBridgeConfig` with the `VIA_BRIDGE_` envy prefix (`core/lib/env_config/src/via_bridge.rs`)

```rust
// core/lib/config/src/configs/via_bridge.rs
#[derive(Debug, Serialize, Default, Deserialize, Clone, PartialEq)]
pub struct ViaBridgeConfig {
    /// The verifiers public keys.
    pub verifiers_pub_keys: Vec<String>,

    /// The bridge address.
    pub bridge_address: String,
}
```

## MuSig2 Bridge Keys

### Key Aggregation

The MuSig2 bridge address is secured by an aggregated key derived from all Verifier public keys:

```rust
// via_verifier/lib/via_musig2/examples/key_generation_setup.rs
fn create_bridge_address(
    pubkeys: Vec<PublicKey>,
) -> Result<BitcoinAddress, Box<dyn std::error::Error>> {
    let secp = bitcoin::secp256k1::Secp256k1::new();

    let musig_key_agg_cache = KeyAggContext::new(pubkeys)?;

    let agg_pubkey = musig_key_agg_cache.aggregated_pubkey::<secp256k1_musig2::PublicKey>();
    let (xonly_agg_key, _) = agg_pubkey.x_only_public_key();

    // Convert to bitcoin XOnlyPublicKey first
    let internal_key = bitcoin::XOnlyPublicKey::from_slice(&xonly_agg_key.serialize())?;

    // Use internal_key for address creation
    let address = BitcoinAddress::p2tr(&secp, internal_key, None, Network::Regtest);

    Ok(address)
}
```

### Taproot Integration

The MuSig2 implementation integrates with Bitcoin's Taproot functionality by applying a taproot tweak to the aggregated public key. The tweak takes an optional `merkle_root`: `None` gives a key-path-only bridge address, while a `Some(TapNodeHash)` (configured via `ViaVerifierConfig.bridge_address_merkle_root`) commits to a script tree as well:

```rust
// via_verifier/lib/via_musig2/src/lib.rs (inside Signer::new)
let mut musig_key_agg_cache =
    KeyAggContext::new(all_pubkeys).map_err(|e| MusigError::Musig2Error(e.to_string()))?;

let agg_pubkey = musig_key_agg_cache.aggregated_pubkey::<secp256k1_musig2::PublicKey>();
let (xonly_agg_key, _) = agg_pubkey.x_only_public_key();

// Convert to bitcoin XOnlyPublicKey first
let internal_key = bitcoin::XOnlyPublicKey::from_slice(&xonly_agg_key.serialize())
    .map_err(|e| {
        MusigError::Musig2Error(format!(
            "Failed to convert to bitcoin XOnlyPublicKey: {}",
            e
        ))
    })?;

// Calculate taproot tweak
let tap_tweak = TapTweakHash::from_key_and_tweak(internal_key, merkle_root);
let tweak = tap_tweak.to_scalar();
let tweak_bytes = tweak.to_be_bytes();
let musig2_compatible_tweak = secp256k1_musig2::Scalar::from_be_bytes(tweak_bytes)
    .with_context(|| TAPROOT_TWEAK_SCALAR_RANGE_ERR)
    .map_err(|e| MusigError::Musig2Error(e.to_string()))?;
// Apply tweak to the key aggregation context before signing
musig_key_agg_cache = musig_key_agg_cache
    .with_xonly_tweak(musig2_compatible_tweak)
    .map_err(|e| MusigError::Musig2Error(format!("Failed to apply tweak: {}", e)))?;
```

## Prover and Verifier Keys

### Prover Keys

The Prover uses several types of keys for generating and verifying zero-knowledge proofs:

1. **Setup Data Keys**: Used for initializing the proving system
   - Location: `prover/setup-data-gpu-keys.json`
   - Format: JSON file containing circuit-specific setup data

2. **Circuit-Specific Keys**: Used for specific circuit types
   - Generated during system initialization
   - Stored in the database and/or file system

### Verification Keys

Verification keys are used to verify the correctness of ZK proofs:

1. **SNARK wrapper verification key hash**: the hash the verifiers check final proofs against
   - Distributed on-chain as `snark_wrapper_vk_hash` in the `SystemBootstrapping` inscription (see the struct below) and updated by `SystemContractUpgrade` messages on protocol upgrades
   - Matched against the verification key baked into the versioned `via_verification` modules (`via_verifier/lib/via_verification/src/`, one module per protocol version such as `version_27`/`version_28`)

2. **Protocol Version-Specific Keys**: Different keys for different protocol versions
   - Allows for smooth protocol upgrades
   - Old keys are retained so historical proofs remain verifiable

## Sequencer Keys

The sequencer's Bitcoin key is used for one thing: signing the inscription transactions that commit batch metadata and proof references to Bitcoin (the `Inscriber` is constructed with a `signer_private_key: &str`, `core/lib/via_btc_client/src/inscriber/mod.rs`). L2 blocks themselves are not signed with a Bitcoin key.

The key is loaded from the environment into `ViaWallets` alongside the verifier wallet (`core/lib/env_config/src/via_wallets.rs`):

```rust
impl FromEnv for ViaWallets {
    fn from_env() -> anyhow::Result<Self> {
        let wallets = Wallets::from_env()?;

        let btc_sender_address = pk_from_env(
            "VIA_BTC_SENDER_WALLET_ADDRESS",
            "Malformed operator address",
        )?;
        let btc_sender_pk = pk_from_env("VIA_BTC_SENDER_PRIVATE_KEY", "Malformed operator pk")?;

        let verifier_address =
            pk_from_env("VIA_VERIFIER_WALLET_ADDRESS", "Malformed verifier address")?;
        let verifier_pk = pk_from_env("VIA_VERIFIER_PRIVATE_KEY", "Malformed verifier pk")?;

        Ok(Self {
            state_keeper: wallets.state_keeper,
            token_multiplier_setter: wallets.token_multiplier_setter,
            btc_sender: Some(ViaWallet {
                address: btc_sender_address.unwrap_or_default(),
                private_key: btc_sender_pk.clone().unwrap_or_default(),
            }),
            vote_operator: Some(ViaWallet {
                address: verifier_address.unwrap_or_default(),
                private_key: verifier_pk.unwrap_or_default(),
            }),
        })
    }
}
```

The sequencer's address is announced on-chain in the `SystemBootstrapping` inscription (`sequencer_address` field) and persisted in `SystemWallets`, so verifiers and indexers only accept protocol inscriptions from that address.

## Celestia Keys

The Via L2 system uses Celestia for data availability, which requires keys for:

1. **Authentication**: Authenticating with the Celestia node
2. **Transaction Signing**: Signing transactions that submit data to Celestia

Key configuration:
- The light node's keyring is seeded from `./etc/docker-data/celestia/keys` by an init container, then the node starts with the `via` key (`docker-compose-via.yml`):
  ```yaml
  celestia-node-init:
    image: &celestia-image "ghcr.io/celestiaorg/celestia-node:v0.29.3-mocha"
    volumes:
      - ./etc/docker-data/celestia/keys:/keys
      - celestia:/home/celestia
    entrypoint:
      - /bin/bash
      - -c
      - |
        cp -r /keys /home/celestia
        celestia light init
        ...

  celestia-node:
    image: *celestia-image
    depends_on:
      celestia-node-init:
        condition: service_completed_successfully
    volumes:
      - celestia:/home/celestia
    command: celestia light start --core.ip=rpc-mocha.pops.one --p2p.network=mocha --keyring.backend=test --keyring.keyname=via
    ports:
      - '26658:26658'
    restart: unless-stopped
  ```
- The node itself authenticates to Celestia with that keyring; Via services authenticate to the node's RPC with a token, carried in `ViaDASecrets` (`core/lib/config/src/configs/via_secrets.rs`, loaded with the `VIA_CELESTIA_CLIENT_` env prefix):
  ```rust
  #[derive(Clone, Deserialize, PartialEq)]
  pub struct ViaDASecrets {
      /// URL for the Celestia node RPC.
      pub api_node_url: SensitiveUrl,

      /// AUTH token for the Celestia node RPC.
      pub auth_token: String,
  }
  ```

## Key Storage and Security

The Via L2 system uses several methods for storing keys securely:

### Environment Variables

Sensitive keys are stored as environment variables:
- `VIA_BTC_SENDER_PRIVATE_KEY`: sequencer/btc-sender inscription key (WIF)
- `VIA_VERIFIER_PRIVATE_KEY`: verifier private key (WIF)
- `VIA_CELESTIA_CLIENT_AUTH_TOKEN`: Celestia node RPC auth token

In-process, these land in `ViaWallet` values whose `Debug` implementation deliberately omits the private key, so wallets can be logged without leaking secrets (`core/lib/config/src/configs/via_wallets.rs`):

```rust
#[derive(Default, Clone, PartialEq)]
pub struct ViaWallet {
    pub address: String,
    pub private_key: String,
}

impl fmt::Debug for ViaWallet {
    fn fmt(&self, formatter: &mut fmt::Formatter<'_>) -> fmt::Result {
        formatter
            .debug_struct("Secret")
            .field("address", &self.address)
            .finish()
    }
}
```

### File System

Some keys are stored in the file system:
- Prover setup data keys: `prover/setup-data-gpu-keys.json`
- Celestia keys: `celestia-keys/keys/`

### Database

Some key-related data is stored in the database:
- Verification key hashes
- Public keys of Verifiers

## Key Rotation and Management

### Verifier Keys and System Wallets

The initial key set is defined in the system bootstrapping process. The current `SystemBootstrapping` inscription input (`core/lib/via_btc_client/src/types.rs`):

```rust
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
pub struct SystemBootstrappingInput {
    pub start_block_height: u32,
    pub protocol_version: ProtocolSemanticVersion,
    pub bootloader_hash: H256,
    pub abstract_account_hash: H256,
    pub snark_wrapper_vk_hash: H256,
    pub evm_emulator_hash: H256,
    pub governance_address: BitcoinAddress<NetworkUnchecked>,
    pub sequencer_address: BitcoinAddress<NetworkUnchecked>,
    pub bridge_musig2_address: BitcoinAddress<NetworkUnchecked>,
    pub verifier_p2wpkh_addresses: Vec<BitcoinAddress<NetworkUnchecked>>,
}
```

Rotation is no longer purely static: the bootstrapped addresses become the initial `SystemWallets` state (sequencer, governance, bridge, verifiers), which governance can update at runtime through `VIA_PROTOCOL:SEQ` (`UpdateSequencerProposal`) and `VIA_PROTOCOL:BRI` (`UpgradeBridgeProposal`) inscriptions. The indexer applies these via `update_system_wallets()`, nodes persist the resulting state in the `via_wallets` table, and the L1 indexer even re-parses the affected block range so messages are interpreted under the new wallet set. Changing the verifier set or bridge key therefore means broadcasting a governance inscription, not re-bootstrapping the system.

### MuSig2 Bridge Key

- The bridge key is derived from Verifier public keys
- Changes to the Verifier set produce a new aggregated key and hence a new bridge address, announced via the `UpgradeBridgeProposal` governance flow
- Funds still need to be migrated from the old bridge address to the new one (see the `transfer_utxos_from_bridge.rs` example in `via_verifier/lib/via_musig2/examples/`)

### Protocol Version Keys

- The system supports different keys for different protocol versions
- New keys can be introduced with protocol upgrades
- Old keys are retained for verifying historical proofs

## Current Limitations and Future Improvements

The current key management system has several limitations:

### 1. Fixed Coordinator Role

- One of the Verifiers holds the Coordinator role
- This creates a potential single point of failure
- Future improvements could implement a dynamic Coordinator selection mechanism

### 2. N-of-N Signature Scheme

- All Verifiers must participate in the signing process
- If any single Verifier becomes unresponsive, withdrawals may be delayed
- Future improvements could implement an m-of-n threshold signature scheme

### 3. Governance-Gated Verifier Set

- Verifier set changes are possible at runtime, but only through governance inscriptions (`UpgradeBridgeProposal`); there is no permissionless join/leave mechanism
- Each node's own signing identity (`VIA_VERIFIER_PRIVATE_KEY`) and the peer set (`VIA_BRIDGE_VERIFIERS_PUB_KEYS`) are still configured via environment
- Future improvements could implement open Verifier rotation or election

### 4. Key Backup and Recovery

- The current system has limited provisions for key backup and recovery
- Future improvements could implement more robust key backup and recovery mechanisms

### 5. Hardware Security Module (HSM) Support

- The current system does not explicitly support HSMs
- Future improvements could add support for storing keys in HSMs for enhanced security

## Conclusion

The Via L2 Bitcoin ZK-Rollup system uses a variety of cryptographic keys for different purposes, from securing the bridge address to verifying ZK proofs. While the current implementation has some limitations, particularly around key rotation and the n-of-n signature scheme, it provides a solid foundation for secure operation of the L2 system.

Future improvements will focus on making the system more decentralized, fault-tolerant, and open to dynamic participation, addressing the current limitations of the key management system.