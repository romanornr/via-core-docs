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
| Sequencer Keys | Signing L2 blocks and L1 batches | Bitcoin ECDSA private key | Environment variables, secure storage |
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
    let verifier_index = self.signer.signer_index().to_string();

    let private_key = bitcoin::PrivateKey::from_wif(&self.config.private_key)?;
    let secret_key = private_key.inner;

    // Sign timestamp + verifier_index as a JSON object
    let payload = serde_json::json!({
        "timestamp": timestamp,
        "verifier_index": verifier_index,
    });
    let signature = crate::auth::sign_request(&payload, &secret_key)?;

    headers.insert("X-Timestamp", header::HeaderValue::from_str(&timestamp)?);
    headers.insert("X-Verifier-Index", header::HeaderValue::from_str(&verifier_index)?);
    headers.insert("X-Signature", header::HeaderValue::from_str(&signature)?);

    Ok(headers)
}
```

Key configuration parameters:
- `VIA_VERIFIER_PRIVATE_KEY`: Private key for the Verifier (WIF format)
- `VIA_VERIFIER_PUBLIC_KEYS`: Comma-separated list of all Verifier public keys

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

The MuSig2 implementation integrates with Bitcoin's Taproot functionality by applying a taproot tweak to the aggregated public key:

```rust
// via_verifier/lib/via_musig2/src/lib.rs
let mut musig_key_agg_cache = KeyAggContext::new(all_pubkeys).map_err(|e| MusigError::Musig2Error(e.to_string()))?;

let agg_pubkey = musig_key_agg_cache.aggregated_pubkey::<secp256k1_musig2::PublicKey>();
let (xonly_agg_key, _) = agg_pubkey.x_only_public_key();

// Convert to bitcoin XOnlyPublicKey first
let internal_key = bitcoin::XOnlyPublicKey::from_slice(&xonly_agg_key.serialize())?;

// Calculate taproot tweak
let tap_tweak = TapTweakHash::from_key_and_tweak(internal_key, None);
let tweak = tap_tweak.to_scalar();
let tweak_bytes = tweak.to_be_bytes();
let musig2_compatible_tweak = secp256k1_musig2::Scalar::from_be_bytes(tweak_bytes).unwrap();
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

1. **L1 Verifier Config**: Contains verification keys for different circuit types
   - Configured in the genesis configuration
   - Example: `recursion_scheduler_level_vk_hash: 0x14f97b81e54b35fe673d8708cc1a19e1ea5b5e348e12d31e39824ed4f42bbca2`

2. **Protocol Version-Specific Keys**: Different keys for different protocol versions
   - Allows for smooth protocol upgrades
   - Stored in the database with protocol version information

## Sequencer Keys

The Sequencer uses keys for:

1. **Signing L2 Blocks**: Cryptographically signing blocks created by the Sequencer
2. **Creating Inscriptions**: Signing transactions that inscribe batch metadata on Bitcoin

Key configuration:
- Private key stored as an environment variable or in a secure storage system
- Public key distributed to Verifiers for signature verification

## Celestia Keys

The Via L2 system uses Celestia for data availability, which requires keys for:

1. **Authentication**: Authenticating with the Celestia node
2. **Transaction Signing**: Signing transactions that submit data to Celestia

Key configuration:
- Keys stored in the `celestia-keys/keys` directory
- Referenced in the Docker Compose configuration:
  ```yaml
  celestia-node:
    volumes:
      - ./celestia-keys/keys:/home/celestia/keys
    command: celestia light start --keyring.backend test --keyring.keyname via
  ```

## Key Storage and Security

The Via L2 system uses several methods for storing keys securely:

### Environment Variables

Sensitive keys are often stored as environment variables:
- `VIA_VERIFIER_PRIVATE_KEY`: Verifier private key
- `VIA_CELESTIA_CLIENT_AUTH_TOKEN`: Celestia authentication token

### File System

Some keys are stored in the file system:
- Prover setup data keys: `prover/setup-data-gpu-keys.json`
- Celestia keys: `celestia-keys/keys/`

### Database

Some key-related data is stored in the database:
- Verification key hashes
- Public keys of Verifiers

## Key Rotation and Management

The current implementation has limited support for key rotation:

### Verifier Keys

- The set of Verifier keys is currently static
- Defined in the system bootstrapping process
- Changes require a system upgrade

From the `SystemBootstrapping` inscription:
```rust
pub struct SystemBootstrappingInput {
    pub start_block_height: u32,
    pub verifier_p2wpkh_addresses: Vec<BitcoinAddress<NetworkUnchecked>>,
    pub bridge_musig2_address: BitcoinAddress<NetworkUnchecked>,
    pub bootloader_hash: H256,
    pub abstract_account_hash: H256,
}
```

### MuSig2 Bridge Key

- The bridge key is derived from Verifier public keys
- Changes to the Verifier set require a new bridge address
- Funds would need to be migrated from the old bridge to the new one

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

### 3. Static Verifier Set

- The current architecture does not support dynamic addition or removal of Verifiers
- Verifier public keys are configured statically
- Future improvements could implement a mechanism for Verifier rotation or election

### 4. Key Backup and Recovery

- The current system has limited provisions for key backup and recovery
- Future improvements could implement more robust key backup and recovery mechanisms

### 5. Hardware Security Module (HSM) Support

- The current system does not explicitly support HSMs
- Future improvements could add support for storing keys in HSMs for enhanced security

## Conclusion

The Via L2 Bitcoin ZK-Rollup system uses a variety of cryptographic keys for different purposes, from securing the bridge address to verifying ZK proofs. While the current implementation has some limitations, particularly around key rotation and the n-of-n signature scheme, it provides a solid foundation for secure operation of the L2 system.

Future improvements will focus on making the system more decentralized, fault-tolerant, and open to dynamic participation, addressing the current limitations of the key management system.