# Via L2 Bitcoin ZK-Rollup Execution Environment / VM

This document provides a comprehensive overview of the Execution Environment (VM) in the Via L2 Bitcoin ZK-Rollup, detailing its architecture, implementation, and interactions with other system components.

## Table of Contents

1. [Introduction](#introduction)
2. [Architecture Overview](#architecture-overview)
3. [Core Components](#core-components)
4. [VM Implementation Details](#vm-implementation-details)
5. [Execution Process](#execution-process)
6. [Bootloader](#bootloader)
7. [VM Versioning](#vm-versioning)
8. [Interactions with Other Components](#interactions-with-other-components)
9. [Key Data Structures](#key-data-structures)
10. [Code Structure](#code-structure)

## Introduction

The Execution Environment, or Virtual Machine (VM), is the core component of the Via L2 Bitcoin ZK-Rollup responsible for executing transactions and calculating state transitions. It provides a deterministic environment for running smart contracts and processing transactions, ensuring that all nodes in the network reach the same state given the same inputs.

The Via L2 system is built on a zkEVM architecture, which consists of three main components:
1. The zkEVM (Virtual Machine)
2. System Contracts (deployed at predefined addresses)
3. The Bootloader (a special program that processes transactions)

## Architecture Overview

The Via L2 execution environment follows a multi-VM architecture, allowing for different VM versions to coexist. This is implemented through the `multivm` crate, which provides a wrapper over several versions of the VM. This architecture enables smooth protocol upgrades while maintaining backward compatibility.

From the VM's perspective, it runs a single program called the bootloader, which internally processes multiple transactions. The bootloader interacts with system contracts to execute transactions and enforce the rules of the rollup.

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      State Keeper                           │
└───────────────────────────────┬─────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────┐
│                      MultiVM Wrapper                        │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────────┐    │
│  │  vm_1_3_2   │   │  vm_1_4_1   │   │    vm_latest    │    │
│  └─────────────┘   └─────────────┘   └─────────────────┘    │
└───────────────────────────────┬─────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────┐
│                        Bootloader                           │
└───────────────────────────────┬─────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────┐
│                     System Contracts                        │
└─────────────────────────────────────────────────────────────┘
```

Note that there is no separate Bitcoin VM implementation: the Via-specific `VmVersion::VmBitcoin1_0_0` variant is dispatched to the inherited zksync-era `vm_latest` implementation (see [VM Versioning](#vm-versioning)).

## Core Components

### VM Interface

The VM interface is defined in `core/lib/vm_interface/src/vm.rs` (inherited zksync-era machinery) and provides a common interface for all VM versions. It includes methods for:

- Pushing transactions to the bootloader memory
- Executing the next VM step with custom tracers (`inspect`)
- Starting new L2 blocks
- Executing transactions with optional bytecode compression
- Finishing batches

The core trait, verbatim from the source:

```rust
// core/lib/vm_interface/src/vm.rs
pub trait VmInterface {
    type TracerDispatcher: Default;

    /// Pushes a transaction to bootloader memory for future execution with bytecode compression (if it's supported by the VM).
    ///
    /// # Return value
    ///
    /// Returns preprocessing results, such as compressed bytecodes. The results may borrow from the VM state,
    /// so you may want to inspect results before next operations with the VM, or clone the necessary parts.
    fn push_transaction(&mut self, tx: Transaction) -> PushTransactionResult<'_>;

    /// Executes the next VM step (either next transaction or bootloader or the whole batch)
    /// with custom tracers.
    fn inspect(
        &mut self,
        dispatcher: &mut Self::TracerDispatcher,
        execution_mode: InspectExecutionMode,
    ) -> VmExecutionResultAndLogs;

    /// Start a new L2 block.
    fn start_new_l2_block(&mut self, l2_block_env: L2BlockEnv);

    /// Executes the provided transaction with optional bytecode compression using custom tracers.
    fn inspect_transaction_with_bytecode_compression(
        &mut self,
        tracer: &mut Self::TracerDispatcher,
        tx: Transaction,
        with_compression: bool,
    ) -> (BytecodeCompressionResult<'_>, VmExecutionResultAndLogs);

    /// Execute batch till the end and return the result, with final execution state
    /// and bootloader memory.
    fn finish_batch(&mut self, pubdata_builder: Rc<dyn PubdataBuilder>) -> FinishedL1Batch;
}
```

Convenience methods such as `execute` and `execute_transaction_with_bytecode_compression` are not part of `VmInterface` itself; they live in an extension trait with default implementations that delegate to the `inspect*` methods:

```rust
// core/lib/vm_interface/src/vm.rs
/// Extension trait for [`VmInterface`] that provides some additional methods.
pub trait VmInterfaceExt: VmInterface {
    /// Executes the next VM step (either next transaction or bootloader or the whole batch).
    fn execute(&mut self, execution_mode: InspectExecutionMode) -> VmExecutionResultAndLogs {
        self.inspect(&mut <Self::TracerDispatcher>::default(), execution_mode)
    }

    /// Executes a transaction with optional bytecode compression.
    fn execute_transaction_with_bytecode_compression(
        &mut self,
        tx: Transaction,
        with_compression: bool,
    ) -> (BytecodeCompressionResult<'_>, VmExecutionResultAndLogs) {
        self.inspect_transaction_with_bytecode_compression(
            &mut <Self::TracerDispatcher>::default(),
            tx,
            with_compression,
        )
    }
}

impl<T: VmInterface> VmInterfaceExt for T {}
```

### VM Factory

The VM Factory trait is responsible for creating new VM instances:

```rust
// core/lib/vm_interface/src/vm.rs
/// Encapsulates creating VM instance based on the provided environment.
pub trait VmFactory<S>: VmInterface {
    /// Creates a new VM instance.
    fn new(batch_env: L1BatchEnv, system_env: SystemEnv, storage: StoragePtr<S>) -> Self;
}
```

### VM History Mode

The VM also supports history mode, which allows for creating snapshots of the VM state and rolling back to previous states:

```rust
// core/lib/vm_interface/src/vm.rs
/// Methods of VM requiring history manipulations.
///
/// # Snapshot workflow
///
/// External callers must follow the following snapshot workflow:
///
/// - Each new snapshot created using `make_snapshot()` must be either popped or rolled back before creating the following snapshot.
///   OTOH, it's not required to call either of these methods by the end of VM execution.
/// - `pop_snapshot_no_rollback()` may be called spuriously, when no snapshot was created. It is a no-op in this case.
///
/// These rules guarantee that at each given moment, a VM instance has at most one snapshot (unless the VM makes snapshots internally),
/// which may allow additional VM optimizations.
pub trait VmInterfaceHistoryEnabled: VmInterface {
    /// Create a snapshot of the current VM state and push it into memory.
    fn make_snapshot(&mut self);

    /// Roll back VM state to the latest snapshot and destroy the snapshot.
    fn rollback_to_the_latest_snapshot(&mut self);

    /// Pop the latest snapshot from memory and destroy it. If there are no snapshots, this should be a no-op
    /// (i.e., the VM must not panic in this case).
    fn pop_snapshot_no_rollback(&mut self);
}
```

## VM Implementation Details

The VM implementation is located in `core/lib/multivm/src/versions/vm_latest/vm.rs`. The latest VM version is based on the zkEVM architecture and includes:

```rust
// core/lib/multivm/src/versions/vm_latest/vm.rs
/// Main entry point for Virtual Machine integration.
/// The instance should process only one l1 batch
#[derive(Debug)]
pub struct Vm<S: WriteStorage, H: HistoryMode> {
    pub(crate) bootloader_state: BootloaderState,
    // Current state and oracles of virtual machine
    pub(crate) state: ZkSyncVmState<S, H::Vm1_5_0>,
    pub(crate) storage: StoragePtr<S>,
    pub(crate) system_env: SystemEnv,
    pub(crate) batch_env: L1BatchEnv,
    // Snapshots for the current run
    pub(crate) snapshots: Vec<VmSnapshot>,
    pub(crate) subversion: MultiVmSubversion,
    _phantom: std::marker::PhantomData<H>,
}
```

The VM implementation includes:
- The bootloader state, which manages the bootloader memory and execution
- The VM state, which includes the current execution state and oracles
- The storage, which provides access to the state
- The system environment, which includes system parameters
- The batch environment, which includes batch parameters
- Snapshots for history mode
- The VM subversion (`MultiVmSubversion`), which indicates the specific version of the VM

### Core VM Logic

The core VM logic is based on the zkEVM architecture, which is a specialized virtual machine designed for generating zero-knowledge proofs of execution. The VM executes bytecode in a deterministic manner, tracking all operations and state changes to produce an execution trace that can be used to generate a proof.

The VM operates on a stack-based architecture similar to the Ethereum Virtual Machine (EVM), but with modifications to make it more amenable to zero-knowledge proof generation. It supports standard EVM opcodes as well as additional opcodes specific to the zkEVM.

### Execution Modes

The VM supports three execution modes, defined in `core/lib/vm_interface/src/types/inputs/execution_mode.rs`:

```rust
// core/lib/vm_interface/src/types/inputs/execution_mode.rs
/// Execution mode determines when the virtual machine execution should stop.
/// We are also using a different set of tracers, depending on the selected mode - for example for OneTx,
/// we use Refund Tracer, and for Bootloader we use 'DefaultTracer` in a special mode to track the Bootloader return code
/// Flow of execution:
/// VmStarted -> Enter the bootloader -> Tx1 -> Tx2 -> ... -> TxN ->
/// -> Terminate bootloader execution -> Exit bootloader -> VmStopped
#[derive(Debug, Copy, Clone)]
pub enum VmExecutionMode {
    /// Stop after executing the next transaction.
    OneTx,
    /// Stop after executing the entire batch.
    Batch,
    /// Stop after executing the entire bootloader. But before you exit the bootloader.
    Bootloader,
}
```

The `inspect` method of `VmInterface` accepts only the subset of modes that require no additional input:

```rust
// core/lib/vm_interface/src/types/inputs/execution_mode.rs
/// Subset of `VmExecutionMode` variants that do not require any additional input
/// and can be invoked with `inspect` method.
#[derive(Debug, Copy, Clone)]
pub enum InspectExecutionMode {
    /// Stop after executing the next transaction.
    OneTx,
    /// Stop after executing the entire bootloader. But before you exit the bootloader.
    Bootloader,
}
```

These modes determine when the VM execution should stop and what tracers should be used during execution.

### Gas Calculation

Gas is calculated and enforced during execution to limit the computational resources used by transactions. The VM tracks gas usage for each operation and ensures that transactions do not exceed their gas limit.

The `gas.rs` module in `vm_latest` is small: it computes the computational gas used between two points in execution. Starting from VM 1.5.0, pubdata is implicitly charged from the user's gas limit rather than explicitly reduced from the gas in the VM state. The full file:

```rust
// core/lib/multivm/src/versions/vm_latest/implementation/gas.rs
use crate::{interface::storage::WriteStorage, vm_latest::vm::Vm, HistoryMode};

impl<S: WriteStorage, H: HistoryMode> Vm<S, H> {
    pub(crate) fn calculate_computational_gas_used(&self, gas_remaining_before: u32) -> u32 {
        // Starting from VM version 1.5.0 pubdata was implicitly charged from users' gasLimit instead of
        // explicitly reduced from the `gas` in the VM state
        gas_remaining_before
            .checked_sub(self.gas_remaining())
            .expect("underflow")
    }
}
```

Refund calculation (including operator-suggested refunds and pubdata accounting) is handled by the `RefundsTracer` in `core/lib/multivm/src/versions/vm_latest/tracers/refunds.rs`, which is enabled when running in `VmExecutionMode::OneTx`.

### State Interaction

The VM interacts with the state through the storage interface, which provides methods for reading and writing storage slots. The storage traits are defined in `core/lib/vm_interface/src/storage/mod.rs`:

```rust
// core/lib/vm_interface/src/storage/mod.rs
/// Functionality to read from the VM storage.
pub trait ReadStorage: fmt::Debug {
    /// Read value of the key.
    fn read_value(&mut self, key: &StorageKey) -> StorageValue;

    /// Checks whether a write to this storage at the specified `key` would be an initial write.
    /// Roughly speaking, this is the case when the storage doesn't contain `key`, although
    /// in case of mutable storage, the caveats apply (a write to a key that is present
    /// in the storage but was not committed is still an initial write).
    fn is_write_initial(&mut self, key: &StorageKey) -> bool;

    /// Load the factory dependency code by its hash.
    fn load_factory_dep(&mut self, hash: H256) -> Option<Vec<u8>>;

    /// Returns whether a bytecode hash is "known" to the system.
    fn is_bytecode_known(&mut self, bytecode_hash: &H256) -> bool {
        let code_key = get_known_code_key(bytecode_hash);
        self.read_value(&code_key) != H256::zero()
    }

    /// Retrieves the enumeration index for a given `key`.
    fn get_enumeration_index(&mut self, key: &StorageKey) -> Option<u64>;
}

/// Functionality to write to the VM storage in a batch.
///
/// So far, this trait is implemented only for [`StorageView`].
pub trait WriteStorage: ReadStorage {
    /// Returns the map with the key–value pairs read by this batch.
    fn read_storage_keys(&self) -> &HashMap<StorageKey, StorageValue>;

    /// Sets the new value under a given key and returns the previous value.
    fn set_value(&mut self, key: StorageKey, value: StorageValue) -> StorageValue;

    /// Returns a map with the key–value pairs updated by this batch.
    fn modified_storage_keys(&self) -> &HashMap<StorageKey, StorageValue>;

    /// Returns the number of read / write ops for which the value was read from the underlying
    /// storage.
    fn missed_storage_invocations(&self) -> usize;
}

/// Smart pointer to [`WriteStorage`].
pub type StoragePtr<S> = Rc<RefCell<S>>;
```

The VM produces storage logs for each storage operation, which are then used to update the state in the State Keeper.

## Execution Process

The execution process is implemented in `core/lib/multivm/src/versions/vm_latest/implementation/execution.rs`:

```rust
// core/lib/multivm/src/versions/vm_latest/implementation/execution.rs
impl<S: WriteStorage, H: HistoryMode> Vm<S, H> {
    pub(crate) fn inspect_inner(
        &mut self,
        dispatcher: &mut TracerDispatcher<S, H::Vm1_5_0>,
        execution_mode: VmExecutionMode,
        custom_pubdata_tracer: Option<PubdataTracer<S>>,
    ) -> VmExecutionResultAndLogs {
        let mut enable_refund_tracer = false;
        if let VmExecutionMode::OneTx = execution_mode {
            // Move the pointer to the next transaction
            self.bootloader_state.move_tx_to_execute_pointer();
            enable_refund_tracer = true;
        }

        let (_, result) = self.inspect_and_collect_results(
            dispatcher,
            execution_mode,
            enable_refund_tracer,
            custom_pubdata_tracer,
        );
        result
    }

    /// Execute VM with given traces until the stop reason is reached.
    /// Collect the result from the default tracers.
    fn inspect_and_collect_results(
        &mut self,
        dispatcher: &mut TracerDispatcher<S, H::Vm1_5_0>,
        execution_mode: VmExecutionMode,
        with_refund_tracer: bool,
        custom_pubdata_tracer: Option<PubdataTracer<S>>,
    ) -> (VmExecutionStopReason, VmExecutionResultAndLogs) {
        let refund_tracers = with_refund_tracer
            .then_some(RefundsTracer::new(self.batch_env.clone(), self.subversion));
        let mut tx_tracer: DefaultExecutionTracer<S, H::Vm1_5_0> = DefaultExecutionTracer::new(
            self.system_env.default_validation_computational_gas_limit,
            self.system_env
                .base_system_smart_contracts
                .evm_emulator
                .is_some(),
            execution_mode,
            mem::take(dispatcher),
            self.storage.clone(),
            refund_tracers,
            custom_pubdata_tracer.or_else(|| {
                Some(PubdataTracer::new(
                    self.batch_env.clone(),
                    execution_mode,
                    self.subversion,
                    None,
                ))
            }),
            self.subversion,
        );

        let timestamp_initial = Timestamp(self.state.local_state.timestamp);
        let cycles_initial = self.state.local_state.monotonic_cycle_counter;
        let gas_remaining_before = self.gas_remaining();

        let stop_reason = self.execute_with_default_tracer(&mut tx_tracer);

        let gas_remaining_after = self.gas_remaining();

        let logs = self.collect_execution_logs_after_timestamp(timestamp_initial);

        let (refunds, pubdata_published) = tx_tracer
            .refund_tracer
            .as_ref()
            .map(|x| (x.get_refunds(), x.pubdata_published()))
            .unwrap_or_default();

        let statistics = self.get_statistics(
            timestamp_initial,
            cycles_initial,
            gas_remaining_before,
            gas_remaining_after,
            pubdata_published,
            logs.total_log_queries_count,
            circuit_statistic_from_cycles(tx_tracer.circuits_tracer.statistics),
        );
        let result = tx_tracer.result_tracer.into_result();
        let factory_deps_marked_as_known = VmEvent::extract_bytecodes_marked_as_known(&logs.events);
        let dynamic_factory_deps = self.decommit_dynamic_bytecodes(factory_deps_marked_as_known);
        *dispatcher = tx_tracer.dispatcher;

        let result = VmExecutionResultAndLogs {
            result,
            logs,
            statistics,
            refunds,
            dynamic_factory_deps,
        };

        (stop_reason, result)
    }

    /// Execute vm with given tracers until the stop reason is reached.
    fn execute_with_default_tracer(
        &mut self,
        tracer: &mut DefaultExecutionTracer<S, H::Vm1_5_0>,
    ) -> VmExecutionStopReason {
        tracer.initialize_tracer(&mut self.state);
        let result = loop {
            // Sanity check: we should never reach the maximum value, because then we won't be able to process the next cycle.
            assert_ne!(
                self.state.local_state.monotonic_cycle_counter,
                u32::MAX,
                "VM reached maximum possible amount of cycles. Vm state: {:?}",
                self.state
            );

            self.state
                .cycle(tracer)
                .expect("Failed execution VM cycle.");

            if let TracerExecutionStatus::Stop(reason) =
                tracer.finish_cycle(&mut self.state, &mut self.bootloader_state)
            {
                break VmExecutionStopReason::TracerRequestedStop(reason);
            }
            if self.has_ended() {
                break VmExecutionStopReason::VmFinished;
            }
        };
        tracer.after_vm_execution(&mut self.state, &self.bootloader_state, result.clone());
        result
    }

    fn has_ended(&self) -> bool {
        match vm_may_have_ended_inner(&self.state) {
            None | Some(VmExecutionResult::MostLikelyDidNotFinish(_, _)) => false,
            Some(
                VmExecutionResult::Ok(_) | VmExecutionResult::Revert(_) | VmExecutionResult::Panic,
            ) => true,
        }
    }
}
```

The execution process involves:
1. Setting up tracers for monitoring execution
2. Executing the VM cycle by cycle
3. Checking for stop conditions after each cycle
4. Collecting execution logs and results
5. Returning the execution result and logs

### Transaction Execution

Transactions are executed by pushing them to the bootloader memory and then executing the bootloader. The bootloader processes transactions one by one, validating them, executing them, and updating the state accordingly.

The transaction execution process includes:
1. Validating the transaction format and parameters
2. Deducting fees upfront
3. Executing the transaction code
4. Handling refunds
5. Updating the state with the changes

### State Diffs

The VM calculates state diffs during execution, which are then applied to the state by the State Keeper. State diffs include:
- Storage writes
- Contract deployments
- Events
- L2-to-L1 messages

## Bootloader

The bootloader is a special "quasi" system contract that serves as the entry point for transaction execution. Unlike other system contracts, the bootloader's code is not stored on L2 but is loaded directly by its hash.

### Role and Purpose

The bootloader serves two main purposes:
1. **Batch Processing**: Instead of processing transactions one by one, the bootloader processes an entire batch of transactions as a single program, making the system more efficient.
2. **Transaction Validation**: The bootloader validates transactions, enforces rules, and manages the execution flow.

### Implementation Details

The bootloader is written in Yul and has a formal address of `0x8001`, defined as a constant in via-core:

```rust
// core/lib/constants/src/contracts.rs
pub const BOOTLOADER_ADDRESS: Address = H160([
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x80, 0x01,
]);
```

It has a special heap that acts as an interface between the server and the zkEVM. The server fills the bootloader's heap with transaction data, and the bootloader executes these transactions.

The bootloader's execution flow includes:
1. Reading batch information and initializing the environment
2. Processing transactions one by one:
   - Validating transaction format and parameters
   - Deducting fees
   - Executing the transaction
   - Handling refunds
3. Finalizing the batch and publishing state changes

The bootloader's hash is stored on L1 and can only be changed as part of a system upgrade.

### Bootloader State

The bootloader state is managed by the `BootloaderState` struct, which is responsible for:
- Managing the bootloader memory
- Tracking the current transaction being executed
- Handling transaction validation and execution
- Managing L2 block information

## VM Versioning

The Via L2 system supports multiple VM versions through the multiVM architecture. Each version has its own implementation of the VM interface and is selected based on the protocol version.

The VM versions are defined in `core/lib/multivm/src/versions/` (all inherited zksync-era EraVM machinery; the directory also contains `shadow/`, `testonly/`, and `shared.rs` support modules) and include:
- `vm_1_3_2`: An older VM version
- `vm_1_4_1`: An older VM version
- `vm_1_4_2`: An older VM version
- `vm_boojum_integration`: A VM version with Boojum integration
- `vm_fast`: A VM version optimized for speed
- `vm_latest`: The latest VM version
- `vm_m5`: A VM version with specific features
- `vm_m6`: A VM version with specific features
- `vm_refunds_enhancement`: A VM version with enhanced refunds
- `vm_virtual_blocks`: A VM version with virtual blocks support

The full list of VM versions is defined in the `VmVersion` enum, with the Via-specific `VmBitcoin1_0_0` variant as the latest:

```rust
// core/lib/basic_types/src/vm.rs
#[derive(Debug, Clone, Copy)]
pub enum VmVersion {
    M5WithoutRefunds,
    M5WithRefunds,
    M6Initial,
    M6BugWithCompressionFixed,
    Vm1_3_2,
    VmVirtualBlocks,
    VmVirtualBlocksRefundsEnhancement,
    VmBoojumIntegration,
    Vm1_4_1,
    Vm1_4_2,
    Vm1_5_0SmallBootloaderMemory,
    Vm1_5_0IncreasedBootloaderMemory,
    VmGateway,
    VmBitcoin1_0_0,
}

impl VmVersion {
    /// Returns the latest supported VM version.
    pub const fn latest() -> VmVersion {
        Self::VmBitcoin1_0_0
    }
}
```

Within `vm_latest`, the `VmBitcoin1_0_0` version maps to a subversion of the v1.5.0 VM:

```rust
// core/lib/multivm/src/versions/vm_latest/vm.rs
/// MultiVM-specific addition.
///
/// In the first version of the v1.5.0 release, the bootloader memory was too small, so a new
/// version was released with increased bootloader memory. The version with the small bootloader memory
/// is available only on internal staging environments.
#[derive(Debug, Copy, Clone)]
pub(crate) enum MultiVmSubversion {
    /// The initial version of v1.5.0, available only on staging environments.
    SmallBootloaderMemory,
    /// The final correct version of v1.5.0
    IncreasedBootloaderMemory,
    /// VM for post-gateway versions.
    Gateway,
}
```

```rust
// core/lib/multivm/src/versions/vm_latest/vm.rs
#[derive(Debug)]
pub(crate) struct VmVersionIsNotVm150Error;
impl TryFrom<VmVersion> for MultiVmSubversion {
    type Error = VmVersionIsNotVm150Error;
    fn try_from(value: VmVersion) -> Result<Self, Self::Error> {
        match value {
            VmVersion::Vm1_5_0SmallBootloaderMemory => Ok(Self::SmallBootloaderMemory),
            VmVersion::Vm1_5_0IncreasedBootloaderMemory => Ok(Self::IncreasedBootloaderMemory),
            VmVersion::VmGateway => Ok(Self::Gateway),
            VmVersion::VmBitcoin1_0_0 => Ok(Self::IncreasedBootloaderMemory),
            _ => Err(VmVersionIsNotVm150Error),
        }
    }
}
```

This indicates that the Bitcoin VM version is based on the VM 1.5.0 with increased bootloader memory.

## Interactions with Other Components

### Interaction with State Keeper

The VM interacts with the State Keeper, which is responsible for maintaining the state of the rollup. The State Keeper processes transactions, executes them using the VM, and updates the state accordingly. In via-core, the Via-specific state keeper lives in `core/node/via_state_keeper/` (with its batch executor integration in `core/node/via_state_keeper/src/executor/`), alongside the inherited `core/node/state_keeper/` crate.

From the State Management documentation:
> The Execution Environment (VM) executes transactions and produces state diffs. These diffs are then applied to the state by the State Keeper.

The State Keeper uses the VM to:
1. Execute transactions
2. Calculate state diffs
3. Determine transaction success or failure
4. Collect execution metrics

### Interaction with System Contracts

The VM interacts with system contracts to execute transactions and enforce the rules of the rollup. System contracts are privileged contracts deployed at predefined addresses that have special permissions within the system.

Key system contracts that interact with the VM include:
- **AccountCodeStorage**: Stores and manages account code
- **NonceHolder**: Manages account nonces for transactions
- **KnownCodesStorage**: Stores known bytecode hashes
- **ContractDeployer**: Handles contract deployment logic
- **L1Messenger**: Facilitates communication from L2 to L1
- **SystemContext**: Provides environmental data like block number, timestamp, etc.

### Interaction with Prover

The VM generates execution traces that are used by the Prover to generate zero-knowledge proofs. The Prover uses these traces to prove that the state transition is valid without revealing the details of the execution.

From the System Contracts documentation:
> The Prover generates zero-knowledge proofs based on the execution trace. System contracts (especially the bootloader) are designed to be efficiently provable.

## Key Data Structures

### VmExecutionResultAndLogs

```rust
// core/lib/vm_interface/src/types/outputs/execution_result.rs
/// Result and logs of the VM execution.
#[derive(Debug, Clone)]
pub struct VmExecutionResultAndLogs {
    pub result: ExecutionResult,
    pub logs: VmExecutionLogs,
    pub statistics: VmExecutionStatistics,
    pub refunds: Refunds,
    /// Dynamic bytecodes decommitted during VM execution (i.e., not present in the storage at the start of VM execution
    /// or in `factory_deps` fields of executed transactions). Currently, the only kind of such codes are EVM bytecodes.
    /// Correspondingly, they may only be present if supported by the VM version, and if the VM is initialized with the EVM emulator base system contract.
    pub dynamic_factory_deps: HashMap<H256, Vec<u8>>,
}
```

This structure contains the result of VM execution, including:
- The execution result (success, revert, or halt)
- Execution logs (storage logs, events, L2-to-L1 logs)
- Execution statistics (gas used, cycles used, etc.)
- Refunds (gas refunded, operator suggested refund)
- Dynamic factory dependencies (EVM bytecodes decommitted during execution, only when the VM runs with an EVM emulator)

The `Refunds` structure referenced above:

```rust
// core/lib/vm_interface/src/types/outputs/execution_result.rs
/// Refunds produced for the user.
#[derive(Debug, Clone, Default, PartialEq)]
pub struct Refunds {
    pub gas_refunded: u64,
    pub operator_suggested_refund: u64,
}
```

### ExecutionResult

```rust
// core/lib/vm_interface/src/types/outputs/execution_result.rs
#[derive(Debug, Clone, PartialEq)]
pub enum ExecutionResult {
    /// Returned successfully
    Success { output: Vec<u8> },
    /// Reverted by contract
    Revert { output: VmRevertReason },
    /// Reverted for various reasons
    Halt { reason: Halt },
}
```

This enum represents the result of execution, which can be:
- Success: The execution completed successfully
- Revert: The execution was reverted by the contract
- Halt: The execution was halted for various reasons

### VmExecutionLogs

```rust
// core/lib/vm_interface/src/types/outputs/execution_result.rs
/// Events/storage logs/l2->l1 logs created within transaction execution.
#[derive(Debug, Clone, Default, PartialEq)]
pub struct VmExecutionLogs {
    pub storage_logs: Vec<StorageLogWithPreviousValue>,
    pub events: Vec<VmEvent>,
    // For pre-boojum VMs, there was no distinction between user logs and system
    // logs and so all the outputted logs were treated as user_l2_to_l1_logs.
    pub user_l2_to_l1_logs: Vec<UserL2ToL1Log>,
    pub system_l2_to_l1_logs: Vec<SystemL2ToL1Log>,
    // This field moved to statistics, but we need to keep it for backward compatibility
    pub total_log_queries_count: usize,
}
```

This structure contains the logs produced during execution, including:
- Storage logs: Changes to storage slots
- Events: Events emitted by contracts
- L2-to-L1 logs: Messages from L2 to L1
- Total log queries count: The total number of log queries

### CurrentExecutionState

```rust
// core/lib/vm_interface/src/types/outputs/execution_state.rs
/// State of the VM since the start of the batch execution.
#[derive(Debug, Clone, PartialEq)]
pub struct CurrentExecutionState {
    /// Events produced by the VM.
    pub events: Vec<VmEvent>,
    /// The deduplicated storage logs produced by the VM.
    pub deduplicated_storage_logs: Vec<StorageLog>,
    /// Hashes of the contracts used by the VM.
    pub used_contract_hashes: Vec<U256>,
    /// L2 to L1 logs produced by the VM.
    pub system_logs: Vec<SystemL2ToL1Log>,
    /// L2 to L1 logs produced by the `L1Messenger`.
    /// For pre-boojum VMs, there was no distinction between user logs and system
    /// logs and so all the outputted logs were treated as user_l2_to_l1_logs.
    pub user_l2_to_l1_logs: Vec<UserL2ToL1Log>,
    /// Refunds returned by `StorageOracle`.
    pub storage_refunds: Vec<u32>,
    /// Pubdata costs returned by `StorageOracle`.
    /// This field is non-empty only starting from v1.5.0.
    /// Note, that it is a signed integer, because the pubdata costs can be negative, e.g. in case
    /// the user rolls back a state diff.
    pub pubdata_costs: Vec<i32>,
}
```

This structure represents the current state of execution, including:
- Events emitted by contracts
- Deduplicated storage logs
- Used contract hashes (as `Vec<U256>`, not a set)
- L2-to-L1 logs (system logs and `L1Messenger` user logs)
- Storage refunds returned by the `StorageOracle` (as `Vec<u32>`)
- Pubdata costs returned by the `StorageOracle` (as `Vec<i32>`, signed because a rolled-back state diff can make costs negative)

## Code Structure

### Key Modules and Files

- **VM Interface**:
  - `core/lib/vm_interface/src/vm.rs`: Core VM interface definition
  - `core/lib/vm_interface/src/types/`: VM interface types
  - `core/lib/vm_interface/src/storage/`: Storage interface

- **MultiVM**:
  - `core/lib/multivm/src/lib.rs`: MultiVM library entry point
  - `core/lib/multivm/src/vm_instance.rs`: VM instance wrapper
  - `core/lib/multivm/src/versions/`: VM version implementations
  - `core/lib/multivm/src/versions/vm_latest/`: Latest VM implementation
  - `core/lib/multivm/src/versions/vm_latest/vm.rs`: VM implementation
  - `core/lib/multivm/src/versions/vm_latest/implementation/`: VM implementation details
  - `core/lib/multivm/src/versions/vm_latest/bootloader_state/`: Bootloader state management

- **Bootloader**:
  - `etc/multivm_bootloaders/`: Compiled bootloader binaries for different VM versions
  - `etc/multivm_bootloaders/vm_bitcoin/`, `etc/multivm_bootloaders/vm_bitcoin_1_0_1/`, `etc/multivm_bootloaders/vm_bitcoin_gateway/`: Bitcoin-specific bootloader builds

### Key Functions and Methods

- **VM Creation**:
  - `Vm::new`: Creates a new VM instance
  - `Vm::new_with_subversion`: Creates a new VM instance with a specific subversion

- **Transaction Execution**:
  - `Vm::push_transaction`: Pushes a transaction to the bootloader memory (returns `PushTransactionResult`)
  - `VmInterfaceExt::execute`: Executes the VM with a specific `InspectExecutionMode` (default implementation delegating to `inspect`)
  - `Vm::inspect`: Executes the VM with custom tracers
  - `Vm::inspect_inner`: Internal implementation of VM execution
  - `Vm::execute_with_default_tracer`: Executes the VM with the default tracer

- **State Management**:
  - `Vm::get_current_execution_state`: Gets the current execution state
  - `Vm::make_snapshot`: Creates a snapshot of the VM state
  - `Vm::rollback_to_the_latest_snapshot`: Rolls back to the latest snapshot
  - `Vm::finish_batch`: Finishes the current batch and returns the result (takes an `Rc<dyn PubdataBuilder>`)

## Conclusion

The Execution Environment (VM) in the Via L2 Bitcoin ZK-Rollup is a sophisticated component that provides a deterministic environment for executing transactions and calculating state transitions. It is based on the zkEVM architecture and supports multiple VM versions through the multiVM architecture.

The VM executes transactions through the bootloader, which processes transactions one by one, validating them, executing them, and updating the state accordingly. The VM produces execution traces that are used by the Prover to generate zero-knowledge proofs, ensuring the validity of state transitions without revealing the details of the execution.

The VM interacts with various components of the system, including the State Keeper, System Contracts, and Prover, to provide a secure and efficient execution environment for the Via L2 Bitcoin ZK-Rollup.