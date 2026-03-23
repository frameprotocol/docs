# FRAME Formal State Machine

This document defines FRAME as a pure deterministic state machine. This is the ultimate correctness model.

## State Machine Definition

FRAME is a deterministic state machine where:

```
StateRootₙ₊₁ = transition(StateRootₙ, Intent)
```

### State

**State is represented by StateRoot:**

```typescript
StateRoot = SHA-256({
  version: STATE_ROOT_VERSION,
  capabilityVersion: CAPABILITY_VERSION,
  identityPublicKey: Ed25519PublicKey,
  installedDApps: [{ id, manifest, codeHash }],
  storage: { key: value, ... },
  receiptChainCommitment: ReceiptHash | null
})
```

### Transition Function

**Transition function:**

```typescript
function transition(stateRoot: StateRoot, intent: Intent): StateRoot {
  // 1. Verify state root matches current state
  assert(computeStateRoot() === stateRoot);
  
  // 2. Resolve intent to dApp
  const dApp = resolveIntent(intent);
  
  // 3. Execute dApp deterministically
  const result = executeDeterministically(dApp, intent);
  
  // 4. Update state
  const newState = updateState(stateRoot, result);
  
  // 5. Generate receipt
  const receipt = generateReceipt(stateRoot, newState, intent, result);
  
  // 6. Return new state root
  return newState;
}
```

### Determinism Property

**For any state root and intent:**

```
transition(stateRoot, intent) = transition(stateRoot, intent)
```

The same state root and intent always produce the same next state root.

### Receipt Property

**Every transition produces a receipt:**

```typescript
Receipt = {
  version: RECEIPT_VERSION,
  timestamp: Timestamp,
  identity: IdentityID,
  dappId: DAppID,
  intent: IntentAction,
  inputHash: SHA-256(inputPayload),
  inputPayload: CanonicalJSON,
  capabilitiesDeclared: Capability[],
  capabilitiesUsed: Capability[],
  previousStateRoot: StateRoot,
  nextStateRoot: StateRoot,
  resultHash: SHA-256(result),
  previousReceiptHash: ReceiptHash | null,
  receiptHash: SHA-256(signablePayload),
  publicKey: Ed25519PublicKey,
  signature: Ed25519Signature
}
```

**Receipt properties:**
- `previousStateRoot` = state root before transition
- `nextStateRoot` = state root after transition
- `receiptHash` links to previous receipt (chain)
- `signature` proves transition authenticity

## State Machine Properties

### Determinism

**Property:** Same inputs always produce same outputs.

**Formal:**
```
∀ stateRoot, intent:
  transition(stateRoot, intent) = transition(stateRoot, intent)
```

**Enforced by:**
- Frozen time (`Date.now()`)
- Seeded randomness (`Math.random()`)
- Blocked async APIs
- Canonical data structures
- Sealed network responses

### Verifiability

**Property:** Any state root can be verified by replaying receipts.

**Formal:**
```
∀ receiptChain:
  replay(receiptChain) = finalStateRoot
```

**Enforced by:**
- Receipt chain linking
- Deterministic replay
- State root recomputation
- Hash verification

### Integrity

**Property:** State root cannot be modified without valid receipt.

**Formal:**
```
∀ stateRoot, intent:
  transition(stateRoot, intent) produces receipt
  receipt.nextStateRoot = transition(stateRoot, intent)
```

**Enforced by:**
- Integrity lock
- State root checks
- Receipt chain linking
- Cryptographic signatures

### Isolation

**Property:** dApps cannot access undeclared capabilities.

**Formal:**
```
∀ dApp, capability:
  capability ∈ declaredCapabilities(dApp) ∧
  permissionGranted(identity, dApp, capability)
  ⇒ capabilityAccessible(dApp, capability)
```

**Enforced by:**
- Capability guard
- Scoped API construction
- Permission checks
- Manifest validation

## State Machine Transitions

### Normal Transition

**Normal execution:**

```
StateRoot₀ → Intent₁ → StateRoot₁ → Receipt₁
StateRoot₁ → Intent₂ → StateRoot₂ → Receipt₂
StateRoot₂ → Intent₃ → StateRoot₃ → Receipt₃
...
```

**Properties:**
- Each transition produces receipt
- Receipts form chain: `receipt[n].previousReceiptHash = receipt[n-1].receiptHash`
- State roots link: `receipt[n].previousStateRoot = receipt[n-1].nextStateRoot`

### Replay Transition

**Replay execution:**

```
Receipt₁ → Replay → StateRoot₁ (verify matches receipt.nextStateRoot)
Receipt₂ → Replay → StateRoot₂ (verify matches receipt.nextStateRoot)
Receipt₃ → Replay → StateRoot₃ (verify matches receipt.nextStateRoot)
...
```

**Properties:**
- Replay uses sealed inputs (no live I/O)
- Replay uses frozen time (receipt timestamp)
- Replay uses seeded randomness (receipt hash)
- Replay must produce same state root

### Composite Transition

**Composite execution:**

```
StateRoot₀ → [Intent₁, Intent₂, Intent₃] → StateRoot₃ → Receipt
```

**Properties:**
- All intents execute atomically
- Single receipt for entire composite
- Rollback on any failure
- State snapshot before execution

## State Machine Invariants

### Invariant 1: Receipt Chain Integrity

```
∀ receipt[n]:
  receipt[n].previousReceiptHash = receipt[n-1].receiptHash (if n > 0)
  receipt[n].previousReceiptHash = null (if n = 0)
```

### Invariant 2: State Root Consistency

```
∀ receipt[n]:
  receipt[n].previousStateRoot = receipt[n-1].nextStateRoot (if n > 0)
  receipt[n].previousStateRoot = initialStateRoot (if n = 0)
```

### Invariant 3: Receipt Hash Integrity

```
∀ receipt:
  receipt.receiptHash = SHA-256(signablePayload(receipt))
```

### Invariant 4: Signature Validity

```
∀ receipt:
  verifySignature(receipt.signature, receipt.receiptHash, receipt.publicKey) = true
```

### Invariant 5: Determinism

```
∀ receipt:
  replay(receipt.inputPayload) produces receipt.resultHash
```

## State Machine Verification

### Verification by Replay

**Verify state root by replaying receipts:**

```typescript
function verifyStateRoot(stateRoot: StateRoot, receiptChain: Receipt[]): boolean {
  let currentState = initialStateRoot;
  
  for (const receipt of receiptChain) {
    // Verify receipt links
    assert(receipt.previousStateRoot === currentState);
    
    // Replay intent
    const replayedState = transition(currentState, receipt.inputPayload);
    
    // Verify state root matches
    assert(replayedState === receipt.nextStateRoot);
    
    // Verify result hash matches
    const replayedResult = executeIntent(receipt.inputPayload);
    assert(SHA-256(replayedResult) === receipt.resultHash);
    
    currentState = replayedState;
  }
  
  return currentState === stateRoot;
}
```

### Verification by Attestation

**Verify state root by attestation:**

```typescript
function verifyAttestation(attestation: Attestation): boolean {
  // Verify signature
  assert(verifySignature(attestation.signature, attestation.payloadHash, attestation.publicKey));
  
  // Verify protocol versions
  assert(attestation.protocolVersion === PROTOCOL_VERSION);
  assert(attestation.receiptVersion === RECEIPT_VERSION);
  assert(attestation.capabilityVersion === CAPABILITY_VERSION);
  
  // Verify state root matches
  const computedRoot = computeStateRoot();
  assert(computedRoot === attestation.stateRoot);
  
  return true;
}
```

## State Machine Correctness

### Correctness Property

**FRAME is correct if:**

1. **Determinism:** Same inputs produce same outputs
2. **Verifiability:** State can be verified by replay
3. **Integrity:** State cannot be modified without receipt
4. **Isolation:** dApps cannot access undeclared capabilities
5. **Completeness:** All state transitions produce receipts

### Correctness Proof

**To prove FRAME is correct:**

1. **Prove determinism:**
   - Show frozen time prevents nondeterminism
   - Show seeded randomness prevents nondeterminism
   - Show blocked async APIs prevent nondeterminism
   - Show canonical data prevents nondeterminism

2. **Prove verifiability:**
   - Show receipts contain all information needed for replay
   - Show replay produces same state root
   - Show state root computation is deterministic

3. **Prove integrity:**
   - Show state root cannot be modified without receipt
   - Show receipts are cryptographically linked
   - Show signatures prevent tampering

4. **Prove isolation:**
   - Show capability guard enforces declarations
   - Show scoped API only includes declared capabilities
   - Show permission checks prevent unauthorized access

5. **Prove completeness:**
   - Show all state transitions produce receipts
   - Show receipts link to previous receipts
   - Show receipts include state roots

## State Machine Model Benefits

### Benefits

1. **Formal Verification** — Can prove properties mathematically
2. **Testing** — Can test state machine properties
3. **Debugging** — Can trace state transitions
4. **Documentation** — Clear correctness model
5. **Implementation** — Guides implementation decisions

### Applications

1. **Protocol Design** — Ensures protocol is well-defined
2. **Implementation** — Ensures implementation matches model
3. **Testing** — Tests state machine properties
4. **Verification** — Verifies runtime correctness
5. **Debugging** — Traces state transitions

## Related Documentation

- [Core Principles](core_principles.md) - Foundational invariants
- [Runtime Invariants](runtime_invariants.md) - Detailed invariants
- [Deterministic Execution](deterministic_execution.md) - Determinism rules
- [State Root](state_root.md) - State root computation
