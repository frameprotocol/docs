# FRAME: A Sovereign, Deterministic Operating System for Verifiable Computing

**Version:** 2.2.0
**Date:** April 13, 2026
**Author:** FRAME Protocol
**Repository:** https://github.com/frameprotocol/docs

---

## Abstract

FRAME is a sovereign, deterministic, local-first operating system and runtime that processes user intents through a capability-scoped execution kernel, producing cryptographically signed receipts that form an immutable chain. Every state transition is verifiable, replayable, and auditable. The system provides 37 frozen capabilities across 17 namespaces, enforced at the kernel level with deny-by-default semantics. State roots are computed as SHA-256 hashes over canonicalized system state, enabling cross-node verification. Identity is Ed25519-based with AES-256-GCM encrypted per-identity storage using authenticated encryption (AEAD). Federation between peers uses a trusted-peer registry with strict mode enforcement in production. The user interface is entirely dynamic — rendered from dApp-produced widget schemas against declarative themes and layouts — with no hardcoded content. FRAME establishes a new paradigm for computing: one where every operation is provably correct, every state transition is cryptographically sealed, and sovereignty rests entirely with the user.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Vision and Design Principles](#2-vision-and-design-principles)
3. [System Architecture](#3-system-architecture)
4. [Deterministic Execution Model](#4-deterministic-execution-model)
5. [Cryptographic Foundations](#5-cryptographic-foundations)
6. [Capability-Based Security Model](#6-capability-based-security-model)
7. [Federation and Synchronization](#7-federation-and-synchronization)
8. [State Roots, Snapshots, and Replay](#8-state-roots-snapshots-and-replay)
9. [dApp Ecosystem](#9-dapp-ecosystem)
10. [FRAME SDK](#10-frame-sdk)
11. [Accessibility and Human-Centered Design](#11-accessibility-and-human-centered-design)
12. [Performance and Observability](#12-performance-and-observability)
13. [Use Cases](#13-use-cases)
14. [Comparison with Existing Platforms](#14-comparison-with-existing-platforms)
15. [Governance and Future Roadmap](#15-governance-and-future-roadmap)
16. [Security Considerations](#16-security-considerations)
17. [Conclusion](#17-conclusion)
18. [Appendices](#18-appendices)

---

## 1. Introduction

Contemporary computing platforms suffer from fundamental limitations that undermine user sovereignty, security, and verifiability:

**Centralized infrastructure.** Cloud-dependent architectures create single points of failure, surveillance vectors, and vendor lock-in. Users surrender control of their data and computation to third-party operators.

**Opaque execution.** Applications mutate state through arbitrary code paths without audit trails. Users cannot verify what operations were performed, what data was accessed, or whether results are correct.

**Non-deterministic software.** Most applications rely on ambient state — system clocks, random number generators, network timing, floating-point hardware — making it impossible to reproduce execution or verify correctness.

**Lack of user sovereignty.** Users cannot inspect, export, or verify their own data. Application state is entangled with platform state, making migration or independent verification infeasible.

FRAME addresses these limitations by introducing a new computing model where:

- Every operation flows through a single, auditable kernel that produces cryptographically signed execution receipts.
- All state transitions are deterministic and reproducible from the receipt chain.
- Applications are isolated through a frozen capability taxonomy with deny-by-default enforcement.
- Identity is user-owned (Ed25519 keypairs) with locally encrypted storage.
- The user interface is entirely derived from execution results — no hardcoded content exists.
- Federation between nodes is opt-in, authenticated, and verifiable.

---

## 2. Vision and Design Principles

FRAME's architecture is guided by eight core principles, each directly reflected in the implementation:

### 2.1 Local-First Computing

All computation occurs on the user's device. The Tauri backend (Rust) handles cryptographic operations, encrypted storage, and receipt persistence locally. No external service is required for core functionality.

*Implementation: `src-tauri/src/lib.rs`, `src-tauri/src/storage.rs`*

### 2.2 Deterministic Execution

The kernel normalizes all inputs through canonical JSON serialization before execution. Non-deterministic APIs (`Date.now()`, `Math.random()`, `setTimeout`) are sandboxed during dApp execution, replaced with receipt-derived deterministic values.

*Implementation: `ui/runtime/engine.js:installDeterministicSandbox()`*

### 2.3 Cryptographic Verifiability

Every state transition produces a signed receipt containing SHA-256 hashes of inputs, outputs, and state roots, linked to the previous receipt via `previousReceiptHash`. The chain is tamper-evident: modifying any receipt invalidates all subsequent hashes.

*Implementation: `ui/runtime/engine.js:buildReceipt()`*

### 2.4 Capability-Based Security

Applications declare required capabilities in their manifests. The kernel exposes only the declared capabilities through a scoped API. Undeclared capability access throws at runtime. The capability taxonomy is frozen at boot and versioned.

*Implementation: `ui/runtime/engine.js:buildScopedApi()`, `capabilityGuard()`*

### 2.5 Modular dApp Architecture

Applications are self-contained directories with a `manifest.json` (identity, intents, capabilities) and an `index.js` (exported `async run()` function). The kernel discovers, routes, and executes dApps uniformly.

*Implementation: `ui/runtime/router.js`, `ui/dapps/*/`*

### 2.6 Sovereign Digital Identity

Each identity is an Ed25519 keypair generated locally, with the private key encrypted via AES-256-GCM and stored in a per-identity directory. Multiple identities are supported with runtime switching.

*Implementation: `src-tauri/src/identity.rs`, `src-tauri/src/crypto.rs`*

### 2.7 Offline-First Functionality

FRAME operates without network connectivity. Federation, peer discovery, and network requests are optional capabilities. Core execution, storage, and receipt generation require no external dependencies.

### 2.8 AI-Native Operating System Design

Natural language intents are first-class inputs. The AI subsystem (`ai.parse_intent` capability) translates user language into structured intents routed through the same deterministic kernel. AI is a regular dApp — not a privileged subsystem.

*Implementation: `ui/dapps/ai/index.js`, `ui/dapps/frame-intent-en/index.js`*

---

## 3. System Architecture

FRAME is organized in four layers: Backend (Rust/Tauri), Runtime (JavaScript kernel), System (JavaScript services), and UI (HTML/CSS/dynamic rendering).

*See: [docs/assets/architecture.mmd](assets/architecture.mmd)*

### 3.1 Backend Layer (Rust/Tauri)

The Tauri backend provides native OS integration and cryptographic operations:

| Module | File | Responsibility |
|---|---|---|
| Identity | `src-tauri/src/identity.rs` | Ed25519 keypair generation, signing, identity switching |
| Crypto | `src-tauri/src/crypto.rs` | AES-256-GCM encryption/decryption, key generation |
| Storage | `src-tauri/src/storage.rs` | Per-identity encrypted key-value storage |
| Receipt Log | `src-tauri/src/receipt_log.rs` | Append-only receipt persistence |
| Journal | `src-tauri/src/journal.rs` | Persistent event journal |
| LLM Host | `src-tauri/src/llm_host.rs` | LLM integration bridge |
| Commands | `src-tauri/src/lib.rs` | Tauri command handlers exposed to frontend |

The backend exposes commands to the frontend via Tauri's IPC bridge (`window.__FRAME_INVOKE__`), including:
- `create_identity`, `switch_identity`, `list_identities`, `get_current_identity`
- `sign_data`, `get_identity_public_key`, `verify_signature`
- `storage_read`, `storage_write`, `storage_list_keys`
- `receipt_append`, `receipt_list`
- `list_dapps`

### 3.2 Runtime Layer

The runtime layer implements the deterministic execution kernel:

| Module | File | Global | Responsibility |
|---|---|---|---|
| Canonical | `runtime/canonical.js` | `_FRAME_CANONICAL` | Deterministic JSON serialization and SHA-256 hashing |
| Engine | `runtime/engine.js` | `FRAME` | Kernel: `handleIntent`, receipts, capabilities, mode system |
| Router | `runtime/router.js` | `FRAME_ROUTER` | Intent → dApp resolution and execution |
| Storage | `runtime/storage.js` | `FRAME_STORAGE` | Per-identity encrypted storage API |
| Identity | `runtime/identity.js` | `FRAME_IDENTITY` | Frontend identity management |
| State Root | `runtime/stateRoot.js` | `FRAME_STATE_ROOT` | Deterministic state root computation |
| Federation | `runtime/federation.js` | — | Peer federation, receipt sync, trusted peers |
| Invariants | `runtime/invariants.js` | — | Boot-time structural verification |
| Event Bus | `runtime/eventBus.js` | `FRAME_EVENT_BUS` | Pub/sub event system |
| Replay | `runtime/replay.js` | — | Receipt chain replay and verification |
| Snapshot | `runtime/snapshot.js` | — | State snapshot export/import with hash verification |
| Contracts | `runtime/contracts.js` | — | Contract system |

### 3.3 System Layer

Over 80 system modules provide higher-level services:

| Category | Key Modules |
|---|---|
| AI | `aiRuntime.js`, `aiBuilder.js`, `aiCommandRouter.js`, `aiPipeline.js` |
| Agents | `agentRuntime.js`, `agentEngine.js`, `agentEvolution.js`, `agentPersistence.js` |
| Capsules | `capsule.js`, `capsuleExecutor.js`, `capsuleScheduler.js`, `capsuleStore.js` |
| Capabilities | `capabilityRegistry.js`, `capabilityPermissions.js`, `capabilityTable.js` |
| Network | `network.js`, `p2pTransport.js`, `peerDiscovery.js`, `peerReputation.js` |
| UI | `themeManager.js`, `layoutManager.js`, `layoutEngine.js`, `widgetManager.js` |
| Context | `contextEngine.js`, `contextMemory.js`, `contextAnalyzer.js` |
| Economy | `creditLedger.js`, `walletManager.js`, `intentMarket.js` |

**Capsule System.** Capsules are portable, signable, hashable execution units. Each capsule declares a requested capability, payload, payment rule, privacy rule (`public`, `restricted`, `encrypted`), and execution policy. Capsules are SHA-256 hashed and Ed25519 signed. The executor validates signatures, capability availability, payment rules, and privacy restrictions before routing through `FRAME.handleIntent()`. Capsules support chained execution (`next` array) with a maximum chain depth of 5.

**AI Runtime.** The AI subsystem supports three engine backends in priority order: HTTP endpoint (custom LLM API with Bearer auth), WebLLM (`Llama-3.1-8B-Instruct-q4f16_1-MLC`), and Transformers.js (`Xenova/LaMini-Flan-T5-248M`). All engines parse model output into structured intent JSON via regex extraction.

**Agent Runtime.** Autonomous agents register with capability arrays and event callbacks. The scheduler ticks every 2 seconds, dispatching events to enabled agents. Agents are throttled to 20 actions per minute per agent. The compute agent evaluates tasks, submits bids via the intent market, and tracks reputation.

**Context Memory.** Three-tier architecture: short-term RAM (50 entries), rolling localStorage (200 events, 10MB max), and remote archive pointer. Events older than 24 hours are compressed into summaries. Event messages are capped at 200 characters.

**Credit Ledger.** Tracks balance, minted, spent, and burned credits. Mint evaluation includes anti-farming checks: intent must be completed and claimed via market, publisher and solver must be different nodes and fingerprints, and capabilities cannot appear more than twice (anti-cycling). Rewards are weighted by reputation score and task complexity.

### 3.4 UI Layer

The shell (`ui/index.html`) provides structural regions — header, sidebar, workspace, dock, status bar, dynamic regions — populated entirely at runtime by dApp-produced widget schemas. No content is hardcoded.

Themes are JSON files (`ui/themes/*.json`) applied via CSS custom properties. Layouts are JSON files (`ui/layouts/*.json`) defining region visibility and widget placement.

### 3.5 Boot Sequence

FRAME follows a strict boot order enforced by script loading in `index.html`:

1. **Canonical primitives** — `canonicalize()`, `sha256()`, `canonicalHash()`
2. **Logger and Event Bus** — structured logging, pub/sub
3. **Storage** — per-identity encrypted key-value API
4. **System modules** — UI state, error handling, telemetry, accessibility
5. **Identity and State Root** — Ed25519 identity, deterministic state root
6. **Engine (Kernel)** — `handleIntent`, capability schema, receipt construction
7. **Router** — intent resolution, manifest loading, dApp execution
8. **Federation and Network** — peer sync, transport layers
9. **System services** — capsules, agents, AI runtime, themes, layouts
10. **Widget Renderer** — ES module for DOM rendering
11. **FRAME_API** — frozen public API surface
12. **App** — final boot orchestration, identity provisioning

The invariants module runs structural verification at boot, checking capability versions, runtime freeze state, and global allowlists.

---

## 4. Deterministic Execution Model

### 4.1 The Kernel Pipeline

Every user action flows through a single function: `handleIntent()` in `ui/runtime/engine.js`. The pipeline consists of nine stages:

*See: [docs/assets/execution-pipeline.mmd](assets/execution-pipeline.mmd)*

**Stage 1: Normalization.** The kernel normalizes raw input into a structured intent object with `action`, `raw`, `payload`, `target`, and `timestamp` fields. Natural language inputs are routed through the AI dApp for parsing.

**Stage 2: Validation.** The intent must have a valid `action` string, a numeric `timestamp`, and a `raw` string. Malformed intents produce a kernel receipt with reason `invalid_intent`.

**Stage 3: Identity Resolution.** The kernel retrieves the current identity via Tauri IPC. Without an active identity, only `identity.create` intents proceed.

**Stage 4: Router Resolution.** The router resolves the intent's `action` to a registered dApp by matching against dApp manifests. Unresolved intents produce a kernel receipt with reason `no_match`.

**Stage 5: Mode Check.** In `safe` mode (triggered by integrity violations), dApp execution is blocked. In `replay` mode, execution is deferred to the replay system.

**Stage 6: Permission Verification.** For write capabilities (`storage.write`, `bridge.mint`, `bridge.burn`, `wallet.send`, `messages.send`, `contacts.import`), the kernel checks per-identity per-dApp permission grants.

**Stage 7: Integrity Verification.** Two checks:
- **State root integrity** — verifies current state root matches the last receipt's `nextStateRoot`.
- **Code hash integrity** — verifies the dApp's source code SHA-256 matches the boot-time snapshot.

If either fails, the kernel enters safe mode.

**Stage 8: Sandboxed Execution.** The kernel:
1. Computes `previousStateRoot` via `FRAME_STATE_ROOT.computeStateRoot()`
2. Installs a deterministic sandbox (replacing `Date.now()`, `Math.random()`, etc.)
3. Builds a scoped API (`buildScopedApi`) exposing only declared capabilities
4. Calls `router.execute(match, intent, buildScopedApi)`
5. The router invokes the dApp's `run(intent, ctx)` with the scoped API
6. Restores the sandbox after execution

**Stage 9: Receipt Construction and Atomic Commit.** After execution:
1. Computes `nextStateRoot`
2. Builds the receipt with `buildReceipt()` (canonicalizes inputs, computes hashes)
3. Signs the receipt hash with the identity's Ed25519 key via Tauri IPC
4. Atomically commits state writes and receipt via the Tauri `storage_transaction_commit` command — this ensures that storage mutations and the receipt that records them are written together or not at all
5. Invalidates the state root cache for the identity

### 4.2 Canonical JSON Serialization

Determinism requires identical serialization for identical data. FRAME implements a deterministic canonical JSON scheme rather than relying on external standards such as RFC 8785. The implementation resides in `ui/runtime/canonical.js`:

```
function canonicalize(value):
    if value is null: return null
    if value is array: return value.map(canonicalize)
    if value is object:
        sortedKeys = Object.keys(value).sort()
        result = {}
        for key in sortedKeys:
            skip if value[key] is undefined
            skip if value[key] is NaN or ±Infinity
            result[key] = canonicalize(value[key])
        return result
    return value  // primitives pass through
```

This ensures:
- Keys are alphabetically sorted at every nesting level
- `undefined` values are omitted (not serializable)
- `NaN` and `Infinity` are omitted (non-deterministic)
- Recursive canonicalization at all depths

### 4.3 Deterministic Sandbox

During dApp execution, the kernel replaces non-deterministic browser APIs:

| API | Replacement |
|---|---|
| `Date.now()` | Receipt timestamp (normalized to seconds) |
| `new Date()` | Fixed date from receipt timestamp |
| `Math.random()` | XORSHIFT4 PRNG seeded from `previousReceiptHash` |
| `setTimeout/setInterval` | Blocked (throws) |
| `requestAnimationFrame` | Blocked (throws) |
| `queueMicrotask` | Blocked (throws) |
| `WebSocket/EventSource` | Blocked (throws) |
| `fetch` | Sandboxed — allows only `ipc:`, `tauri:`, `localhost`, and relative paths |

The sandbox tracks nesting depth (`_sandboxDepth`) to handle re-entrant calls safely. It is installed before execution and restored immediately after, ensuring non-execution code operates normally.

*Implementation: `ui/runtime/engine.js:installDeterministicSandbox()`*

### 4.4 Mode System

The kernel operates in four modes:

| Mode | Description | Transitions |
|---|---|---|
| `normal` | Standard execution | → `safe`, `replay` |
| `safe` | Integrity violation detected, dApp execution blocked | → `normal` (after reset) |
| `replay` | Receipt chain replay active | → `normal`, `safe` |
| `restoring` | State restoration in progress | → `normal`, `safe` |

Mode transitions are strictly controlled — only valid transitions are accepted.

---

## 5. Cryptographic Foundations

### 5.1 Hash Function: SHA-256

All hashing in FRAME uses SHA-256 via the Web Crypto API (`crypto.subtle.digest`):

```
sha256(str) → hex string (64 characters)
```

Applied to:
- Receipt hashes: `receiptHash = SHA-256(JSON.stringify(canonicalize(signablePayload)))`
- Input hashes: `inputHash = SHA-256(JSON.stringify(canonicalize(inputPayload)))`
- Result hashes: `resultHash = SHA-256(JSON.stringify(canonicalize(result)))`
- State roots: `stateRoot = SHA-256(JSON.stringify(canonicalize(statePayload)))`
- Code hashes: `codeHash = SHA-256(dAppSourceCode)`
- Core hash: `coreHash = SHA-256(concatenation of all runtime module sources)`

*Implementation: `ui/runtime/canonical.js:sha256()`*

### 5.2 Digital Signatures: Ed25519

FRAME uses Ed25519 (Edwards-curve Digital Signature Algorithm) for identity and receipt signing:

- **Key generation:** `ed25519-dalek` crate with `OsRng` (OS-level CSPRNG)
- **Key size:** 32-byte private key, 32-byte public key
- **Signature size:** 64 bytes
- **Signing:** Private key signs the `receiptHash` (hex-encoded SHA-256)
- **Verification:** Public key verifies signature against receipt hash

```rust
// src-tauri/src/crypto.rs
pub fn ed25519_generate() -> (SigningKey, VerifyingKey) {
    let mut csprng = OsRng;
    let signing_key = SigningKey::generate(&mut csprng);
    let verifying_key = signing_key.verifying_key();
    (signing_key, verifying_key)
}
```

Private keys are serialized as `IdentityKeys { secret_key: [u8; 32], public_key: [u8; 32] }`, encrypted with AES-256-GCM, and stored as `keys.enc` in the identity directory.

### 5.3 Symmetric Encryption: AES-256-GCM

Per-identity storage encryption uses AES-256-GCM:

- **Key size:** 256 bits (32 bytes), generated via `getrandom`
- **Nonce:** 96 bits (12 bytes), randomly generated per encryption
- **Storage format:** `nonce (12 bytes) || ciphertext`
- **Authentication:** GCM provides authenticated encryption (AEAD)

```rust
// src-tauri/src/crypto.rs
pub fn aes_gcm_encrypt(key: &[u8; 32], plaintext: &[u8]) -> Result<Vec<u8>> {
    let cipher = Aes256Gcm::new_from_slice(key)?;
    let mut nonce = [0u8; 12];
    getrandom(&mut nonce)?;
    let ciphertext = cipher.encrypt((&nonce).into(), plaintext)?;
    // Returns: nonce || ciphertext
}
```

Each identity has a unique 256-bit storage key stored as `.key` in the identity directory. All storage operations for that identity are encrypted with this key. Storage files are written using a crash-safe write-ahead pattern: data is written to a `.tmp` file, fsynced to disk, then atomically renamed to the final path — guaranteeing no partial writes are visible.

**Identity directory structure:**
```
identities/
  id_<timestamp>/
    .key              — 32-byte AES-256-GCM storage key
    keys.enc          — Encrypted Ed25519 keypair (IdentityKeys JSON)
    meta.json         — Identity metadata and capabilities
    storage/          — Encrypted key-value files (<basename>.enc)
    event.log         — Append-only event log (JSON-lines, max 5MB, 3 rotations)
  current_identity    — File containing active identity ID
```

### 5.4 Receipt Hash Construction

The receipt hash is computed over a canonical subset of receipt fields (the "signable payload"):

```
signablePayload = {
    version, timestamp, identity, dappId, intent,
    inputHash, inputPayload, capabilitiesDeclared, capabilitiesUsed,
    previousStateRoot, nextStateRoot, resultHash,
    previousReceiptHash, executionTraceHash (if present)
}

receiptHash = SHA-256(JSON.stringify(canonicalize(signablePayload)))
signature = Ed25519.sign(hex_decode(receiptHash), privateKey)
```

The `signature` and `publicKey` fields are excluded from the signable payload to avoid circular dependency.

### 5.5 Core Integrity Hash

At boot, the kernel computes a SHA-256 hash over the concatenated source of all 12 core runtime modules:

```
coreHash = SHA-256(canonical.js + router.js + engine.js + storage.js +
                   identity.js + stateRoot.js + backup.js + invariants.js +
                   replay.js + federation.js + contracts.js + snapshot.js)
```

If this hash differs from the expected value (`EXPECTED_CORE_HASH`), the kernel enters safe mode, blocking all dApp execution.

---

## 6. Capability-Based Security Model

### 6.1 Capability Schema

FRAME enforces a frozen taxonomy of 37 capabilities organized across 17 namespaces:

*See: [docs/assets/capability-model.mmd](assets/capability-model.mmd)*

| Namespace | Capabilities | Count |
|---|---|---|
| `ai` | `parse_intent`, `chat`, `summarize`, `explain`, `plan`, `analyze` | 6 |
| `bridge` | `burn`, `mint` | 2 |
| `contacts` | `read`, `import` | 2 |
| `context` | `query`, `archive`, `search`, `summarize` | 4 |
| `display` | `timerTick` | 1 |
| `execution` | `feed` | 1 |
| `identity` | `read` | 1 |
| `messages` | `send`, `list` | 2 |
| `network` | `request` | 1 |
| `query` | `explorer` | 1 |
| `receipt` | `inspect` | 1 |
| `storage` | `read`, `write` | 2 |
| `system` | `assistant.open`, `overview` | 2 |
| `timeline` | `view`, `summary` | 2 |
| `ui` | `layout.apply`, `theme.apply`, `widget.render`, `plan`, `preview` | 5 |
| `wallet` | `send`, `read`, `balance` | 3 |
| `weather` | `current` | 1 |

The schema is `Object.freeze()`'d at boot. `CAPABILITY_VERSION = 3`.

### 6.2 Write Capability Separation

Only 6 of 37 capabilities can modify persistent state:

```javascript
var WRITE_CAPABILITIES = Object.freeze([
    'bridge.burn', 'bridge.mint', 'storage.write',
    'wallet.send', 'contacts.import', 'messages.send'
]);
```

Write capabilities require explicit per-identity per-dApp permission grants. Read and UI capabilities are granted implicitly.

### 6.3 Scoped API Construction

When the kernel executes a dApp, it constructs a scoped API object containing only the methods for declared capabilities:

```
buildScopedApi(dappId, declaredCapabilities) → { api, declaredCapabilities }
```

Each method is wrapped with `capabilityGuard()`, which:
1. Verifies the `dappId` is a non-empty string
2. Verifies the `requestedCapability` is declared in the manifest
3. Logs the capability to `executionCapLog` for receipt recording
4. Throws `'Capability denied'` on violation

The resulting API is `Object.freeze()`'d — dApps cannot add methods or modify existing ones.

During execution, the router hides runtime globals (`FRAME_STORAGE`, `FRAME_IDENTITY`, etc.) from the global scope. dApps can only access these services through the scoped API, preventing direct manipulation of runtime state. Globals are restored after execution completes.

### 6.4 Manifest Validation

At registration, every dApp's capabilities are validated against `CAPABILITY_SCHEMA`:

```javascript
function validateCapabilities(caps) {
    if (!Array.isArray(caps)) return false;
    for (var i = 0; i < caps.length; i++) {
        if (typeof caps[i] !== 'string') return false;
        if (!(caps[i] in CAPABILITY_SCHEMA)) return false;
    }
    return true;
}
```

Unknown capabilities are rejected — a dApp cannot declare a capability that does not exist in the frozen schema.

### 6.5 Example: Wallet dApp Manifest

```json
{
    "name": "Wallet",
    "id": "wallet",
    "version": "1.0.0",
    "intents": ["wallet.send", "wallet.balance"],
    "capabilities": ["wallet.send", "wallet.read", "wallet.balance", "storage.read"]
}
```

This dApp can access `wallet.send`, `wallet.read`, `wallet.balance`, and `storage.read` — nothing else. Attempting to call `storage.write` or any other capability will throw.

---

## 7. Federation and Synchronization

### 7.1 Overview

FRAME supports peer-to-peer federation — nodes can exchange receipt chains to synchronize state. Federation is optional and authenticated.

*See: [docs/assets/federation.mmd](assets/federation.mmd)*

### 7.2 Trusted Peer Registry

Peers are identified by their Ed25519 public keys. Trust is explicitly managed:

| API | Description |
|---|---|
| `addTrustedPeer(peerId)` | Add a peer to the trusted registry |
| `removeTrustedPeer(peerId)` | Remove a peer from the trusted registry |
| `isTrustedPeer(peerId)` | Check if a peer is trusted |
| `listTrustedPeers()` | List all trusted peer IDs |

The registry is persisted in local storage.

### 7.3 Strict Mode

In production (`FRAME_DEV_MODE === false`), federation strict mode is enabled by default:

```javascript
var _federationStrictMode =
    (typeof window !== 'undefined' && window.FRAME_DEV_MODE) ? false : true;
```

When strict mode is active, `importReceiptChain()` rejects receipts from peers not in the trusted registry. This prevents untrusted state injection.

### 7.4 Receipt Synchronization Protocol

1. **Discovery** — Peers announce availability via transport layer (LAN, WebRTC, Bluetooth)
2. **Negotiation** — Requesting peer identifies which receipts it is missing
3. **Transfer** — Receipt chain is transmitted
4. **Verification** — Receiving peer verifies:
   - Ed25519 signature on each receipt
   - Chain link integrity (`previousReceiptHash`)
   - State root continuity (`previousStateRoot` → `nextStateRoot`)
5. **Import** — Verified receipts are appended to local chain

### 7.5 Transport Layers

FRAME supports three transport mechanisms:

| Transport | Module | Use Case |
|---|---|---|
| LAN | `runtime/network/transport-lan.js` | Local network peers |
| WebRTC | `runtime/network/transport-webrtc.js` | Browser-to-browser |
| Bluetooth | `runtime/network/transport-bluetooth.js` | Short-range devices |

All transports use Ed25519 message signing (`runtime/network/message-signing.js`) for authentication.

---

## 8. State Roots, Snapshots, and Replay

### 8.1 State Root Computation

The state root is a single SHA-256 hash representing the entire system state. It is computed by `FRAME_STATE_ROOT.computeStateRoot()` in `ui/runtime/stateRoot.js`.

*See: [docs/assets/state-root.mmd](assets/state-root.mmd)*

The state root payload contains six fields (alphabetically ordered, versioned):

```javascript
var STATE_ROOT_KEYS = Object.freeze([
    'capabilityVersion',
    'identityPublicKey',
    'installedDApps',
    'receiptChainCommitment',
    'storage',
    'version'
]);
```

**Computation algorithm:**

```
1. Gather identity public key (Ed25519 hex)
2. Load installed dApps (sorted by id):
   - For each: { id, manifest (canonicalized), codeHash (SHA-256 of index.js) }
3. Gather storage data (all keys except excluded prefixes, sorted)
4. Compute receipt chain commitment:
   - If chain is empty: SHA-256('GENESIS')
   - Otherwise: receiptHash of the last receipt in chain
   (This breaks the circular dependency: receipt[n] cannot include the
    state root that itself depends on receipt[n])
5. Assemble payload:
   { version: 3, capabilityVersion: 3, identityPublicKey,
     installedDApps, storage, receiptChainCommitment }
6. Deep-sort all nested objects
7. Canonicalize the payload
8. stateRoot = SHA-256(JSON.stringify(canonicalPayload))
```

**Excluded storage prefixes** (not part of state root — ephemeral or already represented):
- `chain:` (already in `receiptChainCommitment`)
- `frame_integrity`, `frame_context_`, `frame_layout`, `frame_widgets`
- `frame_capsules`, `frame_logs`, `frame_ui`

**Included:** `frame_capability_grants` — capability grants affect deterministic execution and must be part of the state root.

### 8.2 State Root Self-Test

At boot, `FRAME_STATE_ROOT.selfTest()` computes the state root twice and verifies both computations produce identical results, confirming determinism. In production, a mismatch throws a fatal error.

### 8.3 Snapshot Export and Import

The snapshot system (`ui/runtime/snapshot.js`) enables full state export and import:

**Export:** Serializes the current state (storage, receipt chain, dApp state) into a JSON snapshot with a SHA-256 integrity hash.

**Import:** Before applying a snapshot, the system verifies the declared hash against the computed hash of the snapshot data. Hash mismatches are rejected, preventing state injection attacks.

### 8.4 Deterministic Replay

The replay system (`ui/runtime/replay.js`) reconstructs system state from a receipt chain:

1. Set kernel mode to `replay`
2. For each receipt in chain order:
   a. Set execution timestamp to receipt timestamp
   b. Re-execute the dApp with the receipt's `inputPayload`
   c. Verify the result hash matches the receipt's `resultHash`
   d. Verify the state root matches the receipt's `nextStateRoot`
3. Restore kernel mode to `normal`

If any verification step fails, the replay identifies the divergence point, enabling forensic analysis.

**Guarantee:** Given an identical receipt chain and identical dApp code (verified via `codeHash`), replay produces an identical state root.

---

## 9. dApp Ecosystem

### 9.1 Architecture

Every dApp is a self-contained directory with two files:

```
dapp-name/
  manifest.json    — identity, intents, capabilities
  index.js         — exports async run(intent, ctx)
```

The manifest declares what the dApp is; the index implements what it does.

### 9.2 Manifest Structure

```json
{
    "name": "Human-readable name",
    "id": "namespace.identifier",
    "version": "semver",
    "intents": ["namespace.action1", "namespace.action2"],
    "capabilities": ["storage.read", "storage.write"],
    "capabilitySchemas": [
        {
            "id": "namespace.action1",
            "type": "ui|query|mutation",
            "description": "What this intent does",
            "input": {},
            "output": { "widgetSchema": "object" }
        }
    ]
}
```

### 9.3 Execution Contract

Every dApp must:
1. Export `async function run(intent, ctx)` as a named export
2. Return a result object (plain object, not a class instance)
3. Use `type: 'error'` for failure results
4. Include `widgetSchema` for UI rendering
5. Only access capabilities declared in the manifest
6. Use `FRAME_STORAGE` for persistent data (not raw `localStorage`)

### 9.4 Widget Schema

dApps produce UI by returning a `widgetSchema` in their result:

```javascript
return {
    message: 'Operation complete',
    widgetSchema: {
        type: 'ui.plan',
        widgets: [
            { type: 'stat', title: 'Balance', region: 'workspace', value: '42 frame' },
            { type: 'card', title: 'Recent', region: 'sidebar', source: 'wallet' }
        ]
    }
};
```

Widget types: `card`, `stat`, `chart`, `list`, `table`. Regions: `workspace`, `sidebar`, `dock`, `top`, `left`, `center`, `right`, `bottom`.

### 9.5 Installed dApps (v2.2.0)

FRAME ships with 34 dApps across multiple categories:

| Category | dApps |
|---|---|
| Core | `wallet`, `notes`, `messages`, `contacts`, `timer` |
| AI | `ai`, `ai.assistant`, `frame-intent-en` |
| System | `systemOverview`, `systemtools`, `assistant.panel` |
| Development | `dev`, `dev.monitor`, `dev.system`, `dev.telemetry` |
| Visualization | `timeline.view`, `execution.feed`, `query.explorer`, `receipt.inspect` |
| Infrastructure | `bridge`, `compute`, `contracts`, `agent`, `agentmanager` |
| Context | `context`, `contextArchive`, `memory` |
| UI | `theme`, `layout`, `ui.home`, `ui.planner`, `web`, `maps`, `weather` |

### 9.6 dApp Installation

Remote dApps can be installed via `FRAME_API.installDapp(url)`. The installer:
1. Fetches the dApp archive from the URL
2. Validates the manifest structure
3. Validates all declared capabilities against `CAPABILITY_SCHEMA`
4. Computes the code hash for integrity tracking
5. Registers the dApp with the router

---

## 10. FRAME SDK

### 10.1 Overview

The FRAME SDK (`sdk/frame-sdk/`) provides a stable API surface for third-party dApp developers. It abstracts internal runtime modules behind a versioned, typed interface.

### 10.2 Usage

```javascript
import { FrameSDK } from '@frame-os/sdk';
const sdk = FrameSDK.create('my.dapp');

// Execute intents
await sdk.callIntent('notes.list', {});

// Per-identity encrypted storage (auto-prefixed with dApp ID)
await sdk.storageWrite('settings', { theme: 'dark' });
const settings = await sdk.storageRead('settings');

// Create widgets
sdk.createWidget({ type: 'card', title: 'Status', region: 'sidebar' });

// Event bus
sdk.on('receipt:created', (receipt) => { /* ... */ });

// Query receipts
const receipts = sdk.queryReceipts({ dappId: 'my.dapp', limit: 10 });
```

### 10.3 API Surface

| Method | Description |
|---|---|
| `callIntent(action, payload)` | Execute an intent through the kernel |
| `createWidget(options)` | Create a widget in the shell |
| `removeWidget(widgetId)` | Remove a widget by ID |
| `openPanel(widgetId)` | Open a widget in the workspace panel |
| `storageRead(key)` | Read from per-identity encrypted storage |
| `storageWrite(key, value)` | Write to per-identity encrypted storage |
| `on(event, handler)` | Subscribe to event bus |
| `off(event, handler)` | Unsubscribe from event bus |
| `emit(event, data)` | Emit an event |
| `setTheme(theme)` | Apply a theme |
| `setLayout(layout)` | Apply a layout |
| `getDappState()` | Get dApp state from kernel |
| `setDappState(state)` | Set dApp state in kernel |
| `registerCapability(manifest)` | Register capability manifest |
| `queryReceipts(filter)` | Query receipt chain |
| `uiHint(result, type)` | Wrap result with UI hint |

### 10.4 TypeScript Definitions

Full TypeScript definitions are provided in `sdk/frame-sdk/index.d.ts`, covering all interfaces: `WidgetOptions`, `Widget`, `DappManifest`, `CapabilitySchema`, `IntentResult`, `ReceiptFilter`, `FrameSDKInstance`, `FrameSDKStatic`.

### 10.5 Templates and Examples

- **dApp template** (`templates/dapp-template/`) — Minimal manifest + index.js skeleton
- **Counter example** (`examples/counter-dapp.js`) — Demonstrates state management, storage, event bus

---

## 11. Accessibility and Human-Centered Design

### 11.1 WCAG 2.1 AA Compliance

FRAME's shell meets WCAG 2.1 Level AA accessibility standards:

**Semantic HTML:**
- `<main>` for workspace (proper landmark)
- `<header>`, `<footer>`, `<aside>` for shell regions
- `<h2>` for section headings
- `<label>` for form inputs

**ARIA Attributes:**
- `role="application"` on shell root
- `role="search"` on command bar
- `role="status"` with `aria-live="polite"` on status areas
- `role="log"` with `aria-live="polite"` on activity feed
- `role="dialog"` on modals
- `role="list"` on notification and activity lists
- `aria-label` on all interactive elements

**Keyboard Navigation:**
- Skip-to-content link ("Skip to intent input")
- `:focus-visible` indicators (2px accent outline + 4px box-shadow)
- Logical tab order matching visual layout
- No keyboard traps

**Media Queries:**
- `prefers-reduced-motion: reduce` — disables all animations and transitions
- `prefers-contrast: high` — enhances border and text colors

### 11.2 Color Contrast

All themes meet WCAG AA contrast ratios:
- Normal text: minimum 4.5:1
- Large text: minimum 3:1
- UI components: minimum 3:1

The Matrix theme's secondary text was adjusted from `#008f28` (3.2:1) to `#00b333` (4.5:1) to meet requirements.

---

## 12. Performance and Observability

### 12.1 Telemetry Dashboard

The `dev.telemetry` dApp (developer mode only) provides real-time performance metrics:

| Metric | Source | Method |
|---|---|---|
| FPS | `requestAnimationFrame` sampling | Frames counted per second |
| Widget Render (ms) | `widget:render:start/end` events | Performance.now() delta |
| Intent Execution (ms) | `intent:start/end` events | Performance.now() delta |
| Memory (MB) | `performance.memory.usedJSHeapSize` | Sampled every 2 seconds |
| Receipt Throughput | `receipt:created` events | Count per second |
| State Root (ms) | `stateroot:computed` events | Duration from event |
| Network Latency (ms) | `peer:latency` events | Round-trip time |
| Determinism Check (ms) | `determinism:check` events | Verification duration |

Each metric tracks current value, rolling average, min, max, and sample count (60-sample window).

### 12.2 State Root Caching

State root computation is cached per-identity. Cache invalidation occurs when:
- Storage hash changes (SHA-256 over canonicalized storage)
- Receipt chain commitment changes

This avoids redundant full-state recomputation on every intent.

### 12.3 Execution Tracing

When enabled, the kernel records an execution trace — a sequence of `{ op, data, timestamp }` entries capturing every capability invocation. The trace is canonicalized and hashed as `executionTraceHash` in the receipt, enabling execution proof verification without replaying the full operation.

---

## 13. Use Cases

### 13.1 Sovereign Identity Systems

FRAME's Ed25519 identity model, combined with per-identity encrypted storage and receipt-based audit trails, provides a foundation for self-sovereign identity. Users control their cryptographic keys locally, sign every action, and can prove their history through the receipt chain.

### 13.2 Decentralized AI Agents

The agent subsystem (`agentRuntime.js`, `agentEngine.js`, `agentEvolution.js`) enables autonomous agents that execute through the same capability-scoped kernel. Every agent action produces a receipt, ensuring agent behavior is auditable and deterministic.

### 13.3 Verifiable Governance

Organizations can use FRAME's receipt chain as an immutable governance log. Every decision, vote, or policy change flows through the kernel and produces a signed receipt. The deterministic replay capability enables independent audit.

### 13.4 Secure Enterprise Workflows

FRAME's capability-based security model enforces least privilege at the application level. Enterprise workflows can be modeled as dApps with strictly scoped capabilities, ensuring employees access only authorized resources.

### 13.5 Scientific Reproducibility

Deterministic execution guarantees that computational experiments produce identical results when replayed. Researchers can share receipt chains as provably correct computation records.

### 13.6 Offline-First Computing

FRAME's local-first architecture operates without network connectivity. Federation enables eventual synchronization when networks are available, making it suitable for field operations, disaster response, and resource-constrained environments.

### 13.7 Web3 Infrastructure

The wallet dApp, bridge capabilities (`bridge.mint`, `bridge.burn`), and token adapter system (`ethereumAdapter.js`, `solanaAdapter.js`, `bitcoinAdapter.js`) provide integration points with blockchain ecosystems.

---

## 14. Comparison with Existing Platforms

| Property | Traditional OS | Cloud Platforms | Blockchain | FRAME |
|---|---|---|---|---|
| Deterministic Execution | No | No | Yes | Yes |
| User Sovereignty | Partial | No | Yes | Yes |
| Capability-Based Security | Limited | Partial | Limited | Yes (37 frozen) |
| Verifiable State | No | No | Yes | Yes |
| Receipt/Audit Chain | No | Partial (logs) | Yes (blocks) | Yes (receipts) |
| Offline Operation | Yes | No | Limited | Yes |
| Local-First | Yes | No | Partial | Yes |
| Cryptographic Identity | No | No | Yes (wallets) | Yes (Ed25519) |
| Deterministic Replay | No | No | Partial | Yes |
| Dynamic UI from State | No | No | No | Yes |
| AI-Native | No | Partial | No | Yes |
| Per-Identity Encryption | No | Partial | No | Yes (AES-256-GCM) |

**Key differentiators:**

- Unlike blockchains, FRAME executes locally with no consensus overhead, gas fees, or network latency.
- Unlike cloud platforms, FRAME keeps all data on-device with user-controlled encryption.
- Unlike traditional operating systems, every state transition in FRAME is cryptographically signed and deterministically reproducible.

---

## 15. Governance and Future Roadmap

### 15.1 Current Status (v2.2.0)

- 37 capability taxonomy (frozen, versioned)
- 34 installed dApps
- 4 themes, 4 layouts
- Ed25519 + AES-256-GCM cryptography
- Federation with trusted peer registry
- WCAG 2.1 AA accessibility
- FRAME SDK for third-party developers
- Performance telemetry dashboard

### 15.2 Planned Enhancements

**Advanced Federation:**
- Merkle proof-based partial state sync
- Conflict resolution protocols
- Cross-node capability negotiation

**Hardware-Backed Identity:**
- TPM/Secure Enclave integration for key storage
- Hardware attestation for receipt signing
- Biometric identity binding

**Distributed Compute Capsules:**
- Sandboxed compute containers distributed across peers
- Capability-scoped resource allocation
- Deterministic compute verification across nodes

**dApp and Agent Marketplace:**
- Peer-to-peer dApp distribution
- Reputation-scored agent marketplace
- Capability-based pricing and metering

**Expanded AI Integration:**
- Multi-model AI pipeline orchestration
- On-device model inference
- AI-generated dApp creation from natural language specifications

---

## 16. Security Considerations

### 16.1 Threat Model

FRAME's security model assumes:
- The local device is trusted (physical security is the user's responsibility)
- The Tauri backend is correct (Rust memory safety)
- The Web Crypto API provides correct SHA-256 and random number generation
- The OS provides a reliable CSPRNG via `getrandom`

### 16.2 Attack Vectors and Mitigations

| Vector | Mitigation |
|---|---|
| **Capability escalation** | Frozen `CAPABILITY_SCHEMA`, `capabilityGuard()` deny-by-default, scoped API construction |
| **State tampering** | Receipt chain with SHA-256 hashes, Ed25519 signatures, state root verification |
| **Code injection** | Boot-time core hash verification, dApp code hash verification pre-execution |
| **Non-determinism** | Deterministic sandbox (replaces Date.now, Math.random, etc.), canonical JSON |
| **Untrusted federation** | Trusted peer registry, strict mode (default in production) |
| **Storage compromise** | AES-256-GCM per-identity encryption, separate keys per identity |
| **Replay attacks** | Monotonic timestamps, `previousReceiptHash` chain linking |
| **Path traversal** | Router validates module URLs, blocks `..`, `%2e%2e`, and `%252e%252e` encoded traversals |
| **Global injection** | Boot-time `ALLOWED_GLOBALS` check, strict unknown global enforcement |
| **Runtime modification** | `Object.freeze()` on `FRAME`, `CAPABILITY_SCHEMA`, `FRAME_API`, `RECEIPT_FIELDS` |
| **Snapshot injection** | Hash verification before snapshot import |

### 16.3 Trust Assumptions

1. **Tauri IPC is secure** — commands are invoked through the controlled `__FRAME_INVOKE__` bridge
2. **Ed25519 is secure** — based on the `ed25519-dalek` crate, a widely audited implementation
3. **AES-256-GCM is secure** — based on the `aes-gcm` crate with authenticated encryption
4. **The kernel is correct** — the single-path execution model reduces attack surface
5. **Capability schema is complete** — the frozen taxonomy covers all required operations

### 16.4 Execution Trace Sanitization

When recording execution traces, the kernel sanitizes sensitive data:

```javascript
var secretPatterns = ['key', 'secret', 'token', 'password', 'private', 'signature'];
// Secret values are SHA-256 hashed, not stored raw
```

This ensures execution proofs do not leak credentials while remaining verifiable.

---

## 17. Conclusion

FRAME introduces a fundamentally new computing model: a sovereign, deterministic operating system where every operation is verifiable, every state transition is cryptographically sealed, and sovereignty rests entirely with the user.

The key contributions of FRAME are:

1. **A deterministic execution kernel** that canonicalizes all inputs, sandboxes non-deterministic APIs, and produces signed receipts for every operation.

2. **A frozen capability taxonomy** with deny-by-default enforcement, eliminating the possibility of unauthorized resource access.

3. **A receipt chain** that provides an immutable, tamper-evident audit trail of all state transitions, enabling deterministic replay and cross-node verification.

4. **Per-identity encrypted storage** using AES-256-GCM, ensuring user data sovereignty even on shared devices.

5. **A fully dynamic user interface** derived entirely from execution results, with no hardcoded content — enabling deterministic UI reconstruction from receipt chains.

6. **Federated state synchronization** with trusted-peer authentication and strict mode enforcement, enabling secure multi-node operation.

FRAME demonstrates that it is possible to build a computing platform that is simultaneously sovereign, deterministic, verifiable, accessible, and usable — without sacrificing any of these properties for the others.

---

## 18. Appendices

### Appendix A: Protocol Constants

| Constant | Value | Location |
|---|---|---|
| `PROTOCOL_VERSION` | `2.2.0` | `ui/runtime/engine.js:14` |
| `RECEIPT_VERSION` | `2` | `ui/runtime/engine.js:15` |
| `CAPABILITY_VERSION` | `3` | `ui/runtime/engine.js:16` |
| `STATE_ROOT_VERSION` | `3` | `ui/runtime/stateRoot.js:10` |

### Appendix B: Receipt Fields (Alphabetical)

| Field | Type | Description |
|---|---|---|
| `capabilitiesDeclared` | `string[]` | Capabilities declared in dApp manifest |
| `capabilitiesUsed` | `string[]` | Capabilities exercised during execution |
| `dappId` | `string` | Executing dApp identifier |
| `executionTraceHash` | `string\|null` | SHA-256 of execution trace |
| `identity` | `string` | Active identity at execution time |
| `inputHash` | `string` | SHA-256 of canonicalized input |
| `inputPayload` | `object` | Full canonicalized input payload |
| `intent` | `string` | Action string executed |
| `nextStateRoot` | `string` | State root after execution |
| `previousReceiptHash` | `string\|null` | Hash of preceding receipt |
| `previousStateRoot` | `string` | State root before execution |
| `publicKey` | `string\|null` | Ed25519 public key (hex) |
| `receiptHash` | `string` | SHA-256 of signable payload |
| `resultHash` | `string` | SHA-256 of canonicalized result |
| `signature` | `string\|null` | Ed25519 signature (hex) |
| `timestamp` | `number` | Unix timestamp (seconds) |
| `version` | `number` | Receipt schema version (2) |

### Appendix C: State Root Keys (Alphabetical)

| Key | Type | Description |
|---|---|---|
| `capabilityVersion` | `number` | Current capability schema version |
| `identityPublicKey` | `string\|null` | Ed25519 public key (hex) |
| `installedDApps` | `array` | Sorted dApp entries with manifests and code hashes |
| `receiptChainCommitment` | `string` | Rolling hash of receipt chain |
| `storage` | `object` | All non-excluded storage data (sorted) |
| `version` | `number` | State root schema version (3) |

### Appendix D: Capability Schema (v3)

```
ai.analyze          — Analyze data, patterns, or system behavior.
ai.chat             — Conversational AI interaction with context.
ai.explain          — Explain intent execution, receipts, or system state.
ai.parse_intent     — Parse natural language input into a structured intent.
ai.plan             — Generate multi-step execution plans from goals.
ai.summarize        — Summarize text, receipts, or execution results.
assistant.open      — Open or restore the assistant panel.
bridge.burn         — Burn frame from a vault balance, decreasing total supply.  [WRITE]
bridge.mint         — Mint frame to a vault balance, increasing total supply.   [WRITE]
contacts.import     — Import contacts from external source.                     [WRITE]
contacts.read       — List contacts.
context.archive     — Query archived context for historical activity.
context.query       — Query rolling context memory for recent activity.
context.search      — Search context memory by keyword or time range.
context.summarize   — Summarize context activity over a period.
display.timerTick   — Push timer state updates to the display layer.
execution.feed      — Live receipt append stream widget.
identity.read       — Read current identity and list identities.
messages.list       — List messages with a recipient.
messages.send       — Send a message to a recipient.                            [WRITE]
network.request     — Perform a deterministic, receipt-sealed HTTP request.
query.explorer      — Browse and filter receipt query results.
receipt.inspect     — Inspect a single receipt from the chain by hash.
storage.read        — Read from scoped storage by key.
storage.write       — Write to scoped storage by key.                           [WRITE]
system.overview     — System overview dashboard widgets.
timeline.summary    — Compact activity summary for shortcut widgets.
timeline.view       — Show receipt activity timeline visualization.
ui.layout.apply     — Apply a layout configuration to the shell.
ui.plan             — Execute a UI plan with widget schemas.
ui.preview          — Generate intent preview suggestions.
ui.theme.apply      — Apply a theme configuration to the shell.
ui.widget.render    — Render a dynamic widget in a shell region.
wallet.balance      — Read wallet balance (alias).
wallet.read         — Read wallet balance.
wallet.send         — Send a payment to a recipient.                            [WRITE]
weather.current     — Current weather snapshot for home widgets.
```

### Appendix E: Glossary

| Term | Definition |
|---|---|
| **Canonical JSON** | Deterministic JSON serialization with sorted keys and filtered non-serializable values |
| **Capability** | A named permission that a dApp must declare and the kernel enforces |
| **dApp** | A self-contained application with manifest.json and index.js |
| **Federation** | Peer-to-peer receipt chain synchronization |
| **Intent** | A structured user action with action, payload, and timestamp |
| **Kernel** | The central execution engine (`handleIntent`) |
| **Receipt** | A cryptographically signed record of a single execution |
| **Receipt Chain** | An ordered sequence of receipts linked by `previousReceiptHash` |
| **Safe Mode** | Kernel state where dApp execution is blocked due to integrity violation |
| **Scoped API** | A frozen API object containing only the dApp's declared capabilities |
| **State Root** | SHA-256 hash of the canonicalized aggregate system state |
| **Strict Mode** | Federation mode that rejects receipts from untrusted peers |
| **Widget Schema** | A declarative description of UI widgets returned by dApps |

### Appendix F: Diagram Index

| Diagram | File | Description |
|---|---|---|
| System Architecture | `docs/assets/architecture.mmd` | Four-layer system overview |
| Execution Pipeline | `docs/assets/execution-pipeline.mmd` | handleIntent sequence diagram |
| Receipt Chain | `docs/assets/receipt-chain.mmd` | Chain structure with hash linking |
| State Root | `docs/assets/state-root.mmd` | State root computation flow |
| Capability Model | `docs/assets/capability-model.mmd` | Security enforcement flow |
| Federation | `docs/assets/federation.mmd` | Peer synchronization protocol |
| UI Rendering | `docs/assets/ui-rendering.mmd` | Intent → widget rendering pipeline |

---

*All technical claims in this document are derived from the FRAME v2.2.0 source code. No speculative or unimplemented features are described. Source file references are provided for verification.*
