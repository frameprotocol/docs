# FRAME Architecture

This document describes the complete system architecture of FRAME, including the execution pipeline, component interactions, and system design principles.

## System Overview

FRAME is a local-first, deterministic runtime that executes user intents through capability-scoped dApps, producing cryptographically signed receipts that enable verifiable state reconstruction.

### Core Design Principles

1. **Deterministic Execution**: Same inputs always produce same outputs
2. **Capability Isolation**: dApps only access explicitly granted capabilities
3. **Cryptographic Integrity**: Every state transition is signed and linked
4. **Replay Safety**: Entire sessions reconstructible from receipt chains
5. **Local Sovereignty**: All execution occurs locally
6. **Verifiable UI**: Interfaces generated deterministically from intents

## Execution Pipeline

The complete execution flow from user input to UI rendering:

```
1. User Input
   ↓
2. Intent Parsing (AI or structured)
   ↓
3. Intent Normalization
   ↓
4. Router Resolution (find dApp)
   ↓
5. Permission Check (capability grants)
   ↓
6. State Root Computation (previous)
   ↓
7. Sandbox Installation (freeze time, seed random, block async)
   ↓
8. Application dApp Execution
   ↓
9. UI Planner dApp Execution (generate widget schema)
   ↓
10. Widget Schema Canonicalization
   ↓
11. State Root Computation (next)
   ↓
12. Receipt Creation (cryptographically signed)
   ↓
13. Receipt Append (to chain)
   ↓
14. Deterministic Layout Computation
   ↓
15. UI Rendering
```

### Stage-by-Stage Breakdown

#### 1. User Input

Users provide input via:
- **Text input**: Natural language processed by AI dApp
- **Structured intent**: Direct intent objects with action and payload

**Location**: `ui/app.js` → `handleCommand(text)`

#### 2. Intent Parsing (single pipeline)

User text goes through `processUserInput` → `fastIntentMap` only when confidence ≥ `FAST_INTENT_CONFIDENCE_MIN` (0.85) and `shouldBypassFastIntentMapForLLM` is false; otherwise `resolveIntent` calls `FRAME.handleIntent('ai.parse_intent', { text })` (host LLM + rules). Execution is always `FRAME.handleIntent(action, payload)`.

**Location**: `ui/system/conversationEngine.js` → `processUserInput`, `resolveIntent`; `ui/runtime/engine.js` → `fastIntentMap`; `ui/dapps/ai/index.js` → `ai.parse_intent` handler

#### 3. Intent Normalization

Intents are normalized to ensure consistent structure:
- System intents mapped to canonical forms
- Payloads canonicalized
- Timestamps normalized

**Location**: `ui/runtime/engine.js` → `kernelNormalize(intent)`

#### 4. Router Resolution

The router finds the best dApp for an intent:
- Scans installed dApps
- Matches intent action to dApp intents
- Validates manifest capabilities
- Returns best match

**Location**: `ui/runtime/router.js` → `resolve(intent)`

#### 5. Permission Check

FRAME checks if the identity has required capabilities:
- Reads capability grants from storage
- Verifies required capabilities are granted
- Returns permission request if missing

**Location**: `ui/runtime/engine.js` → `hasPermission(identity, dappId, capability)`

#### 6. State Root Computation (Previous)

Before execution, FRAME computes the current state root:
- Reads receipt chain
- Reads storage keys
- Computes deterministic hash

**Location**: `ui/runtime/stateRoot.js` → `computeStateRoot(identity)`

#### 7. Sandbox Installation

FRAME installs a deterministic sandbox:
- Freezes `Date.now()` to receipt timestamp
- Seeds `Math.random()` deterministically
- Blocks async APIs (`fetch`, `setTimeout`, `WebSocket`)

**Location**: `ui/runtime/engine.js` → `installDeterministicSandbox(timestamp)`

#### 8. Application dApp Execution

The dApp executes in the sandbox:
- dApp code loaded via dynamic import
- Scoped API provided (storage, capabilities)
- dApp `run(intent, ctx)` called
- Result returned

**Location**: `ui/runtime/router.js` → `execute(match, intent, buildScopedApi)`

#### 9. UI Planner dApp Execution

The UI planner generates a widget schema:
- Receives intent, execution result, dApp manifest
- Introspects capabilities from manifest
- Generates widget schema deterministically
- Caches schema for replay

**Location**: `ui/dapps/ui.planner/index.js` → `run(intent, ctx)`

#### 10. Widget Schema Canonicalization

The widget schema is canonicalized:
- Keys sorted
- Nested objects canonicalized
- Nondeterministic data rejected

**Location**: `ui/runtime/engine.js` → `canonicalize(widgetSchema)`

#### 11. State Root Computation (Next)

After execution, FRAME computes the next state root:
- Includes new storage writes
- Includes new receipt (via commitment)
- Computes deterministic hash

**Location**: `ui/runtime/stateRoot.js` → `computeStateRoot(identity)`

#### 12. Receipt Creation

FRAME creates a cryptographically signed receipt:
- Canonicalizes input payload
- Hashes execution result (includes widget schema)
- Links to previous receipt
- Signs with Ed25519

**Location**: `ui/runtime/engine.js` → `buildReceipt(opts)`

#### 13. Receipt Append

The receipt is appended to the chain:
- Written to append-only log (Rust backend)
- Frame format: length + checksum + JSON
- Crash-safe atomic write

**Location**: `src-tauri/src/receipt_log.rs` → `receipt_append_internal()`

#### 14. Deterministic Layout Computation

The layout engine computes widget placement:
- Based solely on widget count
- Deterministic rules (1 → single, 2-3 → vertical, 4 → grid, 5+ → dashboard)
- No system state dependencies

**Location**: `ui/runtime/widgets/layout.js` → `computeLayout(widgetSchema)`

#### 15. UI Rendering

Widgets are rendered to the UI:
- Widgets created from schema
- Layout applied
- Data sources connected
- UI updated

**Location**: `ui/system/widgetManager.js` → `renderAll()`

## Component Architecture

### Runtime Engine

**File**: `ui/runtime/engine.js`

The core execution kernel that:
- Handles intent execution
- Manages receipt creation
- Enforces capability permissions
- Installs deterministic sandbox
- Coordinates UI planner execution

**Key Functions**:
- `handleIntent(intent)`: Main entry point for intent execution
- `buildReceipt(opts)`: Creates cryptographically signed receipts
- `hasPermission(identity, dappId, capability)`: Checks capability grants
- `installDeterministicSandbox(timestamp)`: Sets up deterministic execution environment

### Router

**File**: `ui/runtime/router.js`

Resolves intents to dApps and executes them:
- Scans installed dApps
- Matches intents to dApp capabilities
- Validates manifest structure
- Executes dApps in sandbox

**Key Functions**:
- `resolve(intent)`: Finds best dApp for intent
- `execute(match, intent, buildScopedApi)`: Executes dApp with scoped API

### Storage Engine

**File**: `ui/runtime/storage.js`

Provides canonical JSON storage:
- Validates all writes are canonical JSON
- Rejects nondeterministic types (floats, NaN, Infinity)
- Deep-freezes committed values
- Wraps Tauri storage backend

**Key Functions**:
- `read(key)`: Reads from storage
- `write(key, value)`: Writes to storage (with validation)

### Receipt Log

**File**: `src-tauri/src/receipt_log.rs`

Append-only receipt storage:
- Frame format: u32 length + u32 checksum + JSON payload
- Crash-safe atomic writes
- Migration from legacy format

**Key Functions**:
- `receipt_append_internal()`: Appends receipt to log
- `receipt_read_all()`: Reads all receipts from log

### State Root

**File**: `ui/runtime/stateRoot.js`

Computes deterministic state root:
- Includes identity public key
- Includes installed dApps (with code hashes)
- Includes storage (excluding ephemeral keys)
- Includes receipt chain commitment

**Key Functions**:
- `computeStateRoot(identity)`: Computes state root hash

### UI Planner dApp

**File**: `ui/dapps/ui.planner/index.js`

Generates widget schemas from intents:
- Introspects dApp capabilities
- Generates deterministic widget schemas
- Caches schemas for replay
- Returns canonical JSON only

**Key Functions**:
- `run(intent, ctx)`: Generates widget schema
- `generateWidgetSchema()`: Creates schema from intent
- `introspectCapabilities()`: Discovers capabilities from manifest

### Layout Engine

**File**: `ui/runtime/widgets/layout.js`

Computes deterministic layouts:
- Based solely on widget count
- No system state dependencies
- Assigns widgets to regions

**Key Functions**:
- `computeLayout(widgetSchema)`: Computes layout type
- `assignWidgetsToRegions()`: Assigns widgets to regions
- `applyLayout()`: Applies layout to UI

### Replay System

**File**: `ui/runtime/replay.js`

Reconstructs state from receipt chains:
- Replays receipts in order
- Re-executes dApps with same inputs
- Verifies state root matches
- Detects nondeterminism

**Key Functions**:
- `replayIntent(receiptHash)`: Replays single receipt
- `verifyDeterminism(receiptChain)`: Verifies chain determinism
- `reconstructStateFromReceipts()`: Full state reconstruction

## Data Flow

### Intent → Execution → Receipt

```
Intent Object
  {
    action: "wallet.balance",
    payload: {},
    timestamp: 1234567890
  }
  ↓
Router Resolution
  → Finds "wallet" dApp
  → Validates manifest
  ↓
Permission Check
  → Checks "wallet.read" grant
  ↓
dApp Execution
  → Runs wallet dApp code
  → Returns { balance: 100 }
  ↓
UI Planner
  → Generates widget schema
  → { widgets: [{ type: "card", source: "wallet.balance" }] }
  ↓
Receipt Creation
  {
    version: 2,
    timestamp: 1234567890,
    intent: "wallet.balance",
    inputHash: "...",
    resultHash: "...",  // Includes widget schema
    previousStateRoot: "...",
    nextStateRoot: "...",
    receiptHash: "...",
    signature: "..."
  }
  ↓
Receipt Append
  → Written to chain_<identity>.log
```

### State Root Computation

```
State Root Components:
  - identityPublicKey: Ed25519 public key
  - installedDApps: [{ id, manifest, codeHash }]
  - storage: { key: value } (canonicalized)
  - receiptChainCommitment: Rolling hash of receipt chain
  - version: STATE_ROOT_VERSION

Computation:
  1. Canonicalize all components
  2. Sort keys deterministically
  3. SHA-256 hash of canonical JSON
  4. Result: 64-character hex string
```

## System Interactions

### Runtime ↔ Router

- Runtime calls router to resolve intents
- Router returns dApp match
- Runtime calls router to execute dApp
- Router provides scoped API to dApp

### Runtime ↔ Storage

- Runtime reads/writes via storage abstraction
- Storage validates canonical JSON
- Storage wraps Tauri backend

### Runtime ↔ Receipt Log

- Runtime calls Tauri to append receipts
- Receipt log writes to append-only file
- Receipt log provides crash safety

### Runtime ↔ State Root

- Runtime computes state root before/after execution
- State root includes storage, dApps, receipt chain
- State root used in receipt for verification

### Runtime ↔ UI Planner

- Runtime calls UI planner after dApp execution
- UI planner generates widget schema
- Widget schema included in execution result
- Widget schema hashed in receipt

### Layout Engine ↔ Widget Manager

- Layout engine computes layout from schema
- Layout engine applies layout to widget manager
- Widget manager renders widgets

## Protocol Versions

FRAME uses frozen protocol versions:

- **PROTOCOL_VERSION**: `2.2.0`
- **RECEIPT_VERSION**: `2`
- **CAPABILITY_VERSION**: `2`
- **STATE_ROOT_VERSION**: `3`

These versions ensure compatibility across FRAME instances.

## Security Model

FRAME enforces security through:

1. **Capability Isolation**: dApps only access granted capabilities
2. **Sandbox Execution**: Nondeterministic APIs blocked
3. **Cryptographic Signatures**: Receipts signed with Ed25519
4. **Code Hash Verification**: dApp code integrity verified
5. **State Root Verification**: State integrity verified

See [Security](security.md) for details.

## Determinism Guarantees

FRAME guarantees:

1. Same receipt chain → same state root
2. Same intent → same execution result
3. Same execution result → same widget schema
4. Same widget schema → same layout
5. Same layout → same UI

See [Determinism and Replay](determinism-and-replay.md) for details.
