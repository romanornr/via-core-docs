# Via L2 Examples and Utilities Documentation

## Overview

This document catalogs the example programs and developer utilities that actually ship in the via-core repository. Runnable Rust examples live in four crates (`via_btc_client`, `via_verification`, `via_musig2`, `via_withdrawal_client`), TypeScript developer tooling lives in the `via` CLI under `infrastructure/via/`, and auxiliary binaries live under `core/bin/`.

Rust examples are run with cargo from the repo root:

```bash
cargo run --example <name> -p <crate> [-- <args>]
# e.g.
cargo run --example fee_rate -p via_btc_client -- true
cargo run --example key_generation_setup -p via_musig2 -- contributor
```

## Table of Contents

1. [Bitcoin Client Examples (via_btc_client)](#bitcoin-client-examples-via_btc_client)
2. [ZK Proof Verification Examples (via_verification)](#zk-proof-verification-examples-via_verification)
3. [MuSig2 Examples (via_musig2)](#musig2-examples-via_musig2)
4. [Withdrawal Client Example (via_withdrawal_client)](#withdrawal-client-example-via_withdrawal_client)
5. [Via Test Utils Crate](#via-test-utils-crate)
6. [The via CLI (infrastructure/via)](#the-via-cli-infrastructurevia)
7. [Auxiliary Binaries (core/bin)](#auxiliary-binaries-corebin)
8. [Docker Compose Files](#docker-compose-files)

## Bitcoin Client Examples (via_btc_client)

Location: `core/lib/via_btc_client/examples/`. The crate's `Cargo.toml` declares these `[[example]]` targets: `indexer`, `data_inscription_example`, `inscriber`, `bootstrap`, `verify_batch`, `fee_history`, `deposit_opreturn`, `upgrade_system_contracts`, `upgrade_inscription_parsing`, `propose_new_bridge`, `tx_with_opreturn`, `parse_bootstrap`. Two more files (`deposit.rs`, `fee_rate.rs`) are picked up by cargo's automatic example discovery.

### Fee rate fetching (`fee_rate.rs`)

Fetches the current fee rate through `BitcoinClient::get_fee_rate`, toggling between node RPC estimation and external APIs (mempool.space) via the first CLI argument. This is the real API surface; fee configuration lives in `ViaBtcClientConfig`:

```rust
// core/lib/via_btc_client/examples/fee_rate.rs
use std::{env, str::FromStr, sync::Arc};

use anyhow::Result;
use tracing::info;
use via_btc_client::{
    client::BitcoinClient,
    traits::BitcoinOps,
    types::{BitcoinNetwork, NodeAuth},
};
use zksync_config::configs::via_btc_client::ViaBtcClientConfig;

const RPC_URL: &str = "http://0.0.0.0:18443";
const RPC_USERNAME: &str = "rpcuser";
const RPC_PASSWORD: &str = "rpcpassword";
const NETWORK: BitcoinNetwork = BitcoinNetwork::Regtest;

#[tokio::main]
async fn main() -> Result<()> {
    tracing_subscriber::fmt()
        .with_max_level(tracing::Level::INFO)
        .init();

    let args: Vec<String> = env::args().collect();
    let use_rpc_for_fee_rate = bool::from_str(&args[1].to_string())?;

    let auth = NodeAuth::UserPass(RPC_USERNAME.to_string(), RPC_PASSWORD.to_string());
    let config = ViaBtcClientConfig {
        network: NETWORK.to_string(),
        external_apis: vec!["https://mempool.space/testnet/api/v1/fees/recommended".into()],
        fee_strategies: vec!["fastestFee".into()],
        use_rpc_for_fee_rate: Some(use_rpc_for_fee_rate),
    };
    let client = Arc::new(BitcoinClient::new(&RPC_URL, auth, config)?);
    let fee_rate = client.get_fee_rate(1).await?;
    info!("Fee rate {:?}", fee_rate);

    Ok(())
}
```

```bash
cargo run --example fee_rate -p via_btc_client -- true   # use node RPC estimation
cargo run --example fee_rate -p via_btc_client -- false  # use external API
```

### Fee history (`fee_history.rs`)

Fetches per-block fee history for the last N blocks through the inscriber's client:

```rust
// core/lib/via_btc_client/examples/fee_history.rs (main flow)
let args: Vec<String> = env::args().collect();
let number_blocks = usize::from_str(&args[1].to_string())?;

let auth = NodeAuth::UserPass(RPC_USERNAME.to_string(), RPC_PASSWORD.to_string());
let config = ViaBtcClientConfig {
    network: NETWORK.to_string(),
    external_apis: vec![],
    fee_strategies: vec![],
    use_rpc_for_fee_rate: None,
};
let client = Arc::new(BitcoinClient::new(&RPC_URL, auth, config)?);
let inscriber = Inscriber::new(client, &PK, None)
    .await
    .context("Failed to create Depositor Inscriber")?;

let client = inscriber.get_client().await;

let to_block = client.fetch_block_height().await? as usize;
let from_block = to_block - number_blocks;

let fee_history = client.get_fee_history(from_block, to_block).await?;

info!("Fee history {:?}", fee_history);
```

```bash
cargo run --example fee_history -p via_btc_client -- 10
```

### Inscriber (`inscriber.rs`)

Demonstrates the inscription flow used by the sequencer: create an `Inscriber`, inscribe an `L1BatchDAReference` message, then chain a `ProofDAReference` message that points at the first reveal txid. Node URL and WIF key come from `BITCOIN_NODE_URL` and `BITCOIN_PRV` environment variables:

```rust
// core/lib/via_btc_client/examples/inscriber.rs (main flow)
let rpc_url = std::env::var("BITCOIN_NODE_URL").context("BITCOIN_NODE_URL not set")?;
let prv = std::env::var("BITCOIN_PRV").context("BITCOIN_PRV not set")?;
let auth = NodeAuth::UserPass(RPC_USERNAME.to_string(), RPC_PASSWORD.to_string());
let config = ViaBtcClientConfig {
    network: NETWORK.to_string(),
    external_apis: vec![],
    fee_strategies: vec![],
    use_rpc_for_fee_rate: None,
};
let client = Arc::new(BitcoinClient::new(&rpc_url, auth, config)?);

let mut inscriber_instance = Inscriber::new(client, &prv, None)
    .await
    .context("Failed to create Inscriber")?;

let l1_da_batch_ref = inscribe_types::L1BatchDAReferenceInput {
    l1_batch_hash: zksync_basic_types::H256([0; 32]),
    l1_batch_index: zksync_basic_types::L1BatchNumber(0_u32),
    da_identifier: "da_identifier_celestia".to_string(),
    blob_id: "batch_temp_blob_id".to_string(),
    prev_l1_batch_hash: zksync_basic_types::H256([0; 32]),
};

let inscribe_info = inscriber_instance
    .inscribe(inscribe_types::InscriptionMessage::L1BatchDAReference(
        l1_da_batch_ref,
    ))
    .await
    .context("Failed to inscribe L1BatchDAReference")?;

let l1_da_proof_ref = inscribe_types::ProofDAReferenceInput {
    l1_batch_reveal_txid: inscribe_info.final_reveal_tx.txid,
    da_identifier: "da_identifier_celestia".to_string(),
    blob_id: "proof_temp_blob_id".to_string(),
};

let _da_proof_ref_reveal_txid = inscriber_instance
    .inscribe(inscribe_types::InscriptionMessage::ProofDAReference(
        l1_da_proof_ref,
    ))
    .await
    .context("Failed to inscribe ProofDAReference")?;
```

### Bootstrap (`bootstrap.rs`)

Interactive utility that creates the `SystemBootstrapping` inscription for a new network. It takes positional args (`network`, `rpc_url`, `rpc_username`, `rpc_password`, inscription type, private key), builds an `Inscriber` and sends a `SystemBootstrappingInput` message. Key setup:

```rust
// core/lib/via_btc_client/examples/bootstrap.rs
async fn create_inscriber(
    signer_private_key: &str,
    rpc_url: &str,
    rpc_username: &str,
    rpc_password: &str,
    network: BitcoinNetwork,
) -> anyhow::Result<Inscriber> {
    let auth = NodeAuth::UserPass(rpc_username.to_string(), rpc_password.to_string());
    let config = ViaBtcClientConfig {
        network: network.to_string(),
        external_apis: vec![String::from(
            "https://mempool.space/testnet/api/v1/fees/recommended",
        )],
        fee_strategies: vec![String::from("fastestFee")],
        use_rpc_for_fee_rate: None,
    };
    let client = Arc::new(BitcoinClient::new(rpc_url, auth, config)?);
    Inscriber::new(client, signer_private_key, None)
        .await
        .context("Failed to create Inscriber")
}
```

### Parse bootstrap (`parse_bootstrap.rs`)

Parses a raw hex-encoded bootstrap transaction with the indexer's `MessageParser`, without touching a node:

```rust
// core/lib/via_btc_client/examples/parse_bootstrap.rs (main flow)
// 1. Decode hex string into raw bytes
let raw_tx = hex::decode(hex_tx).expect("invalid hex");

// 2. Deserialize bytes into a bitcoin::Transaction
let tx: Transaction = deserialize(&raw_tx).expect("invalid transaction");

let mut parser = MessageParser::new(BitcoinNetwork::Testnet4);
let messages = parser.parse_system_transaction(&tx, 0, None);

println!("messages: {:?}", messages);
```

### Other via_btc_client examples

- `deposit.rs`: sends a bridge deposit inscription (L1ToL2 message) from a depositor key.
- `deposit_opreturn.rs`: builds and signs a raw deposit transaction carrying the L2 receiver in an OP_RETURN output instead of an inscription.
- `tx_with_opreturn.rs`: constructs and signs a plain transaction with a custom OP_RETURN payload.
- `verify_batch.rs`: fetches batch/proof inscriptions by txid via an `Inscriber` plus `MessageParser` and checks a batch's DA references.
- `upgrade_system_contracts.rs`: creates a protocol upgrade inscription (new protocol version, bootloader/default AA hashes, system contracts) and parses it back.
- `upgrade_inscription_parsing.rs`: parses an existing upgrade inscription from the chain by txid.
- `propose_new_bridge.rs`: creates the bridge update proposal inscription referencing a new bridge address.
- `indexer_init_example.rs`: initializes a `BitcoinInscriptionIndexer` against a regtest node (`regtest::BitcoinRegtest` helper).
- `data_inscription_example.rs`: a large (975-line) self-contained walkthrough of taproot data inscription construction (commit/reveal) at the raw `bitcoin` crate level.

## ZK Proof Verification Examples (via_verification)

Location: `via_verifier/lib/via_verification/examples/`. Contents:

- `zk_v27.rs`: verifies a protocol version 27 proof blob.
- `zk_v28.rs`: verifies a protocol version 28 proof blob.
- `data/`: prefetched proof blobs (e.g. `data_batch_18_0_27_0.bin`, `data_batch_9_0_28_0.bin`).
- `zksync-era-verification-cli/`: vendored zksync-era verification CLI sources kept for reference.

Both examples load a proof blob from disk (with a commented-out helper to fetch it from Celestia), regenerate the public inputs from batch commitments, and run the verifier. The version 28 variant differs only in module path and data file:

```rust
// via_verifier/lib/via_verification/examples/zk_v27.rs
use via_verification::version_27::{
    proof::{ProofTrait, ViaZKProof},
    public_inputs::generate_inputs,
    types::ProveBatches,
    utils::load_verification_key_without_l1_check,
};

// Verify a proof from DA
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // Path to your .bin file
    let path = "via_verifier/lib/via_verification/examples/data/data_batch_18_0_27_0.bin";

    // Open the file in read-only mode
    let mut file = File::open(path)?;

    // Create a buffer to hold the file contents
    let mut buffer = Vec::new();

    // Read the entire file into the buffer
    file.read_to_end(&mut buffer)?;

    let proof_blob = InclusionData { data: buffer };
    let proof_data: ProveBatches = bincode::deserialize(&proof_blob.data).unwrap();

    println!("Block number {:?}", &proof_data.l1_batches[0].header.number);

    let vk_inner = load_verification_key_without_l1_check(
        proof_data.l1_batches[0]
            .header
            .protocol_version
            .unwrap()
            .to_string(),
    )
    .await
    .unwrap();

    let (prev_commitment, curr_commitment) = (
        proof_data.prev_l1_batch.metadata.commitment,
        proof_data.l1_batches[0].metadata.commitment,
    );

    let mut proof = proof_data.proofs[0].scheduler_proof.clone();

    // Put correct inputs
    proof.inputs = generate_inputs(&prev_commitment, &curr_commitment);

    // Verify the proof
    let via_proof = ViaZKProof { proof };

    let is_valid = via_proof.verify(vk_inner).unwrap();
    println!("Result {:?}", is_valid);
    Ok(())
}
```

The optional DA fetch helper in the same file shows the real Celestia client construction:

```rust
// via_verifier/lib/via_verification/examples/zk_v27.rs
async fn _fetch_proof_from_da(path: &str, blob_id: &str) -> anyhow::Result<()> {
    let secrets = ViaDASecrets {
        api_node_url: "https://celestia.stage0.viablockchain.dev:26658".parse::<SensitiveUrl>()?,
        auth_token: /* JWT auth token */,
    };
    let client: Box<dyn DataAvailabilityClient> =
        Box::new(CelestiaClient::new(secrets, 1900000).await?);

    let data = client.get_inclusion_data(blob_id).await?;
    let mut file = File::create_new(path)?;

    file.write(&data.unwrap().data)?;

    Ok(())
}
```

```bash
cargo run --example zk_v27 -p via_verification
cargo run --example zk_v28 -p via_verification
```

## MuSig2 Examples (via_musig2)

Location: `via_verifier/lib/via_musig2/examples/`. Seven example programs; six are declared in `Cargo.toml` (`key_generation_setup`, `withdrawal`, `coordinator`, `musig2_with_script_path`, `compute_musig2`, `transfer_utxos_from_bridge`) and `verify_partial_sig` is picked up by automatic example discovery.

### Key generation and bridge address setup (`key_generation_setup.rs`)

A two-role tool: `contributor` generates (or echoes) a keypair; `coordinator` aggregates the contributed public keys with MuSig2 `KeyAggContext` and derives the P2TR bridge address:

```rust
// via_verifier/lib/via_musig2/examples/key_generation_setup.rs
fn generate_keypair() -> (SecretKey, PublicKey) {
    let mut rng = OsRng;
    let secp = Secp256k1::new();
    let secret_key = SecretKey::new(&mut rng);
    let public_key = PublicKey::from_secret_key(&secp, &secret_key);
    (secret_key, public_key)
}

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

```bash
cargo run --example key_generation_setup -p via_musig2 -- contributor
cargo run --example key_generation_setup -p via_musig2 -- coordinator <pubkey1> <pubkey2> ...
```

### Partial signature verification (`verify_partial_sig.rs`)

A full two-round MuSig2 signing session over a taproot key-spend sighash for three signers, which then calls the crate's real `via_musig2::utils::verify_partial_signature` helper on one participant's partial signature. Key aggregation applies the taproot tweak before signing:

```rust
// via_verifier/lib/via_musig2/examples/verify_partial_sig.rs
use via_musig2::constants::TAPROOT_TWEAK_SCALAR_RANGE_ERR;
use via_musig2::utils::verify_partial_signature;

// ... key aggregation ...
let mut musig_key_agg_cache = KeyAggContext::new(pubkeys.clone())?;

let agg_pubkey = musig_key_agg_cache.aggregated_pubkey::<secp256k1_musig2::PublicKey>();
let (xonly_agg_key, _) = agg_pubkey.x_only_public_key();
let internal_key = bitcoin::XOnlyPublicKey::from_slice(&xonly_agg_key.serialize())?;

// Calculate taproot tweak
let tap_tweak = TapTweakHash::from_key_and_tweak(internal_key, None);
let tweak = tap_tweak.to_scalar();
let tweak_bytes = tweak.to_be_bytes();
let tweak = secp256k1_musig2::Scalar::from_be_bytes(tweak_bytes)
    .with_context(|| TAPROOT_TWEAK_SCALAR_RANGE_ERR)?;

// Apply tweak to the key aggregation context before signing
musig_key_agg_cache = musig_key_agg_cache.with_xonly_tweak(tweak)?;
```

First round (nonce exchange) and second round (partial signatures), followed by the verification call:

```rust
// via_verifier/lib/via_musig2/examples/verify_partial_sig.rs
use musig2::{FirstRound, SecNonceSpices};

let mut first_round_1 = FirstRound::new(
    musig_key_agg_cache.clone(), // Use tweaked context
    thread_rng().gen::<[u8; 32]>(),
    0,
    SecNonceSpices::new()
        .with_seckey(secret_key_1)
        .with_message(&sighash.to_byte_array()),
)?;
// ... first_round_2, first_round_3, receive_nonce() exchanges ...

// Second round: Create partial signatures
let binding = sighash.to_byte_array();
let mut second_round_1 = first_round_1.finalize(secret_key_1, &binding)?;
let second_round_2 = first_round_2.finalize(secret_key_2, &binding)?;
let second_round_3 = first_round_3.finalize(secret_key_3, &binding)?;
// Combine partial signatures
let partial_sig_2: [u8; 32] = second_round_2.our_signature();

verify_partial_signature(
    nonce_2.clone(),
    vec![nonce_1.clone(), nonce_2.clone(), nonce_3.clone()],
    internal_key_2
        .public_key(parity_2)
        .serialize()
        .to_hex_string(bitcoin::hex::Case::Lower),
    pubkeys_str,
    PartialSignature::from_slice(&partial_sig_2.to_vec())?,
    sighash.as_byte_array(),
    None,
)?;
```

The example finishes by aggregating the partial signatures into a Schnorr signature and attaching it as a taproot key-spend witness:

```rust
// via_verifier/lib/via_musig2/examples/verify_partial_sig.rs
second_round_1.receive_signature(1, Scalar::from_slice(&partial_sig_2).unwrap())?;
second_round_1.receive_signature(2, Scalar::from_slice(&partial_sig_3).unwrap())?;

let final_signature: Signature = second_round_1.finalize()?;

// Update the witness stack with the aggregated signature
let signature = bitcoin::taproot::Signature {
    signature: bitcoin::secp256k1::schnorr::Signature::from_slice(
        &final_signature.to_byte_array(),
    )?,
    sighash_type,
};
*sighasher.witness_mut(input_index).unwrap() = Witness::p2tr_key_spend(&signature);
```

### Withdrawal signing (`withdrawal.rs`)

"Demonstrate creating a transaction that spends to and from p2tr outputs with musig2" (file doc comment). Two signers derive the tweaked aggregated key, build the bridge P2TR address, connect to a regtest node (`http://0.0.0.0:18443`, `rpcuser`/`rpcpassword`), and co-sign a withdrawal spend using the crate's `Signer` wrapper (`get_signer(pk_str, pubkeys_str)`). It also uses `via_test_utils::utils::generate_return_data_per_outputs` for OP_RETURN payloads.

### Coordinator (`coordinator.rs`)

The largest example (21 KB): an axum HTTP coordinator that orchestrates a distributed MuSig2 signing session between verifier processes. Real routes:

```rust
// via_verifier/lib/via_musig2/examples/coordinator.rs
let app = Router::new()
    .route("/session/new", post(create_session_handler))
    .route("/session/:id", get(get_session))
    .route("/session/:id/nonce", post(submit_nonce))
    .route("/session/:id/partial", post(submit_partial_signature))
    .route("/session/:id/signature", get(get_final_signature))
    .route("/session/:id/nonces", get(get_nonces))
```

### Compute MuSig2 wallet with governance script path (`compute_musig2.rs`)

A clap-based tool: "Compute a musig 2 wallet with key hash as primary spending and the script path for governance spending hash". It aggregates signer keys as the taproot internal key, builds an M-of-N `OP_CHECKSIG`/`OP_CHECKSIGADD`/`OP_NUMEQUAL` tapscript from governance x-only keys, finalizes the taproot spend info, and exports the result as JSON:

```rust
// via_verifier/lib/via_musig2/examples/compute_musig2.rs
struct Args {
    /// Comma separated signer public keys (compressed hex, 33 bytes)
    #[arg(long)]
    signers: String,

    /// Comma separated governance x-only public keys (hex, N keys)
    #[arg(long)]
    governance_keys: String,

    /// Governance threshold M (e.g. 2 for 2-of-N)
    #[arg(long)]
    threshold: usize,

    /// Bitcoin network (regtest, testnet, mainnet, signet)
    #[arg(long, default_value = "regtest")]
    network: String,

    /// Output JSON file path
    #[arg(long, default_value = "wallet_info.json")]
    output: String,
}
```

```rust
// via_verifier/lib/via_musig2/examples/compute_musig2.rs (script construction)
let mut builder = Builder::new();

for (i, key) in gov_xonly.iter().enumerate() {
    if i == 0 {
        builder = builder.push_x_only_key(key).push_opcode(OP_CHECKSIG);
    } else {
        builder = builder.push_x_only_key(key).push_opcode(OP_CHECKSIGADD);
    }
}

let multisig_script = builder
    .push_int(m as i64)
    .push_opcode(OP_NUMEQUAL)
    .into_script();

let spend_info = bitcoin::taproot::TaprootBuilder::new()
    .add_leaf(0, ScriptBuf::from(multisig_script.clone()))?
    .finalize(&secp, internal_key)
    .unwrap();
```

The exported `ExportData` JSON contains: `public_keys`, `aggregated_internal_key`, `governance_script_hex`, `taproot_output_key`, `merkle_root`, `taproot_address`, `control_block`, `threshold`, `total_governance_keys`.

```bash
cargo run --example compute_musig2 -p via_musig2 -- \
  --signers <pk1,pk2> --governance-keys <gk1,gk2,gk3> --threshold 2 \
  --network regtest --output wallet_info.json
```

### Script-path spending (`musig2_with_script_path.rs`)

End-to-end demonstration of spending a taproot output through the governance script path (tapscript multisig) rather than the MuSig2 key path, against a regtest node.

### Transfer UTXOs from bridge (`transfer_utxos_from_bridge.rs`)

Operational utility example: drains/migrates bridge UTXOs to a new address by building, MuSig2-signing, and serializing a sweep transaction (uses `consensus::encode::serialize_hex` for broadcast-ready output).

### Tip: handling empty withdrawal transactions

When all withdrawals fall below their fair fee share after filtering, the builder produces an unsigned transaction that is effectively empty. `TransactionBuilder::build_transaction_with_op_return()` (`via_verifier/lib/via_musig2/src/transaction_builder.rs:53`) can return such a transaction, and `UnsignedBridgeTx::is_empty()` (`via_verifier/lib/via_verifier_types/src/transaction.rs:25`) is available for downstream code to detect it and skip broadcast or session progression.

## Withdrawal Client Example (via_withdrawal_client)

Location: `via_verifier/lib/via_withdrawal_client/examples/withdraw.rs`. Fetches the withdrawals contained in a given L1 batch by looking up the batch's DA blob in Postgres, downloading it from Celestia, and parsing withdrawal messages:

```rust
// via_verifier/lib/via_withdrawal_client/examples/withdraw.rs (main flow)
const DEFAULT_DATABASE_URL: &str = "postgres://postgres:notsecurepassword@0.0.0.0:5432/via";
const DEFAULT_CELESTIA: &str = "http://0.0.0.0:26658";

let home = env::var("VIA_HOME").context("VIA HOME not set")?;
let _ = dotenv::from_path(home.clone() + "/etc/env/target/via.env");

let celestia_auth_token = env::var("VIA_CELESTIA_CLIENT_AUTH_TOKEN")?;

let args: Vec<String> = env::args().collect();
let block_number = args[1].parse::<u32>().unwrap();

// Connect to db
let url = SensitiveUrl::from_str(DEFAULT_DATABASE_URL).unwrap();
let connection_pool = ConnectionPool::<Core>::builder(url, 100)
    .build()
    .await
    .unwrap();
let l1_batch_number = L1BatchNumber::from(block_number);
let mut storage = connection_pool.connection().await.unwrap();

let header_res = storage
    .via_data_availability_dal()
    .get_da_blob(l1_batch_number)
    .await
    .unwrap();

// ... build CelestiaClient from ViaDASecrets ...

let withdrawal_client =
    WithdrawalClient::new(da_client, bitcoin::Network::Regtest, web3_client);

let withdrawals = withdrawal_client
    .get_withdrawals(header.blob_id.as_str(), L1BatchNumber(block_number))
    .await?;

info!("Withdrawals {:?}", withdrawals);
```

```bash
cargo run --example withdraw -p via_withdrawal_client -- <l1_batch_number>
```

## Via Test Utils Crate

Location: `core/lib/via_test_utils` (`src/lib.rs`, `src/utils.rs`). A helper crate with reusable building blocks for integration and end-to-end tests. Real public functions in `src/utils.rs`:

- `test_bitcoin_client() -> BitcoinClient` and `test_sequencer_wallet() -> (BitcoinAddress, String)`
- `test_sequencer_inscriber() -> anyhow::Result<Inscriber>` (async)
- `test_verifier_add_1() / test_verifier_add_2() / test_verifier_add_3() -> BitcoinAddress` and `test_wallets() -> SystemWallets`
- `bootstrap_state_mock() -> BootstrapState`: bootstrap state mock including the sequencer vote set
- `create_update_sequencer_inscription(address)`, `create_update_governance_inscription(address)`, `create_update_bridge_inscription(...)` returning `FullInscriptionMessage`s
- `create_chained_inscriptions(...)` (async): emits a sequence of `L1BatchDAReference` followed by `ProofDAReference` inscriptions for N chained batches
- `test_create_indexer() -> BitcoinInscriptionIndexer`
- `random_bitcoin_wallet() -> (PrivateKey, BitcoinAddress)`
- `generate_return_data_per_outputs(count) -> Vec<Vec<u8>>` and `generate_dummy_outputs(count, value) -> Vec<TxOut>` (used by the `withdrawal` MuSig2 example)

Typical usage in watcher tests: create N chained batches with `create_chained_inscriptions`, feed the messages in order to the watcher's message processor, then assert DB side effects (batch existence, canonical head, first not-finalized/not-verified queries).

## The via CLI (infrastructure/via)

The `via` workflow tool (`infrastructure/via/src/index.ts`) requires `$VIA_HOME` to point at the repo root and registers these commands: `server`, `up`, `down`, `db`, `init`, `run`, `docker`, `config`, `clean`, `env`, `transactions`, `bootstrap`, `verifier`, `celestia`, `btc-explorer`, `token`, `contract`, `test`, `indexer`, `debug`, `multisig`, `setup_en`, `en`, `completion`, plus the pass-through `via f <command...>`.

### Token bridging: `via token`

Defined in `infrastructure/via/src/token.ts` (`new Command('token').description('Bridge BTC L2<>L1')`). Subcommands: `deposit`, `deposit-with-op-return`, `withdraw`.

```bash
# Deposit BTC to L2 (bridge address is required)
via token deposit \
  --amount 10 \
  --receiver-l2-address 0xYourL2Address \
  --bridge-address bcrt1p... \
  [--sender-private-key <WIF>] \
  [--network regtest|testnet|bitcoin] \
  [--l1-rpc-url <url>] [--l2-rpc-url <url>] \
  [--rpc-username rpcuser] [--rpc-password rpcpassword] \
  [--external-api-fee https://mempool.space/testnet/api/v1/fees/recommended]

# Deposit carrying the L2 receiver in an OP_RETURN output
via token deposit-with-op-return --amount 1 --receiver-l2-address 0x... --bridge-address bcrt1p...

# Withdraw BTC to L1
via token withdraw \
  --amount 1.0 \
  --receiver-l1-address bcrt1... \
  [--user-private-key 0x...] \
  [--rpc-url http://0.0.0.0:3050]
```

`deposit()`, `getWallet(rpcUrl, userL2PrivateKey)`, and `withdraw(...)` are exported from `token.ts` for programmatic use.

### Debug commands: `via debug` (batch deposits and withdrawals)

Defined in `infrastructure/via/src/debug.ts` (`new Command('debug').description('Debug cmds used for testing')`). Wallet helpers (BIP39/BIP32 P2WPKH) come from `generateBitcoinWallet()` in `infrastructure/via/src/helpers.ts`; defaults come from `infrastructure/via/src/constants.ts`.

```bash
# Batch deposits to L2 (splits the runs across --count deposits)
via debug deposit-many \
  --amount 0.10 \
  --receiver-l2-address 0xYourL2Address \
  --bridge-address bcrt1p... \
  [--sender-private-key <WIF>] \
  [--network <network>] \
  [--l1-rpc-url <url>] [--l2-rpc-url <url>] \
  [--rpc-username rpcuser] [--rpc-password rpcpassword] \
  [--count 100]

# Batch withdrawals to freshly generated random L1 addresses
via debug withdraw-many \
  --amount 1.0 \
  [--count 10] \
  [--user-private-key 0x...] \
  [--rpc-url http://0.0.0.0:3050] \
  [--network regtest]
```

`withdraw-many` reuses a single L2 wallet and passes sequential nonces to `withdraw(..., nonce)` to avoid clashes, logging the generated receiver per withdrawal.

### Transactions generator: `via transactions`

Defined in `infrastructure/via/src/transactions.ts` ("Generate random transactions on the Bitcoin regtest network."). By default the bitcoin-cli calls run inside the compose container; `--skip-container` runs them directly on the host:

```bash
via transactions \
  [--rpc-connect bitcoind] \
  [--rpc-username rpcuser] \
  [--rpc-password rpcpassword] \
  [--rpc-wallet <wallet>] \
  [--address <destination-address>] \
  [--skip-container]
```

### Bootstrap: `via bootstrap`

Defined in `infrastructure/via/src/bootstrap.ts`. Subcommands: `system-bootstrapping` (run the network bootstrap flow) and `update-bootstrap-tx`, which calls the exported `updateBootstrapTxidsEnv(network)` to write bootstrap txids into the env files (tolerating empty-string values).

### Multisig governance tooling: `via multisig`

Defined in `infrastructure/via/src/multisig.ts` (`new Command('multisig').description('Multisig helper')`). It constructs, signs, finalizes, and broadcasts OP_RETURN governance execution transactions. Real subcommands and options:

```bash
# Create a bitcoin wallet
via multisig create-wallet [--network <network>]

# Compute multisig address
via multisig compute-multisig \
  --pubkeys <hexPubkey1,hexPubkey2,hexPubkey3> \
  --minimumSigners 2 \
  [--outDir ./upgrade_tx_exec.json] [--network <network>]

# Create an unsigned multisig upgrade transaction
via multisig create-upgrade-tx \
  --inputTxId <tx_id> --inputVout <vout> --inputAmount <sats> \
  --upgradeProposalTxId <proposal_txid> --fee 500 \
  [--outDir ./upgrade_tx_exec.json] [--network <network>]

# Generate an unsigned multisig transaction for updating the sequencer
via multisig create-update-sequencer \
  --inputTxId <tx_id> --inputVout <vout> --inputAmount <sats> \
  --fee 500 --sequencerAddress <bcrt1...> \
  [--outDir ./upgrade_tx_exec.json] [--network <network>]

# Generate an unsigned multisig transaction for updating the bridge
via multisig create-update-bridge \
  --inputTxId <tx_id> --inputVout <vout> --inputAmount <sats> \
  --fee 500 --proposalTxid <proposal_txid> \
  [--outDir ./upgrade_tx_exec.json] [--network <network>]

# Generate an unsigned multisig transaction for updating the governance
via multisig create-update-gov \
  --inputTxId <tx_id> --inputVout <vout> --inputAmount <sats> \
  --fee 500 --governanceAddress <bcrt1...> \
  [--outDir ./upgrade_tx_exec.json] [--network <network>]

# Sign, finalize, broadcast
via multisig sign-tx --privateKey <wif> [--outDir ./upgrade_tx_exec.json]
via multisig finalize-tx [--outDir ./upgrade_tx_exec.json]
via multisig broadcast-tx --rpcUrl http://0.0.0.0:18443 --rpcUser rpcuser --rpcPass rpcpassword \
  [--outDir ./upgrade_tx_exec.json]
```

OP_RETURN prefix constants in `infrastructure/via/src/constants.ts`:

```typescript
// infrastructure/via/src/constants.ts
export const OP_RETURN_WITHDRAW_PREFIX = 'VIA_PROTOCOL:WITHDRAWAL';
export const OP_RETURN_UPGRADE_PROTOCOL_PREFIX = 'VIA_PROTOCOL:UPGRADE';
export const OP_RETURN_UPDATE_SEQUENCER_PREFIX = 'VIA_PROTOCOL:SEQ';
export const OP_RETURN_UPDATE_BRIDGE_PREFIX = 'VIA_PROTOCOL:BRI';
export const OP_RETURN_UPDATE_GOVERNANCE_PREFIX = 'VIA_PROTOCOL:GOV';
```

Related guides in the repo: `docs/via_guides/upgrade.md` and `docs/via_guides/musig2.md`.

### Node runners and infrastructure helpers

- `via verifier [--network <network>]`: starts the verifier node (`cargo run --bin via_verifier --release`).
- `via indexer [--network <network>]`: starts the L1 indexer node (`cargo run --bin via_indexer_bin`).
- `via celestia --backend <backend>`: manages the local Celestia DA container.
- `via btc-explorer`: runs a Bitcoin explorer (uses `docker-compose-via-btc-explorer.yml`).

## Auxiliary Binaries (core/bin)

Directories under `core/bin/` that build standalone binaries:

- `via_server`: the Via sequencer node.
- `via_external_node`: the Via external (read replica) node.
- `via_block_reverter`: Via-specific block revert tooling.
- `zksync_server`, `external_node`, `block_reverter`: inherited zksync-era counterparts.
- `genesis_generator`: regenerates the genesis config.
- `custom_genesis_export`: exports a custom genesis state.
- `snapshots_creator`: creates node state snapshots.
- `merkle_tree_consistency_checker`: verifies Merkle tree integrity.
- `contract-verifier`, `verified_sources_fetcher`: contract verification tooling.
- `selector_generator`, `system-constants-generator`: codegen utilities.
- `zksync_tee_prover`: TEE prover binary.

## Docker Compose Files

Repository root compose files relevant to local development:

- `docker-compose.yml`, `docker-compose-via.yml`: base local stacks (bitcoind, postgres, celestia and friends).
- `docker-compose-via-btc-explorer.yml`: Bitcoin explorer stack used by `via btc-explorer`.
- `docker-compose-unit-tests.yml`, `docker-compose-cpu-runner.yml`, `docker-compose-gpu-runner.yml`, `docker-compose-gpu-runner-cuda-12-0.yml`, `docker-compose-runner-nightly.yml`: CI/test runner stacks.

Docker image build contexts live under `docker/` (including `via-server`, `via-verifier`, `via-external-node`, `via-l1-indexer`, `via-environment`).

## Conclusion

The real example surface of via-core consists of:

1. **Bitcoin client examples** (`via_btc_client`): inscription creation and parsing, deposits (inscription and OP_RETURN variants), fee rate/history, bootstrap, batch verification, and protocol upgrade flows.
2. **ZK proof verification examples** (`via_verification`): `zk_v27` and `zk_v28`, which verify real proof blobs from the `examples/data/` directory or from Celestia.
3. **MuSig2 examples** (`via_musig2`): key generation and bridge address setup, two-round signing with partial signature verification, an HTTP coordinator, governance script-path wallets, and bridge UTXO transfer.
4. **Withdrawal client example** (`via_withdrawal_client`): extracting withdrawals for a batch from the DA layer.
5. **Developer tooling**: the `via` CLI (token bridging, debug batch operations, transaction generation, bootstrap, multisig governance) and the auxiliary binaries under `core/bin/`.
