# FRAME Runtime

This document explains the internals of the FRAME runtime, including execution flow, intent routing, sandbox system, and capability enforcement.

## Runtime Overview

The FRAME runtime (`ui/runtime/engine.js`) is the core execution kernel that processes user intents through capability-scoped dApps, producing cryptographically signed receipts.

**Key Responsibilities**:
- Intent normalization and validation
- Router coordination for dApp resolution
- Capability permission enforcement
- Deterministic sandbox installation
- Receipt creation and signing
- State root computation coordination
- UI planner integration

## Execution Flow

### Main Entry Point: `handleIntent(intent)`

The runtime's main entry point processes intents through these stages:

```javascript
handleIntent(intent)
  ↓
1. Intent Normalization
  ↓
2. Intent Validation
  ↓
3. Router Resolution
  ↓
4. Permission Check
  ↓
5. Integrity Lock Check
  ↓
6. Code Hash Verification
  ↓
7. Previous State Root Computation
  ↓
8. Deterministic Sandbox Installation
  ↓
9. Application dApp Execution
  ↓
10. UI Planner Execution
  ↓
11. Widget Schema Canonicalization
  ↓
12. Next State Root Computation
  ↓
13. Receipt Building
  ↓
14. Receipt Appending
  ↓
15. Layout Application
  ↓
16. Return Result
```

### Stage 1: Intent Normalization

Intents are normalized to ensure consistent structure:

**Location**: `kernelNormalize(intent)`

**Process**:
- System intents mapped to canonical forms
- Payloads canonicalized
- Timestamps normalized
- Raw input preserved

**Example**:
```javascript
// Input
{ action: "wallet.balance", payload: { foo: undefined } }

// Normalized
{ 
  action: "wallet.balance", 
  payload: {},  // undefined removed
  timestamp: 1234567890,
  raw: "show balance"
}
```

### Stage 2: Intent Validation

Intents are validated for structure:

**Checks**:
- `action` must be a string
- `payload` must be an object (if present)
- System intents handled specially

### Stage 3: Router Resolution

The router finds the best dApp for the intent:

**Location**: `router.resolve(intent)`

**Process**:
1. Scans installed dApps
2. Matches intent action to dApp intents
3. Validates manifest structure
4. Returns best match

**Example**:
```javascript
// Intent: { action: "wallet.balance" }
// Router finds: { id: "wallet", intents: ["wallet.balance", ...] }
```

### Stage 4: Permission Check

FRAME checks if the identity has required capabilities:

**Location**: `hasPermission(identity, dappId, capability)`

**Process**:
1. Reads capability grants from storage (`permissions:${identity}`)
2. Checks if dApp's declared capabilities are granted
3. Returns `true` if all required capabilities granted
4. Returns permission request if missing

**Capability Grants Structure**:
```javascript
{
  "wallet": ["wallet.read", "wallet.send"],
  "notes": ["storage.read", "storage.write"]
}
```

### Stage 5: Integrity Lock Check

FRAME verifies state root matches boot-time state root:

**Purpose**: Detects state corruption or tampering

**Process**:
1. Computes current state root
2. Compares to boot-time state root
3. Returns error if mismatch (triggers safe mode)

### Stage 6: Code Hash Verification

FRAME verifies dApp code hash matches manifest:

**Purpose**: Detects code drift or tampering

**Process**:
1. Computes current dApp code hash (SHA-256 of source)
2. Compares to hash stored in state root
3. Returns error if mismatch

### Stage 7: Previous State Root Computation

FRAME computes the current state root before execution:

**Location**: `FRAME_STATE_ROOT.computeStateRoot(identity)`

**Components**:
- Identity public key
- Installed dApps (with code hashes)
- Storage state (canonicalized)
- Receipt chain commitment

**Purpose**: Captures state before execution for receipt

### Stage 8: Deterministic Sandbox Installation

FRAME installs a deterministic execution environment:

**Location**: `installDeterministicSandbox(timestamp, seed)`

**Process**:
1. Freezes `Date.now()` to execution timestamp
2. Seeds `Math.random()` deterministically
3. Blocks async APIs (`fetch`, `setTimeout`, `WebSocket`)
4. Returns restore function

**Implementation**:
```javascript
// Freeze Date.now
var frozenTime = timestamp;
Date.now = function() { return frozenTime; };

// Seed Math.random
var seed = previousReceiptHash || 'GENESIS';
var rng = seededRandom(seed);
Math.random = function() { return rng(); };

// Block async APIs
window.fetch = function() { 
  throw new Error('fetch blocked in deterministic execution'); 
};
```

### Stage 9: Application dApp Execution

The dApp executes in the sandbox:

**Location**: `router.execute(match, intent, buildScopedApi)`

**Process**:
1. Loads dApp code via dynamic import
2. Builds scoped API (only granted capabilities)
3. Calls `dApp.run(intent, ctx)`
4. Returns execution result

**Scoped API Structure**:
```javascript
{
  storage: {
    read: (key) => { /* capability guard */ },
    write: (key, value) => { /* capability guard */ }
  },
  wallet: {
    getBalance: () => { /* capability guard */ },
    send: (target, amount) => { /* capability guard */ }
  },
  identity: {
    getCurrent: () => { /* capability guard */ }
  }
}
```

### Stage 10: UI Planner Execution

The UI planner generates a widget schema:

**Location**: `router.execute(plannerMatch, plannerIntent, buildScopedApi)`

**Process**:
1. Resolves `ui.planner` dApp
2. Passes intent, execution result, dApp manifest
3. Planner generates widget schema deterministically
4. Returns widget schema

**Widget Schema Structure**:
```javascript
{
  type: "ui.plan",
  widgets: [
    {
      type: "card",
      title: "Wallet Balance",
      source: "wallet.balance",
      region: "workspace",
      params: {}
    }
  ]
}
```

### Stage 11: Widget Schema Canonicalization

The widget schema is canonicalized:

**Location**: `canonicalize(widgetSchema)`

**Process**:
1. Sorts all object keys
2. Recursively canonicalizes nested objects
3. Removes undefined values
4. Validates no functions or floating point

**Purpose**: Ensures deterministic hashing in receipt

### Stage 12: Next State Root Computation

FRAME computes the state root after execution:

**Location**: `FRAME_STATE_ROOT.computeStateRoot(identity)`

**Process**:
1. Includes new storage writes
2. Includes new receipt (via commitment)
3. Computes deterministic hash

**Note**: State root computed before receipt is built (to break circular dependency)

### Stage 13: Receipt Building

FRAME creates a cryptographically signed receipt:

**Location**: `buildReceipt(opts)`

**Receipt Structure**:
```javascript
{
  version: 2,
  timestamp: 1234567890,
  identity: "identity_id",
  dappId: "wallet",
  intent: "wallet.balance",
  inputHash: "sha256(canonicalize(inputPayload))",
  inputPayload: { /* canonicalized */ },
  resultHash: "sha256(canonicalize(execResult))",  // Includes widget schema
  previousStateRoot: "...",
  nextStateRoot: "...",
  previousReceiptHash: "...",
  receiptHash: "sha256(signablePayload)",
  signature: "ed25519_signature",
  publicKey: "ed25519_public_key",
  capabilitiesDeclared: ["wallet.read"],
  capabilitiesUsed: ["wallet.read"],
  executionTraceHash: "..."  // If trace enabled
}
```

**Signable Payload** (excludes `receiptHash` and `signature`):
```javascript
{
  version: 2,
  timestamp: 1234567890,
  identity: "identity_id",
  dappId: "wallet",
  intent: "wallet.balance",
  inputHash: "...",
  inputPayload: {...},
  resultHash: "...",
  previousStateRoot: "...",
  nextStateRoot: "...",
  previousReceiptHash: "...",
  capabilitiesDeclared: [...],
  capabilitiesUsed: [...]
}
```

### Stage 14: Receipt Appending

The receipt is appended to the chain:

**Location**: `src-tauri/src/receipt_log.rs` → `receipt_append_internal()`

**Process**:
1. Serializes receipt to canonical JSON
2. Computes checksum
3. Writes frame: `u32 length + u32 checksum + JSON bytes`
4. Atomic write (crash-safe)

### Stage 15: Layout Application

The layout engine applies widget layout:

**Location**: `FRAME_LAYOUT_ENGINE.applyLayout(widgetSchema)`

**Process**:
1. Computes layout from widget count
2. Assigns widgets to regions
3. Applies to layout manager

## Router Behavior

The router (`ui/runtime/router.js`) resolves intents to dApps:

### Resolution Process

1. **Scan Installed dApps**: Lists dApps from Tauri backend
2. **Load Manifests**: Fetches and parses `manifest.json` for each dApp
3. **Match Intents**: Finds dApp whose `intents` array contains the intent action
4. **Validate Manifest**: Ensures manifest structure is valid
5. **Return Match**: Returns best matching dApp

### Manifest Validation

Manifests must contain:
- `name`: String
- `id`: String (unique identifier)
- `intents`: Array of strings
- `capabilities`: Array of strings (must be in `CAPABILITY_SCHEMA`)

### Execution Process

1. **Load dApp Code**: Dynamic import of `dapps/${id}/index.js`
2. **Build Scoped API**: Creates API with only granted capabilities
3. **Call dApp**: `dApp.run(intent, ctx)`
4. **Return Result**: Returns dApp execution result

## Sandbox System

The sandbox isolates dApp execution and ensures determinism:

### Time Freezing

`Date.now()` is frozen to the execution timestamp:

```javascript
var frozenTime = intent.timestamp || Date.now();
Date.now = function() { return frozenTime; };
```

### Random Seeding

`Math.random()` is seeded deterministically:

```javascript
var seed = previousReceiptHash || 'GENESIS';
var rng = seededRandom(seed);
Math.random = function() { return rng(); };
```

### Async Blocking

Nondeterministic async APIs are blocked:

```javascript
window.fetch = function() { 
  throw new Error('fetch blocked in deterministic execution'); 
};
window.setTimeout = function() { 
  throw new Error('setTimeout blocked in deterministic execution'); 
};
window.WebSocket = function() { 
  throw new Error('WebSocket blocked in deterministic execution'); 
};
```

### Sandbox Restoration

After execution, the sandbox is restored:

```javascript
var restore = installDeterministicSandbox(timestamp, seed);
try {
  // Execute dApp
} finally {
  restore(); // Restore original APIs
}
```

## Capability Enforcement

FRAME enforces capability-based permissions:

### Capability Guard

Every API call goes through a capability guard:

```javascript
function capabilityGuard(capability, fn) {
  return function(...args) {
    // Log capability use
    logCapabilityUse(capability);
    
    // Check permission
    if (!hasPermission(identity, dappId, capability)) {
      throw new Error(`Capability ${capability} not granted`);
    }
    
    // Execute function
    return fn(...args);
  };
}
```

### Scoped API Building

The scoped API only exposes granted capabilities:

```javascript
function buildScopedApi(dappId, grantedCaps) {
  var api = {};
  
  if (grantedCaps.indexOf('storage.read') !== -1) {
    api.storage = {
      read: capabilityGuard('storage.read', storageRead)
    };
  }
  
  if (grantedCaps.indexOf('wallet.read') !== -1) {
    api.wallet = {
      getBalance: capabilityGuard('wallet.read', walletGetBalance)
    };
  }
  
  return api;
}
```

### Capability Logging

Used capabilities are logged during execution:

```javascript
var _executionCapLog = [];

function logCapabilityUse(capability) {
  if (_executionCapLog.indexOf(capability) === -1) {
    _executionCapLog.push(capability);
  }
}
```

Logged capabilities are included in the receipt as `capabilitiesUsed`.

## Intent Normalization

Intents are normalized for consistent processing:

### Normalization Rules

1. **System Intents**: Mapped to canonical forms
2. **Payload Canonicalization**: All payloads canonicalized
3. **Timestamp Normalization**: Timestamps normalized to safe integers
4. **Raw Preservation**: Original input preserved in `raw` field

### Example

```javascript
// Input
{
  action: "wallet.balance",
  payload: { foo: undefined, bar: 123.456 },
  timestamp: "2024-01-01T00:00:00Z"
}

// Normalized
{
  action: "wallet.balance",
  payload: { bar: 123 },  // undefined removed, float rounded
  timestamp: 1704067200000,  // normalized
  raw: "show balance"
}
```

## Deterministic Execution Rules

FRAME enforces strict determinism:

### Allowed Operations

- Canonical JSON operations
- Safe integer arithmetic
- Deterministic string operations
- Sorted key iteration

### Blocked Operations

- `Date.now()` (frozen)
- `Math.random()` (seeded)
- `fetch()` (blocked)
- `setTimeout()` (blocked)
- Floating point arithmetic (only safe integers)
- Unordered object iteration (must sort keys)

### Verification

The runtime verifies determinism through:
- State root comparison (replay must match)
- Receipt hash verification
- Code hash verification
- `FRAME.verifyDeterminism()` utility

## Mode System

FRAME supports multiple execution modes:

- **normal**: Standard execution mode
- **safe**: Safe mode (integrity violations detected)
- **restoring**: State restoration mode
- **replay**: Replay mode (no receipts appended)

Modes are managed via `setMode()` and checked via `getMode()`.

## Error Handling

The runtime handles errors at multiple levels:

1. **Intent Validation**: Returns error if intent invalid
2. **Permission Check**: Returns permission request if missing
3. **Integrity Lock**: Returns error if state root mismatch
4. **Code Hash**: Returns error if code hash mismatch
5. **Execution**: Returns error if dApp execution fails
6. **Receipt Building**: Returns error if receipt build fails

All errors are returned as `{ type: 'error', message: '...' }`.

## Protocol Constants

FRAME uses frozen protocol constants:

- **PROTOCOL_VERSION**: `2.2.0`
- **RECEIPT_VERSION**: `2`
- **CAPABILITY_VERSION**: `2`
- **STATE_ROOT_VERSION**: `3`

These constants are non-writable and define protocol compatibility.
