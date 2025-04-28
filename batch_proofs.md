# Via L2 Bitcoin ZK-Rollup: Batch Proofs Aggregation

## 1. Introduction

Batch proof aggregation is a critical component of the Via L2 Bitcoin ZK-Rollup system, enabling the efficient verification of multiple transactions on the L1 (Bitcoin) chain. This document provides a detailed explanation of how the system aggregates multiple transaction proofs into a single batch proof that can be efficiently verified on L1.

The batch proof aggregation process is designed to compress thousands of individual transaction proofs into a single, compact proof that can be efficiently verified on Bitcoin. This is achieved through a multi-stage recursive proof aggregation process that progressively combines proofs while maintaining the cryptographic guarantees of the underlying zero-knowledge proof system.

## 2. Batch Proof Aggregation Architecture

The batch proof aggregation system follows a hierarchical architecture with multiple stages of aggregation:

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Basic Circuits │────►│ Leaf Aggregation│────►│ Node Aggregation│
│  (Round 0)      │     │ (Round 1)       │     │ (Round 2)       │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                                                        │
                                                        ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Final Proof    │◄────│ Scheduler       │◄────│ Recursion Tip   │
│  (Compressed)   │     │ (Round 4)       │     │ (Round 3)       │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

The system is implemented as a pipeline of specialized components:

1. **Witness Generators**: Generate witness data for each aggregation stage
2. **Circuit Provers**: Generate proofs for each circuit using the witness data
3. **Proof Compressor**: Compresses the final proof for submission to L1 (Bitcoin)

## 3. Aggregation Stages and Data Flow

### 3.1 Basic Circuits (Round 0)

The first stage of the aggregation process involves generating proofs for the basic circuits that represent individual transactions and state transitions.

**Input**: 
- `WitnessInputData` containing VM execution data, Merkle paths, and batch metadata
- Protocol version information

**Process**:
- The system generates up to 2400 basic circuit proofs
- Each proof corresponds to a specific circuit type (e.g., Main VM, RAM Permutation, Storage Application)
- Proofs are generated in parallel to maximize throughput

**Output**:
- `ZkSyncBaseLayerProof` objects for each basic circuit

**Code Reference**:
- The basic circuit witness generation is implemented in `prover/crates/bin/witness_generator/src/basic_circuits.rs`

### 3.2 Leaf Aggregation (Round 1)

The leaf aggregation stage combines multiple basic circuit proofs into a smaller number of leaf aggregation proofs.

**Input**:
- `ZkSyncBaseLayerProof` objects from the basic circuits stage
- Closed-form inputs for each circuit type

**Process**:
```rust
// From prover/crates/bin/witness_generator/src/leaf_aggregation.rs
pub async fn process_leaf_aggregation_job(
    started_at: Instant,
    job: LeafAggregationWitnessGeneratorJob,
    object_store: Arc<dyn ObjectStore>,
    max_circuits_in_flight: usize,
) -> LeafAggregationArtifacts {
    let circuit_id = job.circuit_id;
    let queues = split_recursion_queue(job.closed_form_inputs.1);
    
    // Process each queue in parallel
    let semaphore = Arc::new(Semaphore::new(max_circuits_in_flight));
    
    for (circuit_idx, (queue, proofs_ids_for_queue)) in
        queues.into_iter().zip(proofs_ids).enumerate()
    {
        // Load proofs for this queue
        let proofs = load_proofs_for_job_ids(&proofs_ids_for_queue, &*object_store).await;
        
        // Create leaf witness by aggregating base proofs
        let (_, circuit) = create_leaf_witness(
            circuit_id.into(),
            queue,
            base_proofs,
            &base_vk,
            &leaf_params,
        );
        
        // Save the resulting circuit for the next stage
        save_recursive_layer_prover_input_artifacts(
            job.block_number,
            circuit_idx,
            vec![circuit],
            AggregationRound::LeafAggregation,
            0,
            &*object_store,
            None,
        )
        .await
    }
}
```

**Key Aggregation Logic**:
- The system splits the recursion queue for each circuit type
- For each queue, it loads the corresponding base proofs
- It then creates a leaf witness by aggregating the base proofs
- The aggregation is performed using the `create_leaf_witness` function, which combines multiple base proofs into a single leaf circuit
- The resulting leaf circuits are saved for the next stage

**Output**:
- Up to 48 `ZkSyncRecursiveLayerCircuit` objects (leaf aggregation circuits)
- Aggregation metadata for the next stage

### 3.3 Node Aggregation (Round 2)

The node aggregation stage combines leaf aggregation proofs into node aggregation proofs, further reducing the number of proofs.

**Input**:
- `ZkSyncRecursionLayerProof` objects from the leaf aggregation stage
- Aggregation metadata from the previous stage

**Process**:
```rust
// From prover/crates/bin/witness_generator/src/node_aggregation.rs
pub async fn process_job_impl(
    job: NodeAggregationWitnessGeneratorJob,
    started_at: Instant,
    object_store: Arc<dyn ObjectStore>,
    max_circuits_in_flight: usize,
) -> NodeAggregationArtifacts {
    let node_vk_commitment = compute_node_vk_commitment(job.node_vk.clone());
    
    // Process chunks of aggregations in parallel
    for (circuit_idx, (chunk, proofs_ids_for_chunk)) in job
        .aggregations
        .chunks(RECURSION_ARITY)
        .zip(proofs_ids)
        .enumerate()
    {
        // Load proofs for this chunk
        let proofs = load_proofs_for_job_ids(&proofs_ids_for_chunk, &*object_store).await;
        
        // Create node witness by aggregating recursive proofs
        let (result_circuit_id, recursive_circuit, input_queue) = create_node_witness(
            &chunk,
            recursive_proofs,
            &vk,
            node_vk_commitment,
            &all_leafs_layer_params,
        );
        
        // Save the resulting circuit for the next stage
        save_recursive_layer_prover_input_artifacts(
            job.block_number,
            circuit_idx,
            vec![recursive_circuit],
            AggregationRound::NodeAggregation,
            job.depth + 1,
            &*object_store,
            Some(job.circuit_id),
        )
        .await
    }
}
```

**Key Aggregation Logic**:
- The system processes chunks of aggregations, where each chunk contains up to `RECURSION_ARITY` proofs
- For each chunk, it loads the corresponding recursive proofs
- It then creates a node witness by aggregating the recursive proofs
- The aggregation is performed using the `create_node_witness` function, which combines multiple recursive proofs into a single node circuit
- The resulting node circuits are saved for the next stage
- The process can be recursive, with multiple depths of node aggregation, until only a small number of proofs remain

**Output**:
- A smaller number of `ZkSyncRecursiveLayerCircuit` objects (node aggregation circuits)
- Aggregation metadata for the next stage

### 3.4 Recursion Tip (Round 3)

The recursion tip stage further aggregates node aggregation proofs into a single recursion tip proof.

**Input**:
- `ZkSyncRecursionLayerProof` objects from the node aggregation stage
- Verification keys for the node layer

**Process**:
```rust
// From prover/crates/bin/witness_generator/src/recursion_tip.rs
pub fn process_job_sync(
    job: RecursionTipWitnessGeneratorJob,
    started_at: Instant,
) -> RecursionTipArtifacts {
    let config = RecursionTipConfig {
        proof_config: recursion_layer_proof_config(),
        vk_fixed_parameters: job.node_vk.clone().into_inner().fixed_parameters,
        _marker: std::marker::PhantomData,
    };
    
    // Create recursion tip circuit
    let recursive_tip_circuit = RecursionTipCircuit {
        witness: job.recursion_tip_witness,
        config,
        transcript_params: (),
        _marker: std::marker::PhantomData,
    };
    
    RecursionTipArtifacts {
        recursion_tip_circuit: ZkSyncRecursiveLayerCircuit::RecursionTipCircuit(
            recursive_tip_circuit,
        ),
    }
}
```

**Key Aggregation Logic**:
- The system loads all final node proofs for the batch
- It creates a recursion tip witness that includes all the node proofs
- The recursion tip circuit combines all the node proofs into a single proof
- The recursion tip circuit has a fixed arity (`RECURSION_TIP_ARITY`), which limits the number of proofs it can combine

**Output**:
- A single `ZkSyncRecursiveLayerCircuit` object (recursion tip circuit)

### 3.5 Scheduler (Round 4)

The scheduler stage produces the final aggregated proof that will be submitted to L1 (Bitcoin).

**Input**:
- `ZkSyncRecursionLayerProof` object from the recursion tip stage
- Verification keys for the recursion tip layer

**Process**:
```rust
// From prover/crates/bin/witness_generator/src/scheduler.rs
pub fn process_job_sync(
    job: SchedulerWitnessGeneratorJob,
    started_at: Instant,
) -> SchedulerArtifacts {
    let config = SchedulerConfig {
        proof_config: recursion_layer_proof_config(),
        vk_fixed_parameters: job.recursion_tip_vk.clone().into_inner().fixed_parameters,
        capacity: SCHEDULER_CAPACITY,
        _marker: std::marker::PhantomData,
        recursion_tip_vk: job.recursion_tip_vk.into_inner(),
        node_layer_vk: job.node_vk.into_inner(),
        leaf_layer_parameters: job.leaf_layer_parameters,
    };
    
    // Create scheduler circuit
    let scheduler_circuit = SchedulerCircuit {
        witness: job.scheduler_witness,
        config,
        transcript_params: (),
        _marker: std::marker::PhantomData,
    };
    
    SchedulerArtifacts {
        scheduler_circuit: ZkSyncRecursiveLayerCircuit::SchedulerCircuit(scheduler_circuit),
    }
}
```

**Key Aggregation Logic**:
- The system loads the recursion tip proof
- It creates a scheduler witness that includes the recursion tip proof
- The scheduler circuit produces the final proof that will be submitted to L1
- The scheduler circuit also includes verification keys for all previous layers, allowing it to verify the entire proof chain

**Output**:
- A single `ZkSyncRecursiveLayerCircuit` object (scheduler circuit)

### 3.6 Proof Compression

The final stage compresses the scheduler proof into a format that can be efficiently verified on L1 (Bitcoin).

**Input**:
- `ZkSyncRecursionLayerProof` object from the scheduler stage

**Process**:
```rust
// From prover/crates/bin/proof_fri_compressor/src/compressor.rs
pub fn compress_proof(
    l1_batch: L1BatchNumber,
    proof: ZkSyncRecursionLayerProof,
    _compression_mode: u8,
    setup_data_path: String,
) -> anyhow::Result<FinalProof> {
    let keystore = Keystore::new_with_setup_data_path(setup_data_path);
    let scheduler_vk = keystore
        .load_recursive_layer_verification_key(
            ZkSyncRecursionLayerStorageType::SchedulerCircuit as u8,
        )
        .context("get_recursiver_layer_vk_for_circuit_type()")?;
    
    // Compress the proof
    #[cfg(feature = "gpu")]
    let wrapper_proof = {
        let crs = get_trusted_setup();
        let wrapper_config = DEFAULT_WRAPPER_CONFIG;
        let mut prover = WrapperProver::<GPUWrapperConfigs>::new(&crs, wrapper_config).unwrap();
        
        prover
            .generate_setup_data(scheduler_vk.into_inner())
            .unwrap();
        prover.generate_proofs(proof.into_inner()).unwrap();
        
        prover.get_wrapper_proof().unwrap()
    };
    
    // Convert to final proof format
    let serialized = bincode::serialize(&wrapper_proof)
        .expect("Failed to serialize proof with ZkSyncSnarkWrapperCircuit");
    let final_proof: FinalProof =
        bincode::deserialize(&serialized).expect("Failed to deserialize final proof");
    
    Ok(final_proof)
}
```

**Key Compression Logic**:
- The system loads the scheduler verification key
- It uses a wrapper prover to compress the proof
- The compression process converts the FRI proof to a Bellman proof, which is more compact
- The compressed proof is serialized and deserialized into the final proof format

**Output**:
- A `FinalProof` object that can be submitted to L1 (Bitcoin)

### 3.7 Final Batch Proof

The final batch proof is assembled by combining the compressed scheduler proof with aggregation result coordinates.

```rust
// From prover/crates/bin/proof_fri_compressor/src/compressor.rs
let l1_batch_proof = L1BatchProofForL1 {
    aggregation_result_coords,
    scheduler_proof: artifacts,
    protocol_version: self.protocol_version,
};
```

**Final Proof Structure**:
```rust
// From core/lib/prover_interface/src/outputs.rs
pub struct L1BatchProofForL1 {
    pub aggregation_result_coords: [[u8; 32]; 4],
    pub scheduler_proof: FinalProof,
    pub protocol_version: ProtocolSemanticVersion,
}
```

### 3.8 Previous Block Selection for Proof Generation

When generating proofs, the system needs to reference the previous batch to establish continuity in the state transitions. The Data Availability Dispatcher is responsible for loading the previous batch data:

```rust
// From core/node/via_da_dispatcher/src/da_dispatcher.rs
async fn load_real_proof_operation(&self, batch_to_prove: L1BatchNumber) -> Option<Vec<u8>> {
    let mut storage = self.pool.connection_tagged("da_dispatcher").await.ok()?;

    // Use the immediately preceding batch number
    let previous_batch_number = batch_to_prove - 1;

    // Load metadata for the previous batch
    let previous_proven_batch_metadata = match storage
        .blocks_dal()
        .get_l1_batch_metadata(previous_batch_number)
        .await
    {
        Ok(Some(metadata)) => metadata,
        Ok(None) => {
            tracing::error!(
                "L1 batch #{} is not complete in the DB",
                previous_batch_number
            );
            return None;
        }
        Err(e) => {
            tracing::error!(
                "Failed to retrieve L1 batch #{} metadata: {}",
                previous_batch_number,
                e
            );
            return None;
        }
    };

    // Continue with proof generation using the previous batch metadata
    // ...
}
```

This approach ensures that:
1. Proofs are generated in sequence without gaps
2. Each proof properly references the state from the immediately preceding batch
3. The chain of proofs maintains continuity, which is essential for verifying the integrity of state transitions

## 4. Key Data Structures

### 4.1 Proof Wrappers

The system uses several wrapper types to handle different stages of the proving process:

```rust
// From prover/crates/lib/prover_fri_types/src/lib.rs
pub enum CircuitWrapper {
    Base(ZkSyncBaseLayerCircuit),
    Recursive(ZkSyncRecursiveLayerCircuit),
    BasePartial((ZkSyncBaseLayerCircuit, CircuitAuxData)),
}

pub enum FriProofWrapper {
    Base(ZkSyncBaseLayerProof),
    Recursive(ZkSyncRecursionLayerProof),
}
```

These wrappers allow the system to handle different types of circuits and proofs in a unified way.

### 4.2 Base Layer Proofs

Base layer proofs are generated for individual circuits that represent specific parts of the transaction execution:

```rust
// From circuit_definitions/circuit_definitions/base_layer.rs
pub struct ZkSyncBaseLayerProof {
    pub circuit_type: u8,
    pub proof: Proof<Bn256, ZkSyncBaseLayerCircuit>,
}
```

### 4.3 Recursion Layer Proofs

Recursion layer proofs are generated by aggregating lower-level proofs:

```rust
// From circuit_definitions/circuit_definitions/recursion_layer.rs
pub struct ZkSyncRecursionLayerProof {
    pub circuit_type: u8,
    pub proof: Proof<Bn256, ZkSyncRecursiveLayerCircuit>,
}
```

### 4.4 Final Proof

The final proof is a compressed version of the scheduler proof that can be efficiently verified on L1:

```rust
// From circuit_sequencer_api/proof.rs
pub struct FinalProof {
    // Compressed proof data
}
```

### 4.5 L1 Batch Proof

The L1 batch proof combines the final proof with aggregation result coordinates:

```rust
// From core/lib/prover_interface/src/outputs.rs
pub struct L1BatchProofForL1 {
    pub aggregation_result_coords: [[u8; 32]; 4],
    pub scheduler_proof: FinalProof,
    pub protocol_version: ProtocolSemanticVersion,
}
```

## 5. Aggregation Algorithms and Criteria

### 5.1 Proof Selection Criteria

The system uses several criteria to determine which proofs are included in each aggregation step:

1. **Circuit Type**: Proofs are grouped by circuit type in the leaf aggregation stage
2. **Recursion Arity**: The number of proofs that can be aggregated in a single step is limited by the recursion arity
3. **Depth**: Node aggregation can occur at multiple depths, with each depth further reducing the number of proofs

### 5.2 Leaf Aggregation Algorithm

The leaf aggregation algorithm combines multiple base proofs of the same circuit type:

1. Split the recursion queue for each circuit type
2. For each queue, load the corresponding base proofs
3. Create a leaf witness by aggregating the base proofs using the `create_leaf_witness` function
4. The aggregation is performed in parallel for different queues

### 5.3 Node Aggregation Algorithm

The node aggregation algorithm combines multiple leaf proofs:

1. Process chunks of aggregations, where each chunk contains up to `RECURSION_ARITY` proofs
2. For each chunk, load the corresponding recursive proofs
3. Create a node witness by aggregating the recursive proofs using the `create_node_witness` function
4. The aggregation is performed in parallel for different chunks
5. The process can be recursive, with multiple depths of node aggregation

### 5.4 Recursion Tip Algorithm

The recursion tip algorithm combines all final node proofs:

1. Load all final node proofs for the batch
2. Create a recursion tip witness that includes all the node proofs
3. The recursion tip circuit combines all the node proofs into a single proof

### 5.5 Scheduler Algorithm

The scheduler algorithm produces the final proof:

1. Load the recursion tip proof
2. Create a scheduler witness that includes the recursion tip proof
3. The scheduler circuit produces the final proof that will be submitted to L1

## 6. Performance Considerations and Optimizations

### 6.1 Parallelization

The batch proof aggregation process is heavily parallelized to maximize throughput:

1. **Parallel Proof Generation**: Basic circuit proofs are generated in parallel
2. **Parallel Aggregation**: Leaf and node aggregation is performed in parallel for different queues/chunks
3. **Semaphore-Based Concurrency Control**: The system uses semaphores to limit the number of concurrent operations

```rust
// From prover/crates/bin/witness_generator/src/leaf_aggregation.rs
let semaphore = Arc::new(Semaphore::new(max_circuits_in_flight));
```

### 6.2 GPU Acceleration

The system uses GPU acceleration for proof generation and compression:

```rust
// From prover/crates/bin/proof_fri_compressor/src/compressor.rs
#[cfg(feature = "gpu")]
let wrapper_proof = {
    let crs = get_trusted_setup();
    let wrapper_config = DEFAULT_WRAPPER_CONFIG;
    let mut prover = WrapperProver::<GPUWrapperConfigs>::new(&crs, wrapper_config).unwrap();
    
    prover
        .generate_setup_data(scheduler_vk.into_inner())
        .unwrap();
    prover.generate_proofs(proof.into_inner()).unwrap();
    
    prover.get_wrapper_proof().unwrap()
};
```

### 6.3 Caching

The system caches verification keys and setup data to avoid redundant computation:

```rust
// From prover/crates/bin/prover_fri/src/prover_job_processor.rs
fn get_setup_data(
    &self,
    key: ProverServiceDataKey,
) -> anyhow::Result<Arc<GoldilocksProverSetupData>> {
    let key = get_setup_data_key(key);
    Ok(match &self.setup_load_mode {
        SetupLoadMode::FromMemory(cache) => cache
            .get(&key)
            .context("Setup data not found in cache")?
            .clone(),
        SetupLoadMode::FromDisk => {
            // Load from disk if not in cache
        }
    })
}
```

### 6.4 Job Queue Management

The system uses a job queue to manage the proof generation and aggregation process:

1. Jobs are scheduled in the database
2. Workers poll for available jobs
3. Jobs are processed in parallel
4. Results are saved back to the database

This allows the system to scale horizontally by adding more workers.

## 7. Conclusion

The batch proof aggregation process in the Via L2 Bitcoin ZK-Rollup system is a sophisticated multi-stage process that efficiently combines thousands of individual transaction proofs into a single, compact proof that can be verified on Bitcoin. The process is designed to be highly parallel and scalable, with multiple optimizations to maximize throughput and minimize latency.

The key components of the batch proof aggregation process are:

1. **Basic Circuits**: Generate proofs for individual transactions and state transitions
2. **Leaf Aggregation**: Combine basic circuit proofs into leaf aggregation proofs
3. **Node Aggregation**: Combine leaf aggregation proofs into node aggregation proofs
4. **Recursion Tip**: Further aggregate node aggregation proofs into a single recursion tip proof
5. **Scheduler**: Produce the final aggregated proof
6. **Proof Compression**: Compress the final proof for submission to L1 (Bitcoin)

This hierarchical approach allows the system to achieve high throughput while maintaining the security guarantees of the underlying zero-knowledge proof system.

Recent improvements to the proof generation process, such as the optimized previous block selection, have further enhanced the reliability and efficiency of the system. By ensuring that proofs are generated in the correct sequence without gaps (using the immediately preceding batch as the previous batch), the system maintains the continuity of state transitions, which is essential for the integrity of the rollup.