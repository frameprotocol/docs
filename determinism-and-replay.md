# FRAME Determinism and Replay

This document explains FRAME's deterministic execution guarantees and replay system, including how replay produces identical behavior and how determinism is verified.

## Deterministic Guarantees

FRAME guarantees that:

```
same receipt chain
  → same state root
  → same execution results
  → same widget schemas
  → same layout
  → same interface
```

This means FRAME can reconstruct entire application sessions from receipt chains, enabling verifiable computation proofs and cross-instance state verification.

## Deterministic Execution

FRAME enforces determinism through multiple mechanisms:

### Time Freezing

`Date.now()` is frozen to the execution timestamp:

**Location**: `ui/runtime/engine.js` → `installDeterministicSandbox()`

**Implementation**:
```javascript
var frozenTime = intent.timestamp || Date.now();
Date.now = function() { return frozenTime; };
```

**Purpose**: Prevents time-dependent nondeterminism

### Random Seeding

`Math.random()` is seeded deterministically:

**Implementation**:
```javascript
var seed = previousReceiptHash || 'GENESIS';
var rng = seededRandom(seed);
Math.random = function() { return rng(); };
```

**Purpose**: Prevents random-dependent nondeterminism

### Async Blocking

Nondeterministic async APIs are blocked:

**Blocked APIs**:
- `fetch()` - Network requests
- `setTimeout()` - Timers
- `WebSocket()` - WebSocket connections
- `EventSource()` - Server-sent events

**Implementation**:
```javascript
window.fetch = function() {
  throw new Error('fetch blocked in deterministic execution');
};
```

**Purpose**: Prevents network-dependent nondeterminism

### Canonical Data

All storage values are canonical JSON:

**Allowed Types**:
- Strings, booleans, safe integers, null
- Arrays, plain objects

**Rejected Types**:
- Floating point numbers
- NaN, Infinity
- undefined, functions
- Date, RegExp, Map, Set, etc.

**Purpose**: Ensures deterministic serialization

## Receipt Chain Replay

FRAME can replay receipt chains to reconstruct state:

### Replay Process

**Location**: `ui/runtime/replay.js` → `replayExecutionLog()`

**Steps**:
1. Load receipt chain
2. Enter reconstruction mode
3. For each receipt:
   - Restore frozen timestamp
   - Inject network responses (from receipt)
   - Re-execute intent
   - Verify state root matches
   - Verify result hash matches
4. Exit reconstruction mode

### Reconstruction Mode

**Behavior**:
- No receipts appended to chain
- No state mutations persisted
- Used for verification only

**Entry**:
```javascript
window.FRAME._engine.enterReconstructionMode();
```

**Exit**:
```javascript
window.FRAME._engine.exitReconstructionMode();
```

### Replay Verification

**State Root Verification**:
```javascript
var computedRoot = await computeStateRoot(identity);
if (computedRoot !== receipt.nextStateRoot) {
  throw new Error('State root mismatch');
}
```

**Result Hash Verification**:
```javascript
var computedResultHash = await canonicalHash(result);
if (computedResultHash !== receipt.resultHash) {
  throw new Error('Result hash mismatch');
}
```

## State Root Verification

State roots are computed deterministically:

### State Root Components

**Location**: `ui/runtime/stateRoot.js` → `computeStateRoot()`

**Components**:
- Identity public key
- Installed dApps (with code hashes)
- Storage state (canonicalized, sorted keys)
- Receipt chain commitment (rolling hash)

### State Root Computation

**Process**:
1. Canonicalize all components
2. Sort keys deterministically
3. SHA-256 hash of canonical JSON
4. Result: 64-character hex string

**Guarantee**: Identical state always yields identical root

### State Root Verification

**Boot-Time Lock**:
- State root computed at boot
- Stored as `_bootStateRoot`
- Verified before each execution

**Verification**:
```javascript
var currentRoot = await computeStateRoot(identity);
if (currentRoot !== _bootStateRoot) {
  // Enter safe mode
}
```

## Schema Hashing

Widget schemas are hashed in receipts:

### Schema Inclusion

**Process**:
1. UI planner generates widget schema
2. Schema canonicalized
3. Schema added to `execResult.widgetSchema`
4. `resultHash = hash(execResult)` includes schema
5. Receipt includes `resultHash`

**Verification**:
- Replay verifies `resultHash` matches
- Ensures identical widget schemas

## Planner Determinism

The UI planner is fully deterministic:

### Deterministic Operations

- Sorted key iteration (`Object.keys().sort()`)
- Canonical JSON hashing
- Intent timestamp (not `Date.now()`)
- Safe integer arithmetic only

### Blocked Operations

- `Date.now()` (uses intent timestamp)
- `Math.random()` (not used)
- Floating point arithmetic (only safe integers)
- Unordered object iteration (must sort keys)

### Verification

Planner determinism verified through:
- Schema canonicalization
- Cache key determinism
- Replay consistency

## Layout Determinism

The layout engine is fully deterministic:

### Layout Rules

Layout computed solely from widget count:
- 1 widget → single
- 2-3 widgets → vertical
- 4 widgets → grid
- 5+ widgets → dashboard

### Determinism Guarantees

- Same widget count → same layout type
- Same widget order → same region assignments
- Widgets sorted by source before assignment
- No system state dependencies

## FRAME.verifyDeterminism()

FRAME provides a utility to verify determinism:

**Location**: `ui/runtime/replay.js` → `verifyDeterminism(receiptChain)`

**Process**:
1. Replay chain once → compute state root
2. Replay chain again → compute state root again
3. Compare state roots
4. If different → throw deterministic violation error

**Usage**:
```javascript
var result = await window.FRAME.verifyDeterminism(receiptChain);
console.log(result.verified); // true if deterministic
```

## Replay Consistency Check

FRAME verifies replay consistency:

### Replay Intent

**Location**: `ui/runtime/replay.js` → `replayIntent(receiptHash)`

**Process**:
1. Find receipt in chain
2. Replay from genesis to receipt
3. Verify state root matches
4. Return replay result

**Usage**:
```javascript
var result = await window.FRAME.replayIntent(receiptHash);
console.log(result.replayValid); // true if replay matches
```

### Determinism Check

**Location**: `ui/runtime/replay.js` → `replayDeterminismCheck()`

**Process**:
1. Verify receipt chain structure
2. Verify state transition continuity
3. Verify code hash integrity
4. Verify state root stability
5. Verify storage stability

**Usage**:
```javascript
var result = await window.FRAME.replayDeterminismCheck();
console.log(result); // true if all checks pass
```

## Why Replay Produces Identical Behavior

Replay produces identical behavior because:

1. **Frozen Time**: `Date.now()` returns same value
2. **Seeded Random**: `Math.random()` returns same sequence
3. **Blocked Async**: No network calls (uses sealed responses)
4. **Canonical Data**: All data canonicalized
5. **Sorted Iteration**: All iteration uses sorted keys
6. **Same Code**: dApp code verified via hash
7. **Same Inputs**: Receipt contains canonicalized input
8. **Same State**: State root matches at each step

## Determinism Violations

FRAME detects determinism violations:

### Detection Mechanisms

1. **State Root Mismatch**: Replay state root doesn't match receipt
2. **Result Hash Mismatch**: Replay result hash doesn't match receipt
3. **Code Hash Mismatch**: dApp code hash changed
4. **Receipt Chain Broken**: Receipt links don't match

### Response

- Safe mode triggered
- Execution blocked
- User notified
- Recovery mode enabled

## Determinism Guarantees Summary

FRAME guarantees:

1. **Same Receipt Chain → Same State Root**: Replay always produces same state root
2. **Same Intent → Same Result**: Same intent always produces same execution result
3. **Same Result → Same Widget Schema**: Same result always produces same widget schema
4. **Same Schema → Same Layout**: Same schema always produces same layout
5. **Same Layout → Same UI**: Same layout always produces same UI

These guarantees enable:
- Verifiable computation proofs
- Cross-instance state verification
- Deterministic debugging
- Cryptographic integrity guarantees
