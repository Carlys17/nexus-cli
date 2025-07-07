# Proof Generation dan Submission di Nexus CLI

## Overview
Nexus CLI menggunakan arsitektur worker yang terpisah untuk menangani proof generation (offline workers) dan proof submission (online workers), dengan orchestrator sebagai koordinator komunikasi dengan jaringan.

## 🏗️ Arsitektur Sistem

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Main CLI      │    │  Prover Runtime  │    │  Orchestrator   │
│                 │───▶│                  │───▶│     Client      │
│ - User Commands │    │ - Worker Mgmt    │    │ - Network API   │
│ - Config        │    │ - Task Queue     │    │ - HTTP Requests │
└─────────────────┘    └──────────────────┘    └─────────────────┘
                                │
                   ┌────────────┼────────────┐
                   │            │            │
              ┌─────────┐  ┌─────────┐  ┌─────────┐
              │ Offline │  │ Online  │  │  Task   │
              │Workers  │  │Workers  │  │ Cache   │
              │         │  │         │  │         │
              │- Prove  │  │- Fetch  │  │- Dups   │
              │- Compute│  │- Submit │  │- History│
              └─────────┘  └─────────┘  └─────────┘
```

## 📂 Komponen Utama

### 1. **Proof Generation** (`clients/cli/src/prover.rs`)

#### Fungsi Utama:
- **`prove_anonymously()`** - Anonymous proving dengan input hardcoded
- **`authenticated_proving()`** - Authenticated proving dengan task dari network

#### Contoh Proof Generation:
```rust
// Anonymous proving - Fibonacci sequence
let public_input: (u32, u32, u32) = (9, 1, 1); // F(9) = 55
let stwo_prover = get_initial_stwo_prover()?;
let (view, proof) = stwo_prover
    .prove_with_input::<(), (u32, u32, u32)>(&(), &public_input)
    .map_err(|e| ProverError::Stwo(format!("Failed to run prover: {}", e)))?;
```

#### Program Support:
- **`fast-fib`** - Fast Fibonacci dengan input string
- **`fib_input_initial`** - Fibonacci dengan triple input (n, init_a, init_b)

### 2. **Worker Management** (`clients/cli/src/prover_runtime.rs`)

#### Authenticated Workers Flow:
```rust
// 1. Start task fetcher
fetch_prover_tasks() // Online worker - mendapat task dari orchestrator

// 2. Start offline workers 
start_workers() // Offline workers - generate proofs

// 3. Start proof submitter
submit_proofs() // Online worker - submit results ke orchestrator
```

#### Anonymous Workers Flow:
```rust
// Continuous loop proving dengan hardcoded input
start_anonymous_workers() // Prove secara terus-menerus tanpa network
```

### 3. **Offline Workers** (`clients/cli/src/workers/offline.rs`)

#### Tugas:
- **Task Dispatch** - Round-robin distribution ke workers
- **Proof Computation** - Generate proofs menggunakan nexus-sdk
- **Error Handling** - Classify dan handle worker errors

#### Worker Implementation:
```rust
// Setiap worker running di tokio task terpisah
for worker_id in 0..num_workers {
    tokio::spawn(async move {
        loop {
            tokio::select! {
                Some(task) = task_receiver.recv() => {
                    match authenticated_proving(&task, &environment, client_id.clone()).await {
                        Ok(proof) => {
                            // Send hasil ke result channel
                            let _ = results_sender.send((task, proof)).await;
                        }
                        Err(e) => {
                            // Log error dengan level classification
                        }
                    }
                }
                _ = shutdown.recv() => break,
            }
        }
    });
}
```

### 4. **Online Workers** (`clients/cli/src/workers/online.rs`)

#### Task Fetching (`fetch_prover_tasks`):
- **Demand-driven**: Fetch tasks ketika queue < LOW_WATER_MARK
- **Batch processing**: Fetch multiple tasks sekaligus
- **Exponential backoff**: Handle rate limiting dan errors
- **Duplicate prevention**: TaskCache untuk avoid re-fetch

#### Proof Submission (`submit_proofs`):
```rust
// Main submission loop
loop {
    tokio::select! {
        Some((task, proof)) = results.recv() => {
            // 1. Check duplicates
            if !successful_tasks.contains(&task.task_id).await {
                // 2. Serialize proof
                let proof_bytes = postcard::to_allocvec(&proof)?;
                let proof_hash = format!("{:x}", Keccak256::digest(&proof_bytes));
                
                // 3. Submit ke orchestrator
                orchestrator.submit_proof(
                    &task.task_id,
                    &proof_hash,
                    proof_bytes,
                    signing_key.clone(),
                    num_workers,
                ).await?;
                
                // 4. Mark as successful
                successful_tasks.insert(task.task_id.clone()).await;
            }
        }
    }
}
```

### 5. **Orchestrator Client** (`clients/cli/src/orchestrator/client.rs`)

#### Network API Endpoints:
- **`GET /v3/tasks`** - Get existing assigned tasks
- **`POST /v3/tasks`** - Request new proof task
- **`POST /v3/tasks/submit`** - Submit completed proof
- **`POST /v3/users`** - Register user
- **`POST /v3/nodes`** - Register node

#### Proof Submission Details:
```rust
async fn submit_proof(
    &self,
    task_id: &str,
    proof_hash: &str,
    proof: Vec<u8>,
    signing_key: SigningKey,
    num_provers: usize,
) -> Result<(), OrchestratorError> {
    // 1. Create cryptographic signature
    let (signature, public_key) = self.create_signature(&signing_key, task_id, proof_hash);
    
    // 2. Collect telemetry data
    let (program_memory, total_memory) = get_memory_info();
    let flops = estimate_peak_gflops(num_provers);
    let location = self.get_country().await; // Privacy-preserving country detection
    
    // 3. Build request with telemetry
    let request = SubmitProofRequest {
        task_id: task_id.to_string(),
        node_type: NodeType::CliProver as i32,
        proof_hash: proof_hash.to_string(),
        proof,
        node_telemetry: Some(NodeTelemetry {
            flops_per_sec: Some(flops as i32),
            memory_used: Some(program_memory),
            memory_capacity: Some(total_memory),
            location: Some(location),
        }),
        ed25519_public_key: public_key,
        signature,
    };
    
    // 4. Send HTTP POST request
    self.post_request_no_response("v3/tasks/submit", request_bytes).await
}
```

## 🔄 Flow Lengkap: Task → Proof → Submission

### 1. **Task Acquisition**
```
fetch_prover_tasks() → orchestrator.get_proof_task() → Task
```

### 2. **Proof Generation**
```
Task → offline_worker → authenticated_proving() → Proof
```

### 3. **Proof Submission**
```
(Task, Proof) → submit_proofs() → orchestrator.submit_proof() → Success
```

## 🛠️ Konfigurasi Worker

### Worker Limits:
- **Min Workers**: 1
- **Max Workers**: 8 (rate limiting protection)
- **Queue Size**: Configurable (LOW_WATER_MARK untuk fetch trigger)

### Rate Limiting & Backoff:
- **Initial Backoff**: 30 seconds
- **Max Backoff**: 60 seconds
- **Rate Limit (429)**: Exponential backoff
- **Error Handling**: Different backoff strategies per error type

## 🔐 Security & Authentication

### Cryptographic Signing:
```rust
// Signature format
let signature_version = 0;
let msg = format!("{} | {} | {}", signature_version, task_id, proof_hash);
let signature = signing_key.sign(msg.as_bytes());
```

### Node Authentication:
- **Ed25519 Keys**: Generated per session
- **Verifying Key**: Sent dengan task requests
- **Signing Key**: Used untuk proof submission authentication

## 📊 Monitoring & Analytics

### Event Tracking:
- **Anonymous Proofs**: `cli_proof_anon_v3`
- **Authenticated Proofs**: `cli_proof_node_v3`
- **Performance Stats**: Tasks/minute, memory usage, FLOPS

### Telemetry Data:
- System performance (FLOPS, memory)
- Geographic location (country code only)
- Node statistics dan health metrics

## 🔧 Development & Testing

### Local Testing:
```bash
# Build dengan proto support
cargo build --features build_proto

# Run tests
cargo test --package nexus-network
```

### Mock Support:
- `MockOrchestrator` untuk testing
- Unit tests untuk individual components
- Integration tests untuk full flow

## ⚡ Performance Optimizations

1. **Parallel Processing**: Multiple workers per CPU core
2. **Async I/O**: Non-blocking network operations
3. **Connection Pooling**: Reuse HTTP connections
4. **Batch Operations**: Fetch multiple tasks sekaligus
5. **Memory Management**: Efficient proof serialization dengan postcard