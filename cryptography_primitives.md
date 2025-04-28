# Cryptography Primitives in Via L2 Bitcoin ZK-Rollup

This document outlines the core cryptographic primitives used in the Via L2 Bitcoin ZK-Rollup system, focusing on the fundamental cryptographic libraries and functions beyond the ZKP system and MuSig2 implementation (which are documented separately).

## 1. Hash Functions

The Via L2 system uses multiple hash functions for different purposes, implemented in the `core/lib/crypto_primitives/src/hasher/` directory.

### 1.1 Blake2s-256

**Library**: `blake2` crate

**Implementation**: `core/lib/crypto_primitives/src/hasher/blake2.rs`

```rust
use blake2::{Blake2s256, Digest};
use zksync_basic_types::H256;

use crate::hasher::Hasher;

#[derive(Default, Clone, Debug)]
pub struct Blake2Hasher;

impl Hasher for Blake2Hasher {
    type Hash = H256;

    fn hash_bytes(&self, value: &[u8]) -> H256 {
        let mut hasher = Blake2s256::new();
        hasher.update(value);
        H256(hasher.finalize().into())
    }

    fn compress(&self, lhs: &H256, rhs: &H256) -> H256 {
        let mut hasher = Blake2s256::new();
        hasher.update(lhs.as_ref());
        hasher.update(rhs.as_ref());
        H256(hasher.finalize().into())
    }
}
```

**Usage**: 
- Primary hash function for the Merkle tree implementation
- Used for state hashing and verification
- Default hasher for the main Merkle tree implementation

### 1.2 Keccak-256

**Library**: `web3::keccak256` from `zksync_basic_types`

**Implementation**: `core/lib/crypto_primitives/src/hasher/keccak.rs`

```rust
use zksync_basic_types::{web3::keccak256, H256};

use crate::hasher::Hasher;

#[derive(Default, Clone, Debug)]
pub struct KeccakHasher;

impl Hasher for KeccakHasher {
    type Hash = H256;

    fn hash_bytes(&self, value: &[u8]) -> Self::Hash {
        H256(keccak256(value))
    }

    fn compress(&self, lhs: &Self::Hash, rhs: &Self::Hash) -> Self::Hash {
        let mut bytes = [0_u8; 64];
        bytes[..32].copy_from_slice(&lhs.0);
        bytes[32..].copy_from_slice(&rhs.0);
        H256(keccak256(&bytes))
    }
}
```

**Usage**:
- Used for Ethereum compatibility (address derivation, transaction hashing)
- Default hasher for the mini Merkle tree implementation
- Used in EIP-712 signature implementation

### 1.3 SHA-256

**Library**: `sha2` crate

**Implementation**: `core/lib/crypto_primitives/src/hasher/sha256.rs`

```rust
use sha2::{Digest, Sha256};
use zksync_basic_types::H256;

use crate::hasher::Hasher;

#[derive(Debug, Default, Clone, Copy)]
pub struct Sha256Hasher;

impl Hasher for Sha256Hasher {
    type Hash = H256;

    fn hash_bytes(&self, value: &[u8]) -> Self::Hash {
        let mut sha256 = Sha256::new();
        sha256.update(value);
        H256(sha256.finalize().into())
    }

    fn compress(&self, lhs: &Self::Hash, rhs: &Self::Hash) -> Self::Hash {
        let mut hasher = Sha256::new();
        hasher.update(lhs.as_ref());
        hasher.update(rhs.as_ref());
        H256(hasher.finalize().into())
    }
}
```

**Usage**:
- Alternative hash function available for specific components
- Used for Bitcoin-related operations where SHA-256 is the standard

## 2. Merkle Tree Implementations

The Via L2 system uses two different Merkle tree implementations for different purposes.

### 2.1 Main Merkle Tree

**Library**: Custom implementation based on Jellyfish Merkle tree

**Implementation**: `core/lib/merkle_tree/`

The main Merkle tree is a binary Merkle tree implemented using the amortized radix-16 Merkle tree (AR16MT) approach described in the Jellyfish Merkle tree white paper. Unlike the original Jellyfish Merkle tree, this implementation uses a vanilla binary tree hashing algorithm to make it easier for circuit creation.

**Key characteristics**:
- Tree depth: 256
- Default hash function: Blake2s-256
- Persistent and versioned (each version corresponds to a block number)
- Optimized for I/O efficiency by storing in radix-16 format
- Paths of internal nodes that do not fork are removed for optimization

**Hashing specification**:
```rust
// Hash of a vacant leaf
hash([0_u8; 40])

// Hash of an occupied leaf
hash(u64::to_be_bytes(leaf_index) ++ value_hash)

// Hash of an internal node
hash(left_child_hash ++ right_child_hash)
```

### 2.2 Mini Merkle Tree

**Library**: Custom implementation

**Implementation**: `core/lib/mini_merkle_tree/`

A simple in-memory binary Merkle tree implementation for smaller, bounded-depth trees.

**Key characteristics**:
- Bounded depth (up to 1,024 leaves)
- Uses Keccak-256 hash function
- Left-leaning (can specify tree size larger than number of leaves)
- Dynamic size (can grow by factor of 2)
- Optimized for queries on rightmost leaves with caching of leftmost leaves

**Usage**:
- Used for L2ToL1Log operations
- Suitable for scenarios requiring fast in-memory Merkle tree operations

## 3. Signature Schemes

### 3.1 ECDSA (secp256k1)

**Library**: `secp256k1` crate

**Implementation**: `core/lib/crypto_primitives/src/ecdsa_signature.rs`

The ECDSA implementation is based on the secp256k1 curve (the same used in Bitcoin and Ethereum). It provides functionality for:
- Key generation
- Signing messages
- Verifying signatures
- Converting between different formats

```rust
pub struct K256PrivateKey(SecretKey);

impl K256PrivateKey {
    pub fn from_bytes(bytes: H256) -> Result<Self, Error> {
        Ok(Self(SecretKey::from_slice(bytes.as_bytes())?))
    }

    pub fn random() -> Self {
        Self::random_using(&mut rand::rngs::OsRng)
    }

    pub fn sign_web3(&self, message: &H256, chain_id: Option<u64>) -> web3::Signature {
        // Implementation details...
    }
}
```

**Usage**:
- Used for L2 transaction signing
- Used for Ethereum-compatible operations

### 3.2 EIP-712 Signatures

**Library**: Custom implementation

**Implementation**: `core/lib/crypto_primitives/src/eip712_signature/`

EIP-712 is a standard for hashing and signing typed structured data in Ethereum. The implementation provides:
- Typed data structure definition
- Hashing of structured data
- Domain separation

```rust
pub trait EIP712TypedStructure {
    const TYPE_NAME: &'static str;
    fn build_structure<BUILDER: StructBuilder>(&self, builder: &mut BUILDER);
    fn hash_struct(&self) -> H256 {
        // Implementation details...
    }
}
```

**Usage**:
- Used for structured data signing in Ethereum-compatible operations
- Provides type safety for signed messages

### 3.3 Packed Ethereum Signatures

**Library**: Custom implementation

**Implementation**: `core/lib/crypto_primitives/src/packed_eth_signature.rs`

A wrapper around Ethereum signatures with additional functionality for:
- Serialization and deserialization
- Handling of the `v` value (recovery ID)
- Chain ID support
- EIP-712 signature support

```rust
pub struct PackedEthSignature(ETHSignature);

impl PackedEthSignature {
    pub fn sign_typed_data(
        private_key: &K256PrivateKey,
        domain: &Eip712Domain,
        typed_struct: &impl EIP712TypedStructure,
    ) -> Result<PackedEthSignature, ParityCryptoError> {
        // Implementation details...
    }
}
```

**Usage**:
- Used for Ethereum-compatible transaction signing
- Supports EIP-712 typed data signing

### 3.4 MuSig2 (Multi-Signatures)

**Library**: `musig2` and `secp256k1_musig2` crates

**Implementation**: `via_verifier/lib/via_musig2/src/lib.rs`

MuSig2 is a multi-signature scheme based on Schnorr signatures. It allows multiple signers to collaborate to produce a single signature. The implementation includes:

- Two-round signing process
- Taproot tweaking support
- Transaction building with MuSig2 signatures

**Note**: The MuSig2 implementation is documented in detail in `musig2_implementation.md`.

## 4. Other Cryptographic Building Blocks

### 4.1 Transaction Building with Taproot

**Library**: `bitcoin` crate

**Implementation**: `via_verifier/lib/via_musig2/src/transaction_builder.rs`

The transaction builder provides functionality for:
- Creating Bitcoin transactions with OP_RETURN data
- Calculating transaction fees
- Selecting UTXOs
- Generating TapSighash for signing

```rust
pub fn get_tr_sighash(&self, unsigned_tx: &UnsignedBridgeTx) -> anyhow::Result<TapSighash> {
    let mut sighash_cache = SighashCache::new(&unsigned_tx.tx);
    let sighash_type = TapSighashType::All;
    // Implementation details...
}
```

**Usage**:
- Used for building and signing Bitcoin transactions
- Supports Taproot transactions with MuSig2 signatures

## Conclusion

The Via L2 Bitcoin ZK-Rollup system uses a variety of cryptographic primitives to ensure security, efficiency, and compatibility with both Bitcoin and Ethereum ecosystems. The core cryptographic building blocks include:

1. **Hash Functions**: Blake2s-256, Keccak-256, and SHA-256
2. **Merkle Trees**: A main Merkle tree based on the Jellyfish Merkle tree and a mini Merkle tree for in-memory operations
3. **Signature Schemes**: ECDSA (secp256k1), EIP-712 signatures, and MuSig2 multi-signatures
4. **Transaction Building**: Support for Bitcoin transactions with Taproot and MuSig2

These cryptographic primitives work together to provide the security foundation for the Via L2 system, enabling secure state transitions, transaction validation, and cross-chain operations.