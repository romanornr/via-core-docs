# Cryptography Primitives in Via L2 Bitcoin ZK-Rollup

This document outlines the core cryptographic primitives used in the Via L2 Bitcoin ZK-Rollup system, focusing on the fundamental cryptographic libraries and functions beyond the ZKP system and MuSig2 implementation (which are documented separately).

## 1. Hash Functions

The Via L2 system uses multiple hash functions for different purposes, implemented in the `core/lib/crypto_primitives/src/hasher/` directory.

### 1.1 Blake2s-256

**Library**: `blake2` crate

**Implementation**: `core/lib/crypto_primitives/src/hasher/blake2.rs`

```rust
// core/lib/crypto_primitives/src/hasher/blake2.rs
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
// core/lib/crypto_primitives/src/hasher/keccak.rs
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
// core/lib/crypto_primitives/src/hasher/sha256.rs
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
- The `Sha256Hasher` struct itself currently has no consumers in the via-core codebase; it is provided as an alternative `Hasher` implementation
- SHA-256 as an algorithm is used elsewhere via the `sha2` crate directly, for example the verifier coordinator REST API authentication, which signs the SHA-256 digest of the JSON payload with ECDSA and encodes the compact signature in base64:

```rust
// via_verifier/node/via_verifier_coordinator/src/auth.rs
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

## 2. Merkle Tree Implementations

The Via L2 system uses two different Merkle tree implementations for different purposes.

### 2.1 Main Merkle Tree

**Library**: Custom implementation based on Jellyfish Merkle tree

**Implementation**: `core/lib/merkle_tree/`

The main Merkle tree is a binary Merkle tree implemented using the amortized radix-16 Merkle tree (AR16MT) approach described in the Jellyfish Merkle tree white paper. Unlike the original Jellyfish Merkle tree, this implementation uses a vanilla binary tree hashing algorithm to make it easier for circuit creation.

**Key characteristics**:
- Tree depth: 256
- Default hash function: Blake2s-256 (`Blake2Hasher`, see `MerkleTree<DB, H = Blake2Hasher>` in `core/lib/merkle_tree/src/lib.rs`)
- Persistent and versioned (in the domain wrapper `ZkSyncTree`, each tree version corresponds to an L1 batch number; see `core/lib/merkle_tree/src/domain.rs`)
- Optimized for I/O efficiency by storing in radix-16 format

**Hashing specification** (verbatim from the crate docs):

```rust
// core/lib/merkle_tree/src/lib.rs
//! # Tree hashing specification
//!
//! A tree is hashed as if it was a full binary Merkle tree with `2^256` leaves:
//!
//! - Hash of a vacant leaf is `hash([0_u8; 40])`, where `hash` is the hash function used
//!   (Blake2s-256).
//! - Hash of an occupied leaf is `hash(u64::to_be_bytes(leaf_index) ++ value_hash)`,
//!   where `leaf_index` is a 1-based index of the leaf key provided when the leaf is inserted / updated,
//!   `++` is byte concatenation.
//! - Hash of an internal node is `hash(left_child_hash ++ right_child_hash)`.
```

### 2.2 Mini Merkle Tree

**Library**: Custom implementation

**Implementation**: `core/lib/mini_merkle_tree/`

A simple in-memory binary Merkle tree implementation for smaller, bounded-depth trees.

**Key characteristics**:
- Bounded depth: no more than 32, i.e. up to `2^32` leaves
- Uses Keccak-256 hash function by default
- Left-leaning (can specify tree size larger than number of leaves)
- Dynamic size (can grow by factor of 2, does not shrink)
- Optimized for queries on rightmost leaves with caching (trimming) of leftmost leaves

```rust
// core/lib/mini_merkle_tree/src/lib.rs
/// Maximum supported depth of the tree. 32 corresponds to `2^32` elements in the tree, which
/// we unlikely to ever hit.
const MAX_TREE_DEPTH: usize = 32;
```

```rust
// core/lib/mini_merkle_tree/src/lib.rs
#[derive(Debug, Clone)]
pub struct MiniMerkleTree<L, H = KeccakHasher> {
    hasher: H,
    /// Stores untrimmed (uncached) leaves of the tree.
    hashes: VecDeque<H256>,
    /// Size of the tree. Always a power of 2.
    /// If it is greater than `self.start_index + self.hashes.len()`, the remaining leaves are empty.
    binary_tree_size: usize,
    /// Index of the leftmost untrimmed leaf.
    start_index: usize,
    /// Left subset of the Merkle path to the first untrimmed leaf (i.e., a leaf with index `self.start_index`).
    /// Merkle path starts from the bottom of the tree and goes up.
    /// Used to fill in data for trimmed tree leaves when computing Merkle paths and the root hash.
    /// Because only the left subset of the path is used, the cache is not invalidated when new leaves are
    /// pushed into the tree. If all leaves are trimmed, cache is the left subset of the Merkle path to
    /// the next leaf to be inserted, which still has index `self.start_index`.
    cache: Vec<Option<H256>>,
    /// Leaf type marker
    _leaf: PhantomData<L>,
}
```

**Usage**:
- Used to compute the L2-to-L1 logs Merkle root (`l2_l1_logs_merkle_root`) in batch commitments (`core/lib/types/src/commitment/mod.rs`)
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
// core/lib/crypto_primitives/src/ecdsa_signature.rs
/// secp256k1 private key wrapper.
///
/// Provides a safe to use `Debug` implementation (outputting the address corresponding to the key).
/// The key is zeroized on drop.
#[derive(Clone, PartialEq)]
pub struct K256PrivateKey(SecretKey);
```

```rust
// core/lib/crypto_primitives/src/ecdsa_signature.rs (from impl K256PrivateKey)
    /// Converts a 32-byte array into a key.
    ///
    /// # Errors
    ///
    /// Returns an error if the deserialized scalar (as a big-endian number) is zero or is greater or equal
    /// than the secp256k1 group order. The probability of this is negligible if the bytes are random.
    pub fn from_bytes(bytes: H256) -> Result<Self, Error> {
        Ok(Self(SecretKey::from_slice(bytes.as_bytes())?))
    }

    /// Generates a random private key using the OS RNG.
    pub fn random() -> Self {
        Self::random_using(&mut rand::rngs::OsRng)
    }

    /// Generates a random private key using the provided RNG.
    pub fn random_using(rng: &mut impl rand::Rng) -> Self {
        loop {
            if let Ok(this) = Self::from_bytes(H256::random_using(rng)) {
                return this;
            }
        }
    }

    pub fn sign_web3(&self, message: &H256, chain_id: Option<u64>) -> web3::Signature {
        let message = secp256k1::Message::from_slice(message.as_bytes()).unwrap();
        let (recovery_id, signature) = SECP256K1
            .sign_ecdsa_recoverable(&message, &self.0)
            .serialize_compact();

        let standard_v = recovery_id.to_i32() as u64;
        let v = if let Some(chain_id) = chain_id {
            // When signing with a chain ID, add chain replay protection.
            standard_v + 35 + chain_id * 2
        } else {
            // Otherwise, convert to 'Electrum' notation.
            standard_v + 27
        };
        let r = H256::from_slice(&signature[..32]);
        let s = H256::from_slice(&signature[32..]);

        web3::Signature { r, s, v }
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
// core/lib/crypto_primitives/src/eip712_signature/typed_structure.rs
/// Interface for defining the structure for the EIP712 signature.
pub trait EIP712TypedStructure {
    const TYPE_NAME: &'static str;

    fn build_structure<BUILDER: StructBuilder>(&self, builder: &mut BUILDER);

    fn encode_type(&self) -> String {
        let mut builder = EncodeBuilder::new();
        self.build_structure(&mut builder);

        builder.encode_type(Self::TYPE_NAME)
    }

    fn encode_data(&self) -> Vec<H256> {
        let mut builder = EncodeBuilder::new();
        self.build_structure(&mut builder);

        builder.encode_data()
    }

    fn hash_struct(&self) -> H256 {
        // `hashStruct(s : 𝕊) = keccak256(keccak256(encodeType(typeOf(s))) ‖ encodeData(s)).`
        let type_hash = {
            let encode_type = self.encode_type();
            keccak256(encode_type.as_bytes())
        };
        let encode_data = self.encode_data();

        let mut bytes = Vec::new();
        bytes.extend_from_slice(&type_hash);
        for data in encode_data {
            bytes.extend_from_slice(data.as_bytes());
        }

        keccak256(&bytes).into()
    }

    fn get_json_types(&self) -> Vec<Value> {
        let mut builder = EncodeBuilder::new();
        self.build_structure(&mut builder);

        builder.get_json_types(Self::TYPE_NAME)
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
// core/lib/crypto_primitives/src/packed_eth_signature.rs
#[derive(Debug, Clone, PartialEq, Eq, Default)]
pub struct PackedEthSignature(ETHSignature);
```

```rust
// core/lib/crypto_primitives/src/packed_eth_signature.rs (from impl PackedEthSignature)
    /// Signs typed struct using Ethereum private key by EIP-712 signature standard.
    /// Result of this function is the equivalent of RPC calling `eth_signTypedData`.
    pub fn sign_typed_data(
        private_key: &K256PrivateKey,
        domain: &Eip712Domain,
        typed_struct: &impl EIP712TypedStructure,
    ) -> Result<PackedEthSignature, ParityCryptoError> {
        let signed_bytes = H256::from(Self::typed_data_to_signed_bytes(domain, typed_struct).0);
        let signature = sign(private_key, &signed_bytes)?;
        Ok(PackedEthSignature(signature))
    }

    pub fn typed_data_to_signed_bytes(
        domain: &Eip712Domain,
        typed_struct: &impl EIP712TypedStructure,
    ) -> H256 {
        let mut bytes = Vec::new();
        bytes.extend_from_slice("\x19\x01".as_bytes());
        bytes.extend_from_slice(domain.hash_struct().as_bytes());
        bytes.extend_from_slice(typed_struct.hash_struct().as_bytes());
        keccak256(&bytes).into()
    }
```

**Usage**:
- Used for Ethereum-compatible transaction signing
- Supports EIP-712 typed data signing

### 3.4 MuSig2 (Multi-Signatures)

**Library**: `musig2` crate, plus the `secp256k1` crate imported under the alias `secp256k1_musig2`:

```toml
# via_verifier/lib/via_musig2/Cargo.toml
musig2 = "0.2.0"
secp256k1_musig2 = { package = "secp256k1", version = "0.30.0", features = [
    "rand",
    "hashes",
] }
```

**Implementation**: `via_verifier/lib/via_musig2/src/lib.rs`

MuSig2 is a multi-signature scheme based on Schnorr signatures. It allows multiple signers to collaborate to produce a single signature. The implementation includes:

- Two-round signing process (`FirstRound` / `SecondRound` from the `musig2` crate, with `KeyAggContext` for key aggregation)
- Taproot tweaking support (`TapTweakHash::from_key_and_tweak(internal_key, merkle_root)`)
- Transaction building with MuSig2 signatures

**Note**: The MuSig2 implementation is documented in detail in `musig2_implementation.md`.

## 4. Other Cryptographic Building Blocks

### 4.1 Transaction Building with Taproot

**Library**: `bitcoin` crate

**Implementation**: `via_verifier/lib/via_musig2/src/transaction_builder.rs`

The transaction builder provides functionality for:
- Creating Bitcoin transactions with OP_RETURN data (`build_transaction_with_op_return`, `create_op_return_script`)
- Calculating transaction fees
- Selecting UTXOs (`select_utxos`)
- Generating taproot key-spend signature hashes for all inputs

```rust
// via_verifier/lib/via_musig2/src/transaction_builder.rs
    /// Generates taproot signature hashes for all inputs
    #[instrument(skip(self, unsigned_tx), target = "bitcoin_transaction_builder")]
    pub fn get_tr_sighashes(&self, unsigned_tx: &UnsignedBridgeTx) -> Result<Vec<Vec<u8>>> {
        let mut sighash_cache = SighashCache::new(&unsigned_tx.tx);
        let sighash_type = TapSighashType::All;

        let txout_list: Vec<TxOut> = unsigned_tx
            .utxos
            .iter()
            .map(|(_, txout)| txout.clone())
            .collect();

        let mut sighashes = Vec::new();
        for (i, _) in txout_list.iter().enumerate() {
            let sighash = sighash_cache
                .taproot_key_spend_signature_hash(i, &Prevouts::All(&txout_list), sighash_type)
                .context("Error taproot_key_spend_signature_hash")?;
            sighashes.push(sighash.to_raw_hash().to_byte_array().to_vec());
        }

        Ok(sighashes)
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