# FRAME Verifiable Execution Proofs

FRAME's execution proof system enables cryptographically verifiable computation proofs. Every execution can produce a proof that allows any peer to independently verify the correctness of the computation without trusting the original runtime.

## Overview

Execution proofs transform FRAME from a logged execution system into a **verifiable compute runtime**. This enables:

- **Trustless federation** - Peers can verify remote execution results
- **Provable agent actions** - Agents can prove they executed tasks correctly
- **Verifiable AI execution** - AI-generated plans can be proven to produce specific results
- **Distributed deterministic compute** - Nodes can exchange proofs instead of recomputing

## Execution Trace Recording

During execution, FRAME records a minimal deterministic trace of operations:

**File:** `ui/runtime/engine.js` — Execution trace recorder

### Trace Structure

The trace captures **only operations that influence state**:

**Allowed operations:**
- `storage_read` - Storage read operations (key + value hash)
- `storage_write` - Storage write operations (key + value hash)
- `capability_invoke` - Capability invocations (capability + args hash)
- `deterministic_compute` - Deterministic computation steps (function + args hash + result hash)

**Prohibited operations:**
- Timestamps (nondeterministic)
- Random values (nondeterministic)
- Network responses (already sealed in receipt)
- System clock (nondeterministic)

**Example trace:**
```javascript
[
  { op: "storage_read", data: { key: "wallet:balance", valueHash: "abc123..." }, timestamp: 1234567890 },
  { op: "storage_write", data: { key: "wallet:balance", valueHash: "def456..." }, timestamp: 1234567891 },
  { op: "capability_invoke", data: { capability: "wallet.send", argsHash: "ghi789..." }, timestamp: 1234567892 }
]
```

**Trace Serialization Rules:**
- Canonical JSON format
- Sorted keys (alphabetical)
- UTF-8 encoding
- SHA-256 hash computation: `executionTraceHash = sha256(JSON.stringify(canonicalize(trace)))`

These rules ensure independent nodes compute identical hashes.

### Secret Sanitization and Trace Privacy

The trace recorder **never** includes:
- Private keys
- Raw secrets
- Capability tokens
- Passwords

**Trace Privacy Rules:**
- Trace values may be hashed (sensitive data replaced with hash)
- Trace may omit sensitive payloads entirely
- Trace must remain deterministic (same execution produces same trace)

Sensitive values are hashed before recording:
```javascript
// Secret detected in data
if (key.toLowerCase().indexOf('secret') !== -1) {
  sanitizedData[key + 'Hash'] = sha256(data[key]);
} else {
  sanitizedData[key] = data[key];
}
```

**Important:** Trace sanitization must preserve determinism. Same execution with same inputs must produce identical trace hash.

### Trace Recording Lifecycle

1. **Enable trace** - `enableExecutionTrace()` called before execution
2. **Record operations** - Storage and capability operations append to trace
3. **Compute hash** - `executionTraceHash = sha256(canonicalize(trace))`
4. **Include in receipt** - Hash stored in receipt (trace data not stored)
5. **Disable trace** - `disableExecutionTrace()` called after execution

## Receipt Integration

**File:** `ui/runtime/engine.js` — `buildReceipt(opts)`

The execution trace hash is computed and included in receipts:

```javascript
// Compute execution trace hash if trace was recorded
var executionTraceHash = null;
if (_traceEnabled && _executionTrace.length > 0) {
  var canonicalTrace = canonicalize(_executionTrace);
  executionTraceHash = await sha256(JSON.stringify(canonicalTrace));
}

var receipt = {
  // ... other fields ...
  executionTraceHash: executionTraceHash,  // Optional field
  // ... other fields ...
};
```

The `executionTraceHash` is included in the receipt signable payload when present.

## Proof Generation

**API:** `FRAME.generateExecutionProof(receiptHash)`

Generates a verifiable execution proof from a receipt.

**Process:**
1. Locate receipt by hash
2. Verify receipt exists and contains `executionTraceHash`
3. Get dApp code hash
4. Build proof object

**Proof Structure:**
```javascript
{
  proofVersion: 1,                    // Proof format version
  intent: { action, payload, raw },   // Original intent
  dappId: string,                     // dApp identifier
  dappCodeHash: string,               // SHA-256 of dApp source code
  capabilitiesDeclared: string[],     // Capabilities from manifest
  capabilitiesUsed: string[],          // Capabilities actually used
  inputHash: string,                  // SHA-256 of canonical input
  executionTraceHash: string,          // SHA-256 of execution trace
  resultHash: string,                 // SHA-256 of canonical result
  previousStateRoot: string,          // State root before execution
  nextStateRoot: string,              // State root after execution
  receiptHash: string,                // Receipt hash
  timestamp: number,                  // Execution timestamp
  executorPublicKey: string,          // Public key of executing node
  signature: string,                  // Ed25519 signature
  publicKey: string                   // Identity public key (from receipt)
}
```

**Required Fields:**
- `proofVersion` - Must be `1` (enables future proof format evolution)
- `executorPublicKey` - Public key of node that executed the task (for reputation tracking)

**Returns:**
```javascript
{
  proof: proofObject,
  trace: null  // Trace data not stored, would need separate storage
}
```

## Proof Verification

**API:** `FRAME.verifyExecutionProof(proof)`

Verifies an execution proof independently.

**Verification Algorithm:**

1. **Proof signature verification** - Verify Ed25519 signature of proof object
2. **Receipt hash verification** - Verify receipt hash matches proof.receiptHash
3. **Receipt verification** - Verify receipt exists and is valid (signature, chain link)
4. **dApp code hash verification** - Load dApp code, compute hash, verify matches `proof.dappCodeHash`
   - **Critical:** Execution proofs are invalid if code hash mismatches
   - Ensures remote nodes cannot run modified code
5. **Capability scope verification** - Verify `capabilitiesUsed` ⊆ `capabilitiesDeclared`
6. **Trace hash verification** - If trace provided:
   - Reconstruct deterministic execution environment
   - Replay trace operations
   - Recompute `executionTraceHash`
   - Verify matches `proof.executionTraceHash`
7. **Result hash verification** - Recompute result hash, verify matches `proof.resultHash`
8. **State root verification** - Recompute `nextStateRoot`, verify matches `proof.nextStateRoot`

**Returns:** `true` if all checks pass, `false` otherwise

**Failure modes:**
- Code hash mismatch → proof invalid (code tampering detected)
- Trace hash mismatch → proof invalid (nondeterministic execution)
- Result hash mismatch → proof invalid (incorrect result)
- State root mismatch → proof invalid (state corruption)

## Proof Verification Algorithm

A verification node runs:

```
1. Load dApp code by hash
2. Reconstruct deterministic environment
3. Replay trace (if provided)
4. Recompute traceHash
5. Recompute resultHash
6. Recompute nextStateRoot
7. Compare with proof
```

If all hashes match → execution is valid

## Use Cases

### Trustless Federation

Peers can verify remote execution results:
```javascript
var proof = await FRAME.generateExecutionProof(receiptHash);
// Send proof to peer
var valid = await peer.verifyExecutionProof(proof);
if (valid) {
  // Accept result
}
```

### Provable Agent Actions

Agents can prove task execution:
```javascript
var result = await executeTask(intent);
var proof = await FRAME.generateExecutionProof(result.receiptHash);
// Submit proof to coordinator
```

### Distributed Compute

Nodes can exchange proofs instead of recomputing:
```javascript
// Node A executes task
var proof = await FRAME.generateExecutionProof(receiptHash);

// Node B verifies without re-executing
var valid = await FRAME.verifyExecutionProof(proof);
if (valid) {
  await FRAME.commitVerifiedExecution(proof);
}
```

## Performance Considerations

- Trace recording overhead is minimal (simple append operations)
- Typical traces: 5-50 steps
- Trace data not stored in receipt chain (only hash)
- Trace can be exported/shared during verification

## Security Guarantees

- Trace never includes secrets (hashed instead)
- Proof verification is deterministic
- Code hash binding prevents code tampering
- Signature verification prevents forgery
- State root verification ensures state integrity

## Related Documentation

- [Receipt Chain](receipt_chain.md) - Receipt structure with executionTraceHash
- [Distributed Execution](distributed_execution.md) - Using proofs for DDE
- [Kernel Runtime](kernel_runtime.md) - Execution trace recording implementation
