# FRAME Runtime Invariants

This document lists every invariant the FRAME runtime must satisfy. These invariants are the foundation of runtime correctness and are verified at boot and during execution.

**File:** `ui/runtime/invariants.js`

## Invariant Categories

### 1. Protocol Invariants

#### Protocol Version Consistency

**Invariant:** All protocol versions must match across runtime modules.

- `PROTOCOL_VERSION` must be `"2.2.0"` in engine.js
- `RECEIPT_VERSION` must be `2` in engine.js
- `CAPABILITY_VERSION` must be `2` in engine.js
- `STATE_ROOT_VERSION` must be `3` in stateRoot.js

**Verification:** Boot-time checks in `invariants.js`

**Failure:** Runtime enters safe mode if versions mismatch

#### Receipt Schema Invariants

**Invariant:** Receipt fields must exactly match protocol spec.

- Receipt must have exactly 16 fields
- Fields must be in canonical order: `capabilitiesDeclared, capabilitiesUsed, dappId, identity, inputHash, inputPayload, intent, nextStateRoot, previousReceiptHash, previousStateRoot, publicKey, receiptHash, resultHash, signature, timestamp, version`
- `RECEIPT_FIELDS` array must be frozen

**Verification:** `assertReceiptFields()` in invariants.js

**Failure:** Receipt creation throws error

#### State Root Schema Invariants

**Invariant:** State root must have exactly 6 keys in canonical order.

- Keys: `capabilityVersion, identityPublicKey, installedDApps, receiptChainCommitment, storage, version`
- `STATE_ROOT_KEYS` array must be frozen
- Must include `receiptChainCommitment` (not `executionLog`)

**Verification:** `assertStateRootSchema()` in invariants.js

**Failure:** State root computation throws error

### 2. State Root Invariants

#### State Root Consistency

**Invariant:** State root must match before and after execution.

- `stateRoot(previous)` == `receipt.previousStateRoot`
- `stateRoot(next)` == `receipt.nextStateRoot`
- State root recomputed after each execution

**Verification:** Engine compares state roots in `handleIntent()`

**Failure:** Runtime enters safe mode

#### Replay State Root Match

**Invariant:** Replay must reproduce recorded state root.

- `replay(receipts)` must produce `receipt.nextStateRoot`
- State root recomputed after each replayed receipt
- Comparison happens in `replayExecutionLog()`

**Verification:** `replay.js` compares computed vs recorded state roots

**Failure:** Replay throws error, divergence detected

#### Receipt Chain Commitment

**Invariant:** State root must include receipt chain commitment.

- `receiptChainCommitment` = hash of latest receipt
- Not the full chain (efficient)
- Binds state root to receipt chain

**Verification:** `assertReceiptChainCommitment()` in invariants.js

**Failure:** State root computation fails

### 3. Receipt Chain Invariants

#### Chain Link Integrity

**Invariant:** Receipts must form a cryptographic chain.

- `receipt[n].previousReceiptHash` == `receipt[n-1].receiptHash`
- First receipt has `previousReceiptHash = null`
- Chain links verified during replay

**Verification:** `verifyChain()` in engine.js, `verifyReceiptChain()` in replay.js

**Failure:** Chain verification fails, safe mode entered

#### Receipt Hash Integrity

**Invariant:** Receipt hash must match signable payload.

- `receiptHash` = SHA-256 of signable payload (all fields except `receiptHash` and `signature`)
- Hash recomputed during verification
- Must match stored `receiptHash`

**Verification:** `verifyChain()` recomputes hash

**Failure:** Receipt verification fails

#### Signature Verification

**Invariant:** Receipt signatures must verify with identity public key.

- Signature is Ed25519 signature of `receiptHash`
- Public key stored in receipt
- Signature verified during chain verification

**Verification:** `verifyReceiptSignatureWithKey()` in replay.js

**Failure:** Signature verification fails, receipt rejected

#### Input Hash Match

**Invariant:** Input hash must match canonicalized input payload.

- `inputHash` = SHA-256 of canonicalized `inputPayload`
- Hash recomputed during verification
- Must match stored `inputHash`

**Verification:** `verifyReceiptChain()` recomputes input hash

**Failure:** Receipt verification fails

#### Result Hash Match

**Invariant:** Result hash must match canonicalized result.

- `resultHash` = SHA-256 of canonicalized result
- Hash recomputed during replay
- Must match stored `resultHash`

**Verification:** Replay recomputes result hash

**Failure:** Replay detects nondeterminism

### 4. Capability Invariants

#### Capability Schema Frozen

**Invariant:** Capability schema must be frozen at boot.

- `CAPABILITY_SCHEMA` must be frozen (`Object.freeze()`)
- Cannot be modified after boot
- Unknown capabilities rejected

**Verification:** `assertCapabilitySchema()` in invariants.js

**Failure:** Runtime initialization fails

#### Capability Declaration

**Invariant:** dApps must declare capabilities in manifest.

- Capabilities must exist in `CAPABILITY_SCHEMA`
- Invalid capabilities cause manifest rejection
- Router validates capabilities during manifest load

**Verification:** `validateManifest()` in router.js

**Failure:** Manifest rejected, dApp not loaded

#### Capability Permission

**Invariant:** dApps must be granted permission to use capabilities.

- Permission stored per identity: `permissions:<identity>`
- Permission checked before execution
- Missing permission triggers permission request

**Verification:** `hasPermission()` in engine.js

**Failure:** Permission request returned, execution blocked

#### Capability Usage Logging

**Invariant:** Capability usage must be logged in receipt.

- `capabilitiesUsed` array logged during execution
- Only declared capabilities can be used
- Usage logged via `capabilityGuard()`

**Verification:** `getExecutionCapLog()` in engine.js

**Failure:** Undeclared capability usage throws error

#### Capability Guard Enforcement

**Invariant:** All privileged operations must pass capability guard.

- `capabilityGuard()` checks declaration
- Throws if capability not declared
- Logs capability usage

**Verification:** All scoped API methods call `capabilityGuard()`

**Failure:** Undeclared capability access throws error

### 5. Determinism Invariants

#### Frozen Time

**Invariant:** `Date.now()` must be frozen during execution.

- Frozen to execution-start timestamp
- Restored after execution
- Required for deterministic replay

**Verification:** `installDeterministicSandbox()` freezes `Date.now`

**Failure:** Nondeterministic execution, replay fails

#### Seeded Randomness

**Invariant:** `Math.random()` must be seeded during execution.

- Seeded from receipt hash or previous receipt hash
- Restored after execution
- Required for deterministic replay

**Verification:** `installDeterministicSandbox()` replaces `Math.random`

**Failure:** Nondeterministic execution, replay fails

#### Async API Blocking

**Invariant:** Nondeterministic APIs must be blocked during execution.

- Blocked: `setTimeout`, `setInterval`, `fetch`, `WebSocket`, `EventSource`
- Stubs throw errors
- Restored after execution

**Verification:** `installAsyncBlockers()` blocks APIs

**Failure:** Nondeterministic execution, replay fails

#### Network Request Sealing

**Invariant:** Network requests must be sealed in receipts.

- Responses stored in `inputPayload.networkResponses`
- Replay uses sealed responses
- No live network during replay

**Verification:** Network log included in receipt

**Failure:** Nondeterministic network behavior, replay fails

#### Canonical Data Structures

**Invariant:** All storage values must be canonical JSON.

- Only safe integers (no floats, NaN, Infinity)
- No functions, classes, Dates, RegExp
- No circular references
- Deep freeze after write

**Verification:** `validateCanonicalJson()` in storage.js

**Failure:** Storage write rejected

### 6. Wallet Invariants

#### Total Supply Consistency

**Invariant:** Sum of account balances must equal total supply.

- `sum(account balances)` == `totalSupply`
- Total supply tracked globally
- Only mint/burn change total supply

**Verification:** Wallet dApp maintains balance

**Failure:** Supply mismatch, accounting error

#### Escrow Balance

**Invariant:** Escrow reservations must balance.

- Available balance = wallet balance - sum(locked reservations)
- Reservations stored in storage
- Release/refund update reservations

**Verification:** Wallet dApp computes available balance

**Failure:** Balance calculation error

#### Mint/Burn Capability

**Invariant:** Mint/burn require bridge capabilities.

- `wallet.mintFC` requires `bridge.mint`
- `wallet.burnFC` requires `bridge.burn`
- Capability checked before execution

**Verification:** Wallet dApp checks capabilities

**Failure:** Operation rejected

### 7. Federation Invariants

#### Chain Import Verification

**Invariant:** Imported chains must be verified before merge.

- Structural verification (fields, links)
- Signature verification
- Replay verification
- State root comparison

**Verification:** `mergeReceiptChains()` verifies before merge

**Failure:** Import rejected, merge fails

#### Divergence Detection

**Invariant:** Divergence must be detected before merge.

- Find common ancestor
- Extract divergent receipts
- Replay divergent receipts
- Compare state roots

**Verification:** `findDivergenceIndex()` in federation.js

**Failure:** Merge fails, divergence detected

### 8. Runtime Surface Invariants

#### Frozen Runtime Surfaces

**Invariant:** Runtime surfaces must be frozen after boot.

- `FRAME`, `FRAME_STORAGE`, `FRAME_IDENTITY`, etc. frozen
- Cannot be modified after boot
- Prevents runtime mutation

**Verification:** `assertFrozenSurfaces()` in invariants.js

**Failure:** Runtime initialization fails

#### Module Registration Removal

**Invariant:** Module registration hook must be removed after boot.

- `_FRAME_REGISTER` removed after all modules load
- Prevents new module registration
- Ensures runtime stability

**Verification:** `invariants.js` removes registration hook

**Failure:** Runtime instability

### 9. Boot Sequence Invariants

#### Integrity Lock Initialization

**Invariant:** Integrity lock must be initialized at boot.

- Boot state root captured
- Boot code hashes captured
- Lock verified before each execution

**Verification:** `initIntegrityLock()` in engine.js

**Failure:** Integrity checks fail

#### Replay Determinism Check

**Invariant:** Replay determinism must be verified at boot.

- Replay last receipt
- Verify state root matches
- Detect nondeterminism

**Verification:** `replayDeterminismCheck()` at boot

**Failure:** Boot fails, safe mode entered

#### State Reconstruction

**Invariant:** State must be reconstructible from receipt chain.

- Replay all receipts
- Verify state root matches
- Detect corruption

**Verification:** `reconstructAndVerify()` at boot

**Failure:** State corruption detected, safe mode entered

## Invariant Verification

### Boot-Time Verification

All invariants are verified during boot in `invariants.js`:

1. Protocol version checks
2. Receipt field checks
3. State root schema checks
4. Capability schema checks
5. Runtime surface checks
6. Module registration removal
7. Replay determinism check
8. State reconstruction

### Runtime Verification

Invariants are verified during execution:

1. State root consistency (before/after execution)
2. Receipt chain integrity (during append)
3. Capability checks (during execution)
4. Determinism checks (during sandbox installation)

### Failure Response

**On invariant violation:**

1. **Boot failure** — Runtime initialization fails
2. **Safe mode** — Runtime enters safe mode (state-changing operations disabled)
3. **Error thrown** — Operation rejected with error
4. **Logging** — Violation logged for debugging

## Related Documentation

- [Kernel Runtime](kernel_runtime.md) - Runtime implementation
- [Protocol Versions](protocol_versions.md) - Version constants
- [FRAME Protocol Spec](FRAME_PROTOCOL_SPEC.md) - Protocol specification
- [Deterministic Execution](deterministic_execution.md) - Determinism rules
