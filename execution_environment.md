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
│  │  VM v1.3.2  │   │  VM v1.4.1  │   │  VM Bitcoin 1.0 │    │
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

## Core Components

### VM Interface

The VM interface is defined in `core/lib/vm_interface/src/vm.rs` and provides a common interface for all VM versions. It includes methods for:

- Pushing transactions to the bootloader memory
- Executing transactions
- Getting the bootloader memory
- Getting the current execution state
- Starting new L2 blocks
- Finishing batches

Key traits defined in the VM interface:

```rust
pub trait VmInterface {
    type TracerDispatcher: Default;

    // Push transaction to bootloader memory
    fn push_transaction(&mut self, tx: Transaction);

    // Execute next VM step
    fn execute(&mut self, execution_mode: VmExecutionMode) -> VmExecutionResultAndLogs;

    // Execute with custom tracers
    fn inspect(
        &mut self,
        dispatcher: Self::TracerDispatcher,
        execution_mode: VmExecutionMode,
    ) -> VmExecutionResultAndLogs;

    // Get bootloader memory
    fn get_bootloader_memory(&self) -> BootloaderMemory;

    // Get last transaction's compressed bytecodes
    fn get_last_tx_compressed_bytecodes(&self) -> Vec<CompressedBytecodeInfo>;

    // Start a new L2 block
    fn start_new_l2_block(&mut self, l2_block_env: L2BlockEnv);

    // Get the current state of the virtual machine
    fn get_current_execution_state(&self) -> CurrentExecutionState;

    // Execute transaction with optional bytecode compression
    fn execute_transaction_with_bytecode_compression(
        &mut self,
        tx: Transaction,
        with_compression: bool,
    ) -> (
        Result<(), BytecodeCompressionError>,
        VmExecutionResultAndLogs,
    );

    // Record VM memory metrics
    fn record_vm_memory_metrics(&self) -> VmMemoryMetrics;

    // How much gas is left in the current stack frame
    fn gas_remaining(&self) -> u32;

    // Finish batch and return the result
    fn finish_batch(&mut self) -> FinishedL1Batch;
}
```

### VM Factory

The VM Factory trait is responsible for creating new VM instances:

```rust
pub trait VmFactory<S>: VmInterface {
    // Creates a new VM instance
    fn new(batch_env: L1BatchEnv, system_env: SystemEnv, storage: StoragePtr<S>) -> Self;
}
```

### VM History Mode

The VM also supports history mode, which allows for creating snapshots of the VM state and rolling back to previous states:

```rust
pub trait VmInterfaceHistoryEnabled: VmInterface {
    // Create a snapshot of the current VM state
    fn make_snapshot(&mut self);

    // Roll back VM state to the latest snapshot
    fn rollback_to_the_latest_snapshot(&mut self);

    // Pop the latest snapshot from memory
    fn pop_snapshot_no_rollback(&mut self);
}
```

## VM Implementation Details

The VM implementation is located in `core/lib/multivm/src/versions/vm_latest/vm.rs`. The latest VM version is based on the zkEVM architecture and includes:

```rust
pub struct Vm<S: WriteStorage, H: HistoryMode> {
    pub(crate) bootloader_state: BootloaderState,
    // Current state and oracles of virtual machine
    pub(crate) state: ZkSyncVmState<S, H::Vm1_5_0>,
    pub(crate) storage: StoragePtr<S>,
    pub(crate) system_env: SystemEnv,
    pub(crate) batch_env: L1BatchEnv,
    // Snapshots for the current run
    pub(crate) snapshots: Vec<VmSnapshot>,
    pub(crate) subversion: MultiVMSubversion,
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
- The VM subversion, which indicates the specific version of the VM

### Core VM Logic

The core VM logic is based on the zkEVM architecture, which is a specialized virtual machine designed for generating zero-knowledge proofs of execution. The VM executes bytecode in a deterministic manner, tracking all operations and state changes to produce an execution trace that can be used to generate a proof.

The VM operates on a stack-based architecture similar to the Ethereum Virtual Machine (EVM), but with modifications to make it more amenable to zero-knowledge proof generation. It supports standard EVM opcodes as well as additional opcodes specific to the zkEVM.

### Execution Modes

The VM supports three execution modes:

```rust
pub enum VmExecutionMode {
    // Stop after executing the next transaction
    OneTx,
    // Stop after executing the entire batch
    Batch,
    // Stop after executing the entire bootloader
    Bootloader,
}
```

These modes determine when the VM execution should stop and what tracers should be used during execution.

### Gas Calculation

Gas is calculated and enforced during execution to limit the computational resources used by transactions. The VM tracks gas usage for each operation and ensures that transactions do not exceed their gas limit.

Gas calculation is implemented in `core/lib/multivm/src/versions/vm_latest/implementation/gas.rs` and includes:
- Computational gas for CPU operations
- Storage gas for storage operations
- Intrinsic gas for transaction overhead

### State Interaction

The VM interacts with the state through the storage interface, which provides methods for reading and writing storage slots. The storage interface is defined in `core/lib/vm_interface/src/storage/` and includes:

```rust
pub trait ReadStorage {
    // Read a storage slot
    fn read_value(&mut self, key: &StorageKey) -> StorageValue;
    
    // Get the number of storage reads
    fn get_enumeration_index(&self) -> u64;
}

pub trait WriteStorage: ReadStorage {
    // Start a new storage batch
    fn start_transaction(&mut self);
    
    // Commit the current storage batch
    fn commit_transaction(&mut self) -> bool;
    
    // Rollback the current storage batch
    fn rollback_transaction(&mut self) -> bool;
    
    // Write a value to a storage slot
    fn write_value(&mut self, key: &StorageKey, value: StorageValue);
}
```

The VM produces storage logs for each storage operation, which are then used to update the state in the State Keeper.

## Execution Process

The execution process is implemented in `core/lib/multivm/src/versions/vm_latest/implementation/execution.rs` and includes:

```rust
impl<S: WriteStorage, H: HistoryMode> Vm<S, H> {
    pub(crate) fn inspect_inner(
        &mut self,
        dispatcher: TracerDispatcher<S, H::Vm1_5_0>,
        execution_mode: VmExecutionMode,
        custom_pubdata_tracer: Option<PubdataTracer<S>>,
    ) -> VmExecutionResultAndLogs {
        // Implementation details
    }

    fn inspect_and_collect_results(
        &mut self,
        dispatcher: TracerDispatcher<S, H::Vm1_5_0>,
        execution_mode: VmExecutionMode,
        with_refund_tracer: bool,
        custom_pubdata_tracer: Option<PubdataTracer<S>>,
    ) -> (VmExecutionStopReason, VmExecutionResultAndLogs) {
        // Implementation details
    }

    fn execute_with_default_tracer(
        &mut self,
        tracer: &mut DefaultExecutionTracer<S, H::Vm1_5_0>,
    ) -> VmExecutionStopReason {
        // Implementation details
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

The bootloader is written in Yul and is loaded into memory at a specific address (`BOOTLOADER_ADDRESS = 0x8001`). It has a special heap that acts as an interface between the server and the zkEVM. The server fills the bootloader's heap with transaction data, and the bootloader executes these transactions.

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

The VM versions are defined in `core/lib/multivm/src/versions/` and include:
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

The Bitcoin-specific VM version is indicated by the `VmBitcoin1_0_0` variant in the `VmVersion` enum:

```rust
impl TryFrom<VmVersion> for MultiVMSubversion {
    type Error = VmVersionIsNotVm150Error;
    fn try_from(value: VmVersion) -> Result<Self, Self::Error> {
        match value {
            VmVersion::Vm1_5_0SmallBootloaderMemory => Ok(Self::SmallBootloaderMemory),
            VmVersion::Vm1_5_0IncreasedBootloaderMemory => Ok(Self::IncreasedBootloaderMemory),
            VmVersion::VmBitcoin1_0_0 => Ok(Self::IncreasedBootloaderMemory),
            _ => Err(VmVersionIsNotVm150Error),
        }
    }
}
```

This indicates that the Bitcoin VM version is based on the VM 1.5.0 with increased bootloader memory.

## Interactions with Other Components

### Interaction with State Keeper

The VM interacts with the State Keeper, which is responsible for maintaining the state of the rollup. The State Keeper processes transactions, executes them using the VM, and updates the state accordingly.

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
pub struct VmExecutionResultAndLogs {
    pub result: ExecutionResult,
    pub logs: VmExecutionLogs,
    pub statistics: VmExecutionStatistics,
    pub refunds: Refunds,
}
```

This structure contains the result of VM execution, including:
- The execution result (success, revert, or halt)
- Execution logs (storage logs, events, L2-to-L1 logs)
- Execution statistics (gas used, cycles used, etc.)
- Refunds (gas refunded, operator suggested refund)

### ExecutionResult

```rust
pub enum ExecutionResult {
    // Returned successfully
    Success { output: Vec<u8> },
    // Reverted by contract
    Revert { output: VmRevertReason },
    // Reverted for various reasons
    Halt { reason: Halt },
}
```

This enum represents the result of execution, which can be:
- Success: The execution completed successfully
- Revert: The execution was reverted by the contract
- Halt: The execution was halted for various reasons

### VmExecutionLogs

```rust
pub struct VmExecutionLogs {
    pub storage_logs: Vec<StorageLogWithPreviousValue>,
    pub events: Vec<VmEvent>,
    pub user_l2_to_l1_logs: Vec<UserL2ToL1Log>,
    pub system_l2_to_l1_logs: Vec<SystemL2ToL1Log>,
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
pub struct CurrentExecutionState {
    pub events: Vec<VmEvent>,
    pub deduplicated_storage_logs: Vec<StorageLog>,
    pub used_contract_hashes: HashSet<H256>,
    pub user_l2_to_l1_logs: Vec<UserL2ToL1Log>,
    pub system_logs: Vec<SystemL2ToL1Log>,
    pub storage_refunds: HashMap<Address, u32>,
    pub pubdata_costs: HashMap<Address, u32>,
}
```

This structure represents the current state of execution, including:
- Events emitted by contracts
- Deduplicated storage logs
- Used contract hashes
- L2-to-L1 logs
- Storage refunds
- Pubdata costs

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
  - `etc/multivm_bootloaders/`: Bootloader implementations for different VM versions
  - `etc/multivm_bootloaders/vm_bitcoin/`: Bitcoin-specific bootloader implementation

### Key Functions and Methods

- **VM Creation**:
  - `Vm::new`: Creates a new VM instance
  - `Vm::new_with_subversion`: Creates a new VM instance with a specific subversion

- **Transaction Execution**:
  - `Vm::push_transaction`: Pushes a transaction to the bootloader memory
  - `Vm::execute`: Executes the VM with a specific execution mode
  - `Vm::inspect`: Executes the VM with custom tracers
  - `Vm::inspect_inner`: Internal implementation of VM execution
  - `Vm::execute_with_default_tracer`: Executes the VM with the default tracer

- **State Management**:
  - `Vm::get_current_execution_state`: Gets the current execution state
  - `Vm::make_snapshot`: Creates a snapshot of the VM state
  - `Vm::rollback_to_the_latest_snapshot`: Rolls back to the latest snapshot
  - `Vm::finish_batch`: Finishes the current batch and returns the result

## Conclusion

The Execution Environment (VM) in the Via L2 Bitcoin ZK-Rollup is a sophisticated component that provides a deterministic environment for executing transactions and calculating state transitions. It is based on the zkEVM architecture and supports multiple VM versions through the multiVM architecture.

The VM executes transactions through the bootloader, which processes transactions one by one, validating them, executing them, and updating the state accordingly. The VM produces execution traces that are used by the Prover to generate zero-knowledge proofs, ensuring the validity of state transitions without revealing the details of the execution.

The VM interacts with various components of the system, including the State Keeper, System Contracts, and Prover, to provide a secure and efficient execution environment for the Via L2 Bitcoin ZK-Rollup.