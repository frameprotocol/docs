# FRAME Deterministic Distributed Execution (DDE)

FRAME's Deterministic Distributed Execution (DDE) system enables verifiable distributed compute where nodes can delegate computation, execute tasks remotely, and verify results cryptographically without trusting remote machines.

## Overview

DDE transforms FRAME from a single-node runtime into a **verifiable distributed compute network**. This enables:

- **Distributed AI agents** - Agents can execute across multiple nodes
- **Distributed computation markets** - Nodes can sell compute capacity
- **Low-power device delegation** - Lightweight devices can delegate heavy compute
- **Trustless execution verification** - Results verified cryptographically
- **Peer-to-peer compute networks** - Nodes form compute clusters

## Core Concepts

### ComputeTask

A compute task represents a request for distributed execution.

**Structure:**
```javascript
{
  taskVersion: 1,                    // Task format version
  taskId: string,                    // Unique task identifier (SHA-256)
  intent: {                          // Intent to execute
    action: string,
    payload: object,
    raw: string,
    timestamp: number
  },
  dappId: string,                    // dApp identifier
  dappCodeHash: string,              // SHA-256 of dApp code (for verification)
  requiredCapabilities: string[],    // Capabilities needed
  inputHash: string,                 // SHA-256 of input payload
  requesterPublicKey: string,         // Public key of requesting node
  timestamp: number,                  // Task creation timestamp
  maxExecutionSteps: number,        // Maximum trace length (default: 500)
  maxExecutionTime: number            // Maximum execution time in ms (optional)
}
```

**Code Hash Requirements:**
- `dappCodeHash = SHA256(canonicalized source)` - dApp code referenced by hash
- Execution proofs are invalid if code hash mismatches
- Ensures remote nodes cannot run modified code

The task references code by hash, not by name, guaranteeing deterministic execution.

## API Reference

### FRAME.createComputeTask(intent)

Creates a compute task for distributed execution.

**Process:**
1. Resolve intent to dApp
2. Get dApp code hash
3. Extract required capabilities
4. Compute input hash
5. Generate task ID

**Returns:**
```javascript
{
  taskId: string,
  task: ComputeTask
}
```

**Example:**
```javascript
var taskData = await FRAME.createComputeTask({
  action: "wallet.send",
  payload: { recipient: "Alice", amount: 5 },
  raw: "Send 5 frame to Alice",
  timestamp: Date.now()
});
```

### FRAME.executeComputeTask(task)

Executes a remote compute task.

**Process:**
1. Verify dApp code hash matches
2. Execute intent via `handleStructuredIntent`
3. Get receipt hash from result
4. Generate execution proof
5. Return proof and metadata

**Returns:**
```javascript
{
  taskId: string,
  proof: ExecutionProof,
  resultHash: string,
  executorPublicKey: string,
  executionLatency: number
}
```

**Example:**
```javascript
var result = await FRAME.executeComputeTask(task);
// result.proof can be verified by requester
```

### FRAME.verifyRemoteExecution(proof)

Verifies a remote execution proof.

**Process:**
1. Verify execution proof (`verifyExecutionProof`)
2. Verify receipt signature
3. Verify state root validity

**Returns:** `true` if valid, `false` otherwise

**Example:**
```javascript
var valid = await FRAME.verifyRemoteExecution(proof);
if (valid) {
  // Accept result
}
```

### FRAME.commitVerifiedExecution(proof)

Commits a verified remote execution to local chain.

**Remote Execution Commit Rules:**

1. **Verify execution proof** - `verifyRemoteExecution(proof)` must return `true`
2. **Verify receipt integrity** - Receipt signature and chain link must be valid
3. **Verify capability scope** - `capabilitiesUsed` must be subset of `capabilitiesDeclared`
4. **Append remote execution receipt** - Create receipt with updated chain links:
   - `previousStateRoot` = local chain's last `nextStateRoot`
   - `previousReceiptHash` = local chain's last `receiptHash`
   - Preserve original signature and proof data
5. **Update state root** - Recompute state root after append

**Critical:** Remote execution does not modify local state until proof verification succeeds. This prevents malicious nodes from mutating state.

**Returns:** `true` on success

**Example:**
```javascript
var valid = await FRAME.verifyRemoteExecution(proof);
if (valid) {
  await FRAME.commitVerifiedExecution(proof);
  // Remote execution now in local chain
}
```

### FRAME.broadcastComputeTask(task)

Broadcasts a compute task to peers.

**Process:**
1. Create message: `{ type: 'compute_task', payload: task }`
2. Broadcast to all peers via peer router
3. Peers receive and optionally execute

**Returns:**
```javascript
{
  taskId: string,
  broadcast: true
}
```

## Message Protocol

### compute_task Message

**Type:** `compute_task`

**Payload:** ComputeTask object

**Handler:** `FRAME_PEER_ROUTER.handleComputeMessage()`

**Process:**
1. Receive task from peer
2. Optionally execute via `FRAME.executeComputeTask()`
3. Generate proof and result
4. Send `compute_result` message back

### compute_result Message

**Type:** `compute_result`

**Payload:**
```javascript
{
  taskId: string,
  proof: ExecutionProof,
  resultHash: string,
  executorPublicKey: string,
  executionLatency: number
}
```

**Handler:** `FRAME_PEER_ROUTER.handleComputeMessage()`

**Process:**
1. Receive result from peer
2. Verify proof via `FRAME.verifyRemoteExecution()`
3. Update peer reputation
4. Commit verified execution if valid

## Peer Reputation System

**File:** `ui/system/peerReputation.js`

Tracks peer reputation for distributed execution.

### Reputation Data

```javascript
{
  peerPublicKey: {
    successfulProofs: number,
    failedProofs: number,
    latencySum: number,
    latencyCount: number,
    latencyAvg: number,
    lastSeen: number
  }
}
```

### Reputation Tracking

- **Success** - `recordSuccess(peerPublicKey, latency)` - Increments successful proofs, updates latency average
- **Failure** - `recordFailure(peerPublicKey)` - Increments failed proofs
- **Blacklist** - `isBlacklisted(peerPublicKey)` - Returns true if failure rate > 50% (with at least 5 attempts)

### API

```javascript
window.FRAME_PEER_REPUTATION = {
  recordSuccess: function(peerPublicKey, latency),
  recordFailure: function(peerPublicKey),
  getReputation: function(peerPublicKey),
  isBlacklisted: function(peerPublicKey),
  getAllReputations: function(),
  clearReputation: function(peerPublicKey),
  clearAll: function()
}
```

## Execution Flow

### Step 1: Task Creation

```javascript
var taskData = await FRAME.createComputeTask(intent);
// taskData.taskId, taskData.task
```

### Step 2: Task Broadcast

```javascript
await FRAME.broadcastComputeTask(taskData.task);
// Task sent to all peers
```

### Step 3: Remote Execution

Peer receives task and executes:
```javascript
var result = await FRAME.executeComputeTask(task);
// Returns proof + metadata
```

### Step 4: Result Verification

Requester receives result and verifies:
```javascript
var valid = await FRAME.verifyRemoteExecution(result.proof);
if (valid) {
  await FRAME.commitVerifiedExecution(result.proof);
}
```

## Deterministic Safety Checks

Every remote execution must verify:

1. **dApp code hash** - Code matches expected hash
2. **Capability scope** - Only declared capabilities used
3. **Input hash** - Input matches expected hash
4. **Execution trace hash** - Trace matches proof
5. **State root transition** - State transition is valid

If any mismatch occurs → reject result

## Security Protections

### Task Validation and Resource Limits

**Execution Limits:**
- `maxExecutionSteps` - Maximum trace length (default: 500 steps)
- `maxExecutionTime` - Maximum execution time in ms (optional)
- `maxMemoryUsage` - Maximum memory usage (optional, if supported)
- `maxComputeTasksPerPeer` - Maximum concurrent tasks per peer (prevents DoS)

**Example:**
```javascript
if (trace.length > maxExecutionSteps) {
  abort execution; // Trace too long
}
if (executionTime > maxExecutionTime) {
  abort execution; // Execution timeout
}
```

### Reputation-Based Filtering

**Blacklist Threshold:**
- 3 invalid proofs → temporary ban (configurable)
- Failure rate > 50% with ≥5 attempts → blacklisted
- Blacklisted peers don't execute critical tasks
- Reputation decays over time (configurable decay rate)

**Reputation Rules:**
- Success increments `successfulProofs`
- Failure increments `failedProofs`
- Average latency tracked (`latencyAvg`)
- Last seen timestamp updated

## Storage of Remote Results

Remote results produce local receipts with updated chain links:

```javascript
{
  version: RECEIPT_VERSION,
  timestamp: receipt.timestamp,
  identity: receipt.identity,
  dappId: receipt.dappId,
  intent: receipt.intent.action,
  // ... other fields ...
  previousStateRoot: localChain[localChain.length - 1].nextStateRoot,
  previousReceiptHash: localChain[localChain.length - 1].receiptHash,
  // ... preserves original signature and proof data ...
}
```

This preserves chain integrity while incorporating remote execution results.

## Performance Optimization

### Speculative Parallel Execution

Multiple peers can execute the same task:
- Accept the first valid proof
- Discard others
- Reduces latency for critical tasks

### Reputation-Based Routing

- Route tasks to high-reputation peers first
- Fallback to lower-reputation peers if needed
- Optimize for latency and reliability

## Benefits

DDE enables FRAME to become:

- **Deterministic runtime** - Core execution model
- **Verifiable compute engine** - Cryptographic proofs
- **Distributed execution network** - Multi-node compute
- **Agent coordination platform** - Agent swarms across machines

## Related Documentation

- [Execution Proofs](execution_proofs.md) - Proof generation and verification
- [Receipt Chain](receipt_chain.md) - Receipt structure
- [Networking Model](networking_model.md) - Peer communication
- [Peer Router](peerRouter.js) - Message routing implementation
