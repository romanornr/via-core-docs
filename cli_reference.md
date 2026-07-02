# Via L2 CLI Reference Guide

## Overview

Via L2 exposes two real command-line surfaces, both living in the `via-core` repository:

1. **The `via` developer CLI**: a TypeScript tool built on `commander`, located at `infrastructure/via/` and invoked through the `bin/via` wrapper script. It is the day-to-day workflow tool for developers (starting servers, managing databases, bridging test BTC, running tests).
2. **Rust binaries**: `via_server` (the sequencer), `via_verifier` (the verifier/coordinator node), and `via_indexer_bin` (the L1 indexer). Only `via_server` and `via_verifier` define `clap` CLIs. `via_indexer_bin` accepts **no command-line arguments at all**; it is configured entirely through environment variables.

The legacy ZKsync `zk` tool also still exists at `infrastructure/zk/` (with its own `bin/zk` wrapper) and is still used for a few tasks such as `zk fmt`.

> **Note:** There are no standalone `via-server`, `via-indexer`, `via-restart-indexer`, `via-db-*`, `via-docker-*`, `via-test`, `via-loadtest`, `via-dev-setup`, `via-generate-keys`, `via-config-validate`, `via-config-generate`, or `via-indexer-status` executables. Everything below is quoted from source.

## Table of Contents

1. [Installation and Setup](#installation-and-setup)
2. [The `via` Developer CLI](#the-via-developer-cli)
3. [Rust Binaries](#rust-binaries)
4. [Server Components](#server-components)
5. [Runnable Examples](#runnable-examples)
6. [Environment Variables](#environment-variables)
7. [Command Workflows](#command-workflows)
8. [Troubleshooting](#troubleshooting)

## Installation and Setup

### Prerequisites

- Rust toolchain (the workspace pins its version in `rust-toolchain`)
- Node.js and Yarn 1.22 (the `bin/via` wrapper warns if Yarn is not at 1.22)
- PostgreSQL and Docker (development containers are started with `via up`)
- Bitcoin Core (run as the `bitcoind` container by `via up` in development)

### Environment Setup

`$VIA_HOME` must point to the root of the `via-core` checkout. Both the wrapper script and the CLI entry point enforce this:

```typescript
// infrastructure/via/src/index.ts
async function main() {
    const cwd = process.cwd();
    const VIA_HOME = process.env.VIA_HOME;

    if (!VIA_HOME) {
        throw new Error('Please set $VIA_HOME to the root of Via repo!');
    } else {
        process.chdir(VIA_HOME);
    }

    env.load();

    program.version('0.1.0').name('via').description('via workflow tools');
```

```bash
# Set Via home directory and put the wrapper scripts on PATH
export VIA_HOME=/path/to/via-core
export PATH=$VIA_HOME/bin:$PATH

# Optional: install shell completion for the via CLI
via completion install
```

### Building the Rust Binaries

The binary names come from the crate manifests: `core/bin/via_server/Cargo.toml` (`name = "via_server"`), `via_verifier/bin/verifier_server/Cargo.toml` (`name = "via_verifier"`), and `via_indexer/bin/indexer/Cargo.toml` (`name = "via_indexer_bin"`).

```bash
# Build specific binaries
cargo build --release --bin via_server
cargo build --release --bin via_verifier
cargo build --release --bin via_indexer_bin
```

In normal development you rarely run `cargo` yourself; the `via` CLI wraps it (for example `via server` runs `cargo run --bin via_server --release`).

## The `via` Developer CLI

The full set of registered top-level commands, verbatim from the entry point:

```typescript
// infrastructure/via/src/index.ts
const COMMANDS = [
    server,
    up,
    down,
    db,
    initCommand,
    run,
    docker,
    config,
    clean,
    env.command,
    transactions,
    bootstrap,
    verifier,
    celestia,
    btc_explorer,
    token,
    contract,
    test,
    indexer,
    debug,
    multisig,
    enSetup,
    en,
    completion(program as Command)
];
```

There is also a special-cased `f` passthrough command that runs an arbitrary command from your current directory instead of `$VIA_HOME`:

```typescript
// infrastructure/via/src/index.ts
    // f command is special-cased because it is necessary
    // for it to run from $PWD and not from $VIA_HOME
    program
        .command('f <command...>')
        .allowUnknownOption()
        .action((command: string[]) => {
            process.chdir(cwd);
            const result = spawnSync(command[0], command.slice(1), { stdio: 'inherit' });
            if (result.error) {
                throw result.error;
            }
            process.exitCode = result.status || undefined;
        });
```

### via server

```typescript
// infrastructure/via/src/server.ts
export const serverCommand = new Command('server')
    .description('start via server')
    .option('--genesis', 'generate genesis data via server')
    .option('--uring', 'enables uring support for RocksDB')
    .option('--components <components>', 'comma-separated list of components to run')
    .option('--chain-name <chain-name>', 'environment name')
    .action(async (cmd: Command) => {
        cmd.chainName ? env.reload(cmd.chainName) : env.load();
        if (cmd.genesis) {
            await genesisFromSources();
        } else {
            await server(cmd.rebuildTree, cmd.uring, cmd.components, cmd.useNodeFramework);
        }
    });
```

Under the hood it compiles and runs the Rust sequencer:

```typescript
// infrastructure/via/src/server.ts
export async function server(rebuildTree: boolean, uring: boolean, components?: string, useNodeFramework?: boolean) {
    let options = '';
    if (uring) options += '--features=rocksdb/io-uring';
    if (rebuildTree || components || useNodeFramework) options += ' --';
    if (components) options += ` --components=${components}`;
    await utils.spawn(`cargo run --bin via_server --release ${options}`);
}
```

With `--genesis`, it runs `cargo run --bin via_server --release -- --genesis` (see `genesisFromSources()` in the same file).

```bash
# Start the sequencer with default components
via server

# Start with an explicit component list
via server --components api,btc,tree,tree_api,state_keeper

# Generate genesis data
via server --genesis
```

### via external-node

```typescript
// infrastructure/via/src/server.ts
export const enCommand = new Command('external-node')
    .description('start via external node')
    .option('--reinit', 'reset postgres and rocksdb before starting')
    .action(async (cmd: Command) => {
        await externalNode(cmd.reinit, cmd.args);
    });
```

It expects `VIA_ENV=via_ext_node` and runs `cargo run --bin via_external_node --release`. There is also a companion `via setup-external-node` command ("prepare local setup for running external-node on mainnet/testnet", `infrastructure/via/src/setup_en.ts`).

### via verifier

```typescript
// infrastructure/via/src/verifier.ts
export const verifierCommand = new Command('verifier')
    .description('start via verifier node')
    .option('--network <network>', 'network', DEFAULT_NETWORK)
    .action(async (cmd: Command) => {
        cmd.chainName ? env.reload(cmd.chainName) : env.load();
        env.get(true);
        await verifier(cmd.network);
    });
```

`DEFAULT_NETWORK` is `'regtest'`. The action updates the bootstrap txid env vars for the chosen network and then runs `cargo run --bin via_verifier --release`.

### via indexer

```typescript
// infrastructure/via/src/indexer.ts
export const indexerCommand = new Command('indexer')
    .description('start via indexer node')
    .option('--network <network>', 'network', 'regtest')
    .action(async (cmd: Command) => {
        cmd.chainName ? env.reload(cmd.chainName) : env.load();
        env.get(true);
        await indexer(cmd.network);
    });
```

The action runs `cargo run --bin via_indexer_bin`. `--network` is the **only** flag; there are no `--start-block`, `--batch-size`, or `--poll-interval` options, and there is no TOML config file. The indexer binary reads all of its configuration from environment variables (see [via_indexer_bin](#via_indexer_bin)).

```bash
# Start the L1 indexer on regtest
via indexer

# Start the L1 indexer against another network's bootstrap txids
via indexer --network testnet
```

### via init

```typescript
// infrastructure/via/src/init.ts
export const initCommand = new Command('init')
    .option('--skip-submodules-checkout')
    .option('--skip-env-setup')
    .option('--skip-test-token-deployment') // ?
    .option('--base-token-name <base-token-name>', 'base token name') // ?
    // .option('--validium-mode', 'deploy contracts in Validium mode')
    .option('--run-observability', 'run observability suite')
    .option('--skip-submodules-checkout')
    .option('--profile <profile>', '') // docker compose profile [reorg]
    .option('--mode [type]', 'init mode', Mode.SEQUENCER)
    .option(
        '--local-legacy-bridge-testing',
        'used to test LegacyBridge compatibily. The chain will have the same id as the era chain id, while eraChainId in L2SharedBridge will be 0'
    )
    .option('--should-check-postgres', 'Whether to perform cargo sqlx prepare --check during database setup', true)
    .description('Deploys the shared bridge and registers a hyperchain locally, as quickly as possible.')
    .action(init);
```

The `--mode` values come from the `Mode` enum:

```typescript
// infrastructure/via/src/types.ts
export enum Mode {
    SEQUENCER = 'sequencer',
    VERIFIER = 'verifier',
    COORDINATOR = 'coordinator',
    INDEXER = 'indexer'
}
```

```bash
# Initialize a local sequencer environment (default mode)
via init

# Initialize for verifier / coordinator / indexer development
via init --mode verifier
via init --mode coordinator
via init --mode indexer
```

### via db

All subcommands operate on databases selected with `-p, --prover` and `-c, --core` flags (plus verifier and indexer databases handled internally through `DATABASE_VERIFIER_URL` and `DATABASE_INDEXER_URL`):

```typescript
// infrastructure/via/src/database.ts
export const command = new Command('db').description('database management');

command.command('drop').description('drop the database').option('-p, --prover').option('-c, --core').action(drop);
command.command('migrate').description('run migrations').option('-p, --prover').option('-c, --core').action(migrate);
command
    .command('new-migration')
    .description('generate a new migration for a specific database')
    .option('-p, --prover <name>')
    .option('-c, --core <name>')
    .action((opts: DbGenerateMigrationOpts) => { /* ... */ });
command
    .command('setup')
    .description('initialize the database and perform migrations')
    .option('-p, --prover')
    .option('-c, --core')
    .action(setup);
command
    .command('wait')
    .description('wait for database to get ready for interaction')
    .option('-p, --prover')
    .option('-c, --core')
    .action(wait);
command
    .command('reset')
    .description('reinitialize the database')
    .option('-p, --prover')
    .option('-c, --core')
    .action(reset);
command
    .command('reset-test')
    .description('reinitialize the database for test')
    .option('-p, --prover')
    .option('-c, --core')
    .action(resetTest);
command
    .command('check-sqlx-data')
    .description('check sqlx-data.json is up to date')
    .option('-p, --prover')
    .option('-c, --core')
    .action(async (cmd) => {
        await prepareSqlxData(cmd, true);
    });
command
    .command('prepare')
    .description('check sqlx-data.json is up to date')
    .option('-p, --prover')
    .option('-c, --core')
    .action(async (cmd) => {
        await prepareSqlxData(cmd, false);
    });
```

```bash
# Initialize the core database and run migrations
via db setup --core

# Run pending migrations
via db migrate --core

# Reinitialize the database
via db reset --core

# Generate a new core migration
via db new-migration --core name_of_migration
```

### via env

```typescript
// infrastructure/via/src/env.ts
export const command = new Command('env')
    .description('get or set via environment')
    .option('--soft', 'Skip reloading the environment')
    .action((cmd: Command) => {
        // If user typed `cli env dev --soft`, then:
        // cmd.args[0] = 'dev'
        // cmd.soft = true
        const environment = cmd.args[0];
        const { soft } = cmd.opts();

        environment ? set(environment, /*print=*/ true, soft) : get(/*print=*/ true);
    });
```

```bash
# Print the current environment
via env

# Switch to the dev environment
via env dev
```

### via up / via down

```typescript
// infrastructure/via/src/up.ts
export const command = new Command('up')
    .description('start development containers')
    .option('--docker-file <dockerFile>', 'path to a custom docker file', VIA_DOCKER_COMPOSE)
    .option('--run-observability', 'whether to run observability stack')
    .action(async (cmd) => {
        await up(cmd.dockerFile);
    });
```

```typescript
// infrastructure/via/src/down.ts
export const command = new Command('down').description('stop development containers').action(down);
```

```bash
# Set up bitcoind, bitcoin-cli, postgres and celestia-node containers
via up

# Set up the Bitcoin explorer containers
via up --docker-file $VIA_HOME/docker-compose-via-btc-explorer.yml

# Shut all development containers down
via down
```

### via clean

```typescript
// infrastructure/via/src/clean.ts
export const command = new Command('clean')
    .option('--config [environment]')
    .option('--database')
    .option('--contracts')
    .option('--artifacts')
    .option('--profile <profile>', 'The docker compose profile', '')
    .option('--all')
    .description('removes generated files')
```

If no selector flag is given, `--all` is assumed. `--all` also brings the containers down and removes `volumes`, the local `db` directory, and `core/lib/via_btc_client/depositor_inscriber_context.json`.

### via docker

```typescript
// infrastructure/via/src/docker.ts
export const command = new Command('docker').description('docker management');

command
    .command('build <image>')
    .option('--custom-tag <value>', 'Custom tag for image')
    .option('--platform <platform>', 'Docker platform')
    .description('build docker image')
    .action(build);
command
    .command('push <image>')
    .option('--custom-tag <value>', 'Custom tag for image')
    .option('--platform <platform>', 'Docker platform')
    .action(push);
command.command('pull').description('pull all containers').action(pull);
command.command('restart <container>').description('restart container in docker-compose.yml').action(restart);
```

### via test

```typescript
// infrastructure/via/src/test/test.ts
export const command = new Command('test').description('run test suites');

command
    .command('rust [command...]')
    .allowUnknownOption()
    .description(
        "run unit-tests. the default is running all tests in all rust bins and libs. accepts optional arbitrary cargo test flags. use `-p 'via_*'` to run all unit tests for VIA crates."
    )
    .action(async (args: string[]) => {
        await rust(args);
    });
```

```bash
# Run all Rust unit tests
via test rust

# Run a single test
via test rust --package zksync_core --lib eth_sender::tests::resend_each_block

# Run all Via crate tests
via test rust -p 'via_*'
```

### via token

```typescript
// infrastructure/via/src/token.ts
export const command = new Command('token').description('Bridge BTC L2<>L1');
command
    .command('deposit')
    .description('deposit BTC to l2')
    .requiredOption('--amount <amount>', 'amount of BTC to deposit', parseFloat)
    .requiredOption('--receiver-l2-address <receiverL2Address>', 'receiver l2 address')
    .requiredOption('--bridge-address <bridgeAddress>', 'The bridge address')
    .option('--sender-private-key <senderPrivateKey>', 'sender private key', DEFAULT_DEPOSITOR_PRIVATE_KEY)
    .option('--network <network>', 'network', DEFAULT_NETWORK)
    .option('--l1-rpc-url <l1RcpUrl>', 'RPC URL', DEFAULT_L1_RPC_URL)
    .option('--l2-rpc-url <l2RcpUrl>', 'RPC URL', DEFAULT_L2_RPC_URL)
    .option('--rpc-username <rcpUsername>', 'RPC username', DEFAULT_RPC_USERNAME)
    .option('--rpc-password <rpcPassword>', 'RPC password', DEFAULT_RPC_PASSWORD)
    .option(
        '--external-api-fee <externalApiFee>',
        'External API to use as fallback fee',
        'https://mempool.space/testnet/api/v1/fees/recommended'
    )
```

```typescript
// infrastructure/via/src/token.ts
command
    .command('deposit-with-op-return')
    .description('deposit BTC to l2 with op-return')
    .requiredOption('--amount <amount>', 'amount of BTC to deposit', parseFloat)
    .requiredOption('--receiver-l2-address <receiverL2Address>', 'receiver l2 address')
    .requiredOption('--bridge-address <bridgeAddress>', 'The bridge address')
    .option('--sender-private-key <senderPrivateKey>', 'sender private key', DEFAULT_DEPOSITOR_PRIVATE_KEY)
    .option('--network <network>', 'network', DEFAULT_NETWORK)
    .option('--l1-rpc-url <l1RcpUrl>', 'RPC URL', DEFAULT_L1_RPC_URL)
    .option('--l2-rpc-url <l2RcpUrl>', 'RPC URL', DEFAULT_L2_RPC_URL)
    .option('--rpc-username <rcpUsername>', 'RPC username', DEFAULT_RPC_USERNAME)
    .option('--rpc-password <rpcPassword>', 'RPC password', DEFAULT_RPC_PASSWORD)
```

```typescript
// infrastructure/via/src/token.ts
command
    .command('withdraw')
    .description('withdraw BTC to l1')
    .requiredOption('--amount <amount>', 'amount of BTC to withdraw')
    .requiredOption('--receiver-l1-address <receiverL1Address>', 'receiver l1 address')
    .option('--user-private-key <userPrivateKey>', 'user private key', DEFAULT_L2_PRIVATE_KEY)
    .option('--rpc-url <rcpUrl>', 'RPC URL', DEFAULT_L2_RPC_URL)
    .action((cmd: Command) => withdraw(cmd.amount, cmd.receiverL1Address, cmd.userPrivateKey, cmd.rpcUrl));
```

### via transactions

```typescript
// infrastructure/via/src/transactions.ts
export const command = new Command('transactions')
    .description('Generate random transactions on the Bitcoin regtest network.')
    .option('--rpc-connect <rpcConnect>', 'RPC connect', RPC_CONNECT)
    .option('--rpc-username <rpcUsername>', 'RPC username', RPC_USER)
    .option('--rpc-password <rpcPassword>', 'RPC password', RPC_PASSWORD)
    .option('--rpc-wallet <rpcPassword>', 'RPC wallet', RPC_WALLET)
    .option('--address <address>', 'Destination address', DESTINATION_ADDRESS)
    .option('--skip-container', 'Skip execution inside a Docker container and run directly on the host')
    .action(async (options) => {
        await generateRandomTransactions(options);
    });
```

### via bootstrap

```typescript
// infrastructure/via/src/bootstrap.ts
export const command = new Command('bootstrap');

command
    .command('system-bootstrapping')
    .description('Create a system bootstrapping inscription')
    .option('--network <network>', 'network', DEFAULT_NETWORK)
    .option('--rpc-url <rpcUrl>', 'RPC URL', DEFAULT_RPC_URL)
    .option('--rpc-username <rpcUsername>', 'RPC username', DEFAULT_RPC_USERNAME)
    .option('--rpc-password <rpcPassword>', 'RPC password', DEFAULT_RPC_PASSWORD)
    .requiredOption('--start-block <startBlock>', 'Start block')
    .requiredOption('--private-key <privateKey>', 'The inscriber private key')
    .requiredOption('--bridge-wallet-path <bridgeWalletPath>', 'The musig2 bridge address file path')
    .requiredOption('--verifiers-pub-keys <verifiersPubKeys>', 'verifiers public keys')
    .requiredOption('--governance-address <governanceAddress>', 'The governance address')
    .requiredOption('--sequencer-address <sequencerAddress>', 'The sequencer address')
```

```typescript
// infrastructure/via/src/bootstrap.ts
command
    .command('update-bootstrap-tx')
    .description('Update the bootstrap envs')
    .option('--network <network>', 'network', DEFAULT_NETWORK)
    .action((cmd: Command) => updateBootstrapTxidsEnv(cmd.network));
```

`system-bootstrapping` runs the `bootstrap` cargo example from `via_btc_client` (`cargo run --example bootstrap ...`).

### Other `via` commands

| Command | Description (from source) | Subcommands / notes |
|---------|---------------------------|---------------------|
| `via config` | `config management` | `load [environment]`, `compile [environment]` |
| `via contract` | `contract management` | `build`, `deploy-l2` |
| `via run` | `Run miscellaneous applications` | `yarn`, `cat-logs [exit_code]`, `snapshots-creator`, `loadtest` |
| `via debug` | `Debug cmds used for testing` | `deposit` ("deposit BTC to l2"), a "TEST: withdraw many BTC to l1" command |
| `via multisig` | `Multisig helper` | create a bitcoin wallet, compute multisig address, create/sign/finalize/broadcast unsigned multisig upgrade transactions, update sequencer/bridge/governance |
| `via celestia` | `VIA celestia` | `--backend <backend>` (required, defaults to Celestia) |
| `via btc-explorer` | `Running a Bitcoin explorer` | no options |
| `via setup-external-node` | `prepare local setup for running external-node on mainnet/testnet` | see `setup_en.ts` |
| `via completion` | shell completion management | `completion install` |
| `via f <command...>` | passthrough runner from `$PWD` | see [index.ts quote above](#the-via-developer-cli) |

## Rust Binaries

### via_server

The full `clap` CLI, verbatim:

```rust
// core/bin/via_server/src/main.rs
#[derive(Debug, Parser)]
#[command(author = "Via Protocol", version, about = "Via sequencer node", long_about = None)]
struct Cli {
    /// Generate genesis block for the first contract deployment using temporary DB.
    #[arg(long)]
    genesis: bool,

    /// Comma-separated list of components to launch.
    #[arg(
        long,
        default_value = "api,btc,tree,tree_api,state_keeper,housekeeper,proof_data_handler,commitment_generator,celestia,da_dispatcher,vm_runner_protective_reads,vm_runner_bwip"
    )]
    components: ComponentsToRun,

    /// Path to the YAML config. If set, it will be used instead of env vars.
    #[arg(long)]
    config_path: Option<std::path::PathBuf>,

    /// Path to the YAML with secrets. If set, it will be used instead of env vars.
    #[arg(long)]
    secrets_path: Option<std::path::PathBuf>,

    /// Path to the yaml with contracts. If set, it will be used instead of env vars.
    #[arg(long)]
    contracts_config_path: Option<std::path::PathBuf>,

    /// Path to the wallets config. If set, it will be used instead of env vars.
    #[arg(long)]
    wallets_path: Option<std::path::PathBuf>,

    /// Path to the YAML with genesis configuration. If set, it will be used instead of env vars.
    #[arg(long)]
    genesis_path: Option<std::path::PathBuf>,
}

#[derive(Debug, Clone)]
struct ComponentsToRun(Vec<ViaComponent>);

impl FromStr for ComponentsToRun {
    type Err = String;

    fn from_str(s: &str) -> Result<Self, Self::Err> {
        let components = s.split(',').try_fold(vec![], |mut acc, component_str| {
            let components = ViaComponents::from_str(component_str.trim())?;
            acc.extend(components.0);
            Ok::<_, String>(acc)
        })?;
        Ok(Self(components))
    }
}
```

Important caveat: although `--config-path`, `--contracts-config-path`, and `--genesis-path` are declared, passing any of them currently returns an error. From `main()`:

```rust
// core/bin/via_server/src/main.rs
    // Load configurations
    let configs = match opt.config_path {
        Some(_path) => {
            return Err(anyhow::anyhow!(
                "The Via Server does not support configuration files at this point. Please use env variables."
            ));
        }
        None => {
            tracing::info!("Loading configs from env");
            ViaTempConfigStore::general()?
        }
    };
```

`--secrets-path` and `--wallets-path` do accept YAML files; without them, secrets and wallets are loaded from env vars.

```bash
# Run with the default component set
via_server

# Run a subset of components
via_server --components api,btc,tree

# Generate genesis (uses a temporary DB)
via_server --genesis
```

### via_verifier

The verifier binary has exactly three flags, all optional YAML paths:

```rust
// via_verifier/bin/verifier_server/src/main.rs
#[derive(Debug, Parser)]
#[command(author = "Via verifer", version, about = "Via verifer node", long_about = None)]
struct Cli {
    /// Path to the YAML config. If set, it will be used instead of env vars.
    #[arg(long)]
    config_path: Option<std::path::PathBuf>,

    /// Path to the YAML with secrets. If set, it will be used instead of env vars.
    #[arg(long)]
    secrets_path: Option<std::path::PathBuf>,

    /// Path to the wallets config. If set, it will be used instead of env vars.
    #[arg(long)]
    wallets_path: Option<std::path::PathBuf>,
}
```

There is no `--components` flag and no verifier/coordinator mode flag on the binary itself; the node's role and all other settings come from env-loaded configs (`ViaGeneralVerifierConfig` built from `FromEnv` in `main()`).

### via_indexer_bin

The indexer binary defines **no CLI arguments whatsoever**. Its `main()` builds the whole configuration from environment variables:

```rust
// via_indexer/bin/indexer/src/main.rs
fn main() -> anyhow::Result<()> {
    let via_indexer_config = ViaIndexerConfig {
        health_check: HealthCheckConfig::from_env()?,
        postgres_config: PostgresConfig::from_env()?,
        prometheus_config: PrometheusConfig::from_env()?,
        via_btc_client_config: ViaBtcClientConfig::from_env()?,
        via_btc_watch_config: ViaBtcWatchConfig::from_env()?,
        via_genesis_config: ViaGenesisConfig::from_env()?,
        observability_config: ObservabilityConfig::from_env()?,
        secrets: ViaSecrets {
            base_secrets: Secrets {
                consensus: None,
                database: DatabaseSecrets::from_env().ok(),
                l1: None,
                data_availability: None,
            },
            via_l1: ViaL1Secrets::from_env().ok(),
            via_l2: None,
            via_da: None,
        },
        via_bridge_config: ViaBridgeConfig::from_env()?,
    };

    let node_builder = node_builder::ViaNodeBuilder::new(via_indexer_config.clone())?;

    let observability_guard = {
        // Observability initialization should be performed within tokio context.
        let _context_guard = node_builder.runtime_handle().enter();
        via_indexer_config.observability_config.install()?
    };

    // Build the node

    let node = node_builder.build()?;
    node.run(observability_guard)?;

    Ok(())
}
```

Start it in development through the CLI (`via indexer`), or directly with `cargo run --bin via_indexer_bin` inside a loaded environment.

## Server Components

Valid strings for `via_server --components`, verbatim from the source of truth:

```rust
// core/lib/zksync_core_leftovers/src/lib.rs
#[derive(Debug, Clone, Copy, PartialEq)]
pub enum ViaComponent {
    /// Public Web3 API running on HTTP server.
    HttpApi,
    /// Public Web3 API (including PubSub) running on WebSocket server.
    WsApi,
    /// Metadata calculator.
    Tree,
    /// Merkle tree API.
    TreeApi,
    /// State keeper.
    StateKeeper,
    /// Component for housekeeping task such as cleaning blobs from GCS, reporting metrics etc.
    Housekeeper,
    /// Component for exposing APIs to prover for providing proof generation data and accepting proofs.
    ProofDataHandler,
    /// Component generating commitment for L1 batches.
    CommitmentGenerator,
    /// Component sending a pubdata to the DA layers.
    DADispatcher,
    /// VM runner-based component that saves protective reads to Postgres.
    VmRunnerProtectiveReads,
    /// VM runner-based component that saves VM execution data for basic witness generation.
    VmRunnerBwip,
    /// Component that interacts with Bitcoin network
    Btc,
    /// Component that writes data to Celestia network
    Celestia,
}

#[derive(Debug)]
pub struct ViaComponents(pub Vec<ViaComponent>);

impl FromStr for ViaComponents {
    type Err = String;

    fn from_str(s: &str) -> Result<ViaComponents, String> {
        match s {
            "api" => Ok(ViaComponents(vec![
                ViaComponent::HttpApi,
                ViaComponent::WsApi,
            ])),
            "http_api" => Ok(ViaComponents(vec![ViaComponent::HttpApi])),
            "ws_api" => Ok(ViaComponents(vec![ViaComponent::WsApi])),
            "tree" => Ok(ViaComponents(vec![ViaComponent::Tree])),
            "tree_api" => Ok(ViaComponents(vec![ViaComponent::TreeApi])),
            "state_keeper" => Ok(ViaComponents(vec![ViaComponent::StateKeeper])),
            "housekeeper" => Ok(ViaComponents(vec![ViaComponent::Housekeeper])),
            "proof_data_handler" => Ok(ViaComponents(vec![ViaComponent::ProofDataHandler])),
            "commitment_generator" => Ok(ViaComponents(vec![ViaComponent::CommitmentGenerator])),
            "da_dispatcher" => Ok(ViaComponents(vec![ViaComponent::DADispatcher])),
            "vm_runner_protective_reads" => {
                Ok(ViaComponents(vec![ViaComponent::VmRunnerProtectiveReads]))
            }
            "vm_runner_bwip" => Ok(ViaComponents(vec![ViaComponent::VmRunnerBwip])),
            "btc" => Ok(ViaComponents(vec![ViaComponent::Btc])),
            "celestia" => Ok(ViaComponents(vec![ViaComponent::Celestia])),
            other => Err(format!("{} is not a valid component name", other)),
        }
    }
}
```

Note that `eth`, `data_fetcher`, `eth_watcher`, `consensus`, and similar ZKsync Era component names are **not** valid for `via_server`; they belong to the upstream Era `Components` parser, not `ViaComponents`.

## Runnable Examples

These are `cargo` examples, not installed CLIs. They are run with `cargo run --example <name>` from the owning crate (the `via bootstrap system-bootstrapping` command does exactly this for the `bootstrap` example).

`core/lib/via_btc_client/examples/`:

- `bootstrap.rs`
- `data_inscription_example.rs`
- `deposit_opreturn.rs`
- `deposit.rs`
- `fee_history.rs`
- `fee_rate.rs`
- `indexer_init_example.rs`
- `inscriber.rs`
- `parse_bootstrap.rs`
- `propose_new_bridge.rs`
- `tx_with_opreturn.rs`
- `upgrade_inscription_parsing.rs`
- `upgrade_system_contracts.rs`
- `verify_batch.rs`

`via_verifier/lib/via_musig2/examples/`:

- `compute_musig2.rs`
- `coordinator.rs`
- `key_generation_setup.rs`
- `musig2_with_script_path.rs`
- `transfer_utxos_from_bridge.rs`
- `verify_partial_sig.rs`
- `withdrawal.rs`

## Environment Variables

### Workflow variables

```bash
# Root of the via-core checkout; required by bin/via and the CLI entry point
export VIA_HOME=/path/to/via-core

# Current environment name (selects etc/env/target/<env>.env); managed by `via env`
export VIA_ENV=dev
```

### Database URLs

These are the variables the `via db` tooling actually reads (`infrastructure/via/src/database.ts`):

```bash
DATABASE_URL            # core database
DATABASE_PROVER_URL     # prover database (optional)
DATABASE_VERIFIER_URL   # verifier database
DATABASE_INDEXER_URL    # indexer database

# Test databases used by `via db reset-test`
TEST_DATABASE_URL
TEST_DATABASE_PROVER_URL
TEST_DATABASE_VERIFIER_URL
TEST_DATABASE_INDEXER_URL
```

### Bitcoin client configuration

Both the Bitcoin client config and the L1 RPC secrets load from the `VIA_BTC_CLIENT_` env prefix:

```rust
// core/lib/env_config/src/via_btc_client.rs
impl FromEnv for ViaBtcClientConfig {
    fn from_env() -> anyhow::Result<Self> {
        envy_load("via_btc_client", "VIA_BTC_CLIENT_")
    }
}
```

```rust
// core/lib/config/src/configs/via_secrets.rs
pub struct ViaL1Secrets {
    /// URL of the Bitcoin node RPC.
    pub rpc_url: SensitiveUrl,

    /// Username for the Bitcoin node RPC.
    pub rpc_user: String,

    /// Password for the Bitcoin node RPC.
    pub rpc_password: String,
}
```

The corresponding base config values live in `etc/env/base/via_btc_client.toml` (`network`, `rpc_url`, `external_apis`, `fee_strategies`, `use_rpc_for_fee_rate`) and `etc/env/base/via_private.toml` (`rpc_user`, `rpc_password`). Compiled to env vars they become, for example:

```bash
export VIA_BTC_CLIENT_NETWORK="regtest"
export VIA_BTC_CLIENT_RPC_URL="http://0.0.0.0:18443"
export VIA_BTC_CLIENT_RPC_USER="rpcuser"
export VIA_BTC_CLIENT_RPC_PASSWORD="rpcpassword"
```

### Logging

Logging uses the standard `RUST_LOG` filter:

```bash
export RUST_LOG=info
export RUST_LOG=via_btc_client=debug
```

## Command Workflows

### Development Workflow

Grounded in `docs/via_guides/development.md`:

```bash
# 1. Initialize the project (containers, databases, contracts)
via init

# 2. Or, if you only need containers without full init:
via up      # bitcoind, bitcoin-cli, postgres, celestia-node
via down    # shut everything down

# 3. Start the sequencer
via server

# 4. Start the verifier and/or indexer (each in its own environment)
via verifier --network regtest
via indexer --network regtest

# 5. Bridge some test BTC
via token deposit --amount 1 --receiver-l2-address <addr> --bridge-address <bridge>

# 6. Run tests
via test rust
via test rust -p 'via_*'
```

### Resetting a broken environment

From the development guide:

```bash
# Remove generated configs, database and backups, take containers down
via clean --all

# Then re-initialize
via init
```

If `via init` still fails after pulling new changes, remove `$VIA_HOME/etc/env/target/dev.env` and run `via init` again.

### Contracts

```bash
# Re-build contracts
via contract build

# Deploy L2 contracts
via contract deploy-l2
```

## Troubleshooting

### 1. `via: command not found`

```bash
# Make sure the wrapper is on PATH and VIA_HOME is set
export VIA_HOME=/path/to/via-core
export PATH=$VIA_HOME/bin:$PATH

# Install JS dependencies if the CLI fails to start
via run yarn   # or: yarn install from $VIA_HOME
```

### 2. Database connection errors

```bash
# Test database connections
psql $DATABASE_URL -c "SELECT 1;"
psql $DATABASE_INDEXER_URL -c "SELECT 1;"

# Wait for the database to become ready
via db wait --core
```

### 3. Bitcoin RPC connection issues

```bash
# Test Bitcoin RPC directly
curl -u $VIA_BTC_CLIENT_RPC_USER:$VIA_BTC_CLIENT_RPC_PASSWORD \
  -d '{"jsonrpc":"1.0","id":"test","method":"getblockchaininfo","params":[]}' \
  -H 'content-type: text/plain;' \
  $VIA_BTC_CLIENT_RPC_URL

# In development the bitcoind container is managed by `via up` / `via down`
via docker restart <container>
```

### 4. Invalid component name

If `via_server` fails with `<name> is not a valid component name`, check the component string against the [ViaComponents parser](#server-components). Era-only names such as `eth` or `consensus` are rejected.

### Debug logging

```bash
export RUST_LOG=debug
via server 2>&1 | tee debug.log
```

All commands above are quoted from or verified against the `via-core` sources: `infrastructure/via/src/`, `core/bin/via_server/src/main.rs`, `via_verifier/bin/verifier_server/src/main.rs`, `via_indexer/bin/indexer/src/main.rs`, and `core/lib/zksync_core_leftovers/src/lib.rs`.
