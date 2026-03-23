# FRAME Boot Sequence

This document describes the exact startup order from `index.html` through runtime initialization, integrity lock, and verification checks.

## Boot Sequence Overview

FRAME boot follows a strict sequence:

1. **HTML Load** — `index.html` loads
2. **Runtime Spine** — 12 runtime modules load in order
3. **System Modules** — System modules load
4. **Invariants** — Boot assertions run
5. **Integrity Lock** — Integrity lock initialized
6. **Verification** — Replay determinism check and state reconstruction
7. **Application** — `app.js` loads, UI ready

## Detailed Boot Sequence

### Phase 1: HTML and Setup

**File:** `ui/index.html`

```
1. HTML structure loads
2. CSS loads (style.css)
3. DOM elements created
4. Script setup:
   - window.FRAME_DEV_MODE = false
   - window.__FRAME_INVOKE__ defined (Tauri bridge)
```

### Phase 2: Runtime Spine (12 Modules)

**Files load in strict order:**

#### 1. canonical.js

**Purpose:** Canonicalization and hashing primitives

**Creates:**
- `window._FRAME_CANONICAL` (temporary, removed by invariants.js)
  - `canonicalize(value)` — Sort object keys recursively
  - `sha256(str)` — SHA-256 hash computation
  - `canonicalHash(value)` — Canonical hash computation

**Dependencies:** None

#### 2. storage.js

**Purpose:** Canonical JSON storage wrapper

**Creates:**
- `window.FRAME_STORAGE` (frozen)
  - `read(key)` — Read from storage
  - `write(key, value)` — Write to storage (validates canonical JSON)
  - `clear()` — Clear all storage
  - `validateCanonicalJson(value)` — Validation function

**Dependencies:** `_FRAME_CANONICAL`, `__FRAME_INVOKE__`

#### 3. identity.js

**Purpose:** Identity/vault wrapper

**Creates:**
- `window.FRAME_IDENTITY` (frozen)
  - `list()` — List identities
  - `getCurrent()` — Get current identity
  - `switch(identity_id)` — Switch identity
  - `create()` — Create identity
  - `getCapabilities()` — Get identity capabilities

**Dependencies:** `__FRAME_INVOKE__`

#### 4. stateRoot.js

**Purpose:** Deterministic state root computation

**Creates:**
- `window.FRAME_STATE_ROOT` (frozen)
  - `computeStateRoot(identity)` — Compute state root
  - `computeCodeHash(dappId)` — Compute dApp code hash
  - `loadInstalledDApps()` — Load installed dApps
  - `selfTest()` — Determinism self-test
  - `STATE_ROOT_VERSION` — Version constant (3)
  - `STATE_ROOT_KEYS` — Schema keys (frozen)

**Dependencies:** `_FRAME_CANONICAL`, `__FRAME_INVOKE__`, `FRAME_STORAGE`

#### 5. backup.js

**Purpose:** Identity backup/restore

**Creates:**
- `window.FRAME_BACKUP` (frozen)
  - `exportIdentity(password)` — Export identity bundle
  - `importIdentity(bundle, password)` — Import identity bundle

**Dependencies:** `_FRAME_CANONICAL`, `__FRAME_INVOKE__`, `FRAME_STORAGE`

#### 6. engine.js

**Purpose:** Runtime kernel

**Creates:**
- `window.FRAME` (frozen after invariants.js)
  - `handleIntent(input)` — Main intent handler
  - `handleStructuredIntent(step)` — Structured intent handler
  - `executeCompositeIntent(steps)` — Composite execution
  - `verifyChain(identity)` — Verify receipt chain
  - `computeStateRoot(identity)` — Compute state root
  - `initIntegrityLock()` — Initialize integrity lock
  - Protocol constants: `PROTOCOL_VERSION`, `RECEIPT_VERSION`, `CAPABILITY_VERSION`, `CAPABILITY_SCHEMA`
- `window._FRAME_REGISTER(type, module)` — Temporary registration hook
- `window.FRAME._engine` — Internal engine API (frozen)

**Dependencies:** `_FRAME_CANONICAL`, `FRAME_STORAGE`, `FRAME_IDENTITY`, `FRAME_STATE_ROOT`, `__FRAME_INVOKE__`

#### 7. router.js

**Purpose:** Intent resolution and dApp execution

**Registers:**
- `window._FRAME_REGISTER('router', { resolve, execute, queryPreviews })`

**Dependencies:** `FRAME`, `FRAME_STORAGE`, `FRAME_CAPABILITY_REGISTRY` (if available)

#### 8. replay.js

**Purpose:** Deterministic replay and verification

**Exposes on FRAME:**
- `replayExecutionLog(identity, receipts)` — Replay receipts
- `reconstructStateFromReceipts(identity)` — Reconstruct state
- `replayDeterminismCheck(identity)` — Check determinism
- `reconstructAndVerify(identity)` — Reconstruct and verify
- `exportAttestation(identity)` — Export attestation
- `verifyAttestation(attestation, receiptLog)` — Verify attestation
- `exportGenesis()` — Export genesis attestation
- `exportStateAttestation()` — Export state attestation
- `verifyStateAttestation(attestation)` — Verify state attestation
- `compareStateRoots(localAttestation, remoteAttestation)` — Compare state roots

**Dependencies:** `FRAME._engine`, `FRAME_STORAGE`, `FRAME_STATE_ROOT`

#### 9. federation.js

**Purpose:** Peer synchronization

**Exposes on FRAME:**
- `exportReceiptChain()` — Export receipt chain
- `importReceiptChain(chain, identity)` — Import receipt chain
- `findDivergenceIndex(localChain, remoteChain)` — Find divergence
- `mergeReceiptChains(localChain, remoteChain, identity)` — Merge chains

**Dependencies:** `FRAME._engine`, `FRAME_STORAGE`, `FRAME_STATE_ROOT`

#### 10. contracts.js

**Purpose:** Contract layer

**Exposes on FRAME:**
- `createContract(participants, state)` — Create contract
- `proposeContract(contractId, proposal)` — Propose contract
- `counterContract(contractId, counter)` — Counter contract
- `signContract(contractId)` — Sign contract
- `activateContract(contractId)` — Activate contract
- `updateContractState(contractId, updates)` — Update contract state
- `finalizeContract(contractId)` — Finalize contract
- `cancelContract(contractId)` — Cancel contract
- `updateDriverLocation(contractId, location)` — Update driver location
- `updateRiderLocation(contractId, location)` — Update rider location
- `transitionRideStatus(contractId, newStatus)` — Transition ride status

**Dependencies:** `FRAME._engine`, `FRAME_STORAGE`

#### 11. snapshot.js

**Purpose:** Snapshot export/import

**Creates:**
- `window.FRAME_SNAPSHOT` (frozen)
  - `exportSnapshot()` — Export snapshot
  - `importSnapshot(snapshot)` — Import snapshot

**Dependencies:** `FRAME_STORAGE`, `FRAME_STATE_ROOT`, `FRAME._replay`

#### 12. invariants.js

**Purpose:** Boot assertions and cleanup

**Actions:**
1. **Run all boot assertions:**
   - `assertCapabilityIsolation()` — Capability isolation checks
   - `assertDeterminism()` — Determinism checks
   - `assertIntegrityLock()` — Integrity lock checks
   - `assertFrozenSurfaces()` — Frozen surface checks
   - `assertNoPrivilegedDApps()` — No privileged dApps
   - `assertKernelBootsWithoutDApps()` — Kernel primitives available
   - `assertCapabilitySchemaIntegrity()` — Capability schema checks
   - `assertReceiptDeterminism()` — Receipt determinism checks
   - `assertDeterministicSandbox()` — Sandbox checks
   - `assertReceiptChainCommitment()` — Chain commitment checks
   - `assertCompositeExecution()` — Composite execution checks
   - `assertAttestationPrimitives()` — Attestation checks
   - `assertCrossInstanceAttestation()` — Cross-instance checks
   - `assertEscrowLayer()` — Escrow checks
   - `assertRideEventLayer()` — Ride event checks
   - `assertProtocolFrozen()` — Protocol version checks
   - `assertReceiptFields()` — Receipt field checks
   - `assertStateRootSchema()` — State root schema checks
   - `assertCoreHash()` — Core hash check

2. **Freeze runtime surfaces:**
   - `Object.freeze(window.FRAME)`
   - `Object.freeze(window.FRAME_STORAGE)`
   - `Object.freeze(window.FRAME_IDENTITY)`
   - `Object.freeze(window.FRAME_BACKUP)`
   - `Object.freeze(window.FRAME_STATE_ROOT)`
   - `Object.freeze(window.FRAME_SNAPSHOT)`

3. **Remove registration hook:**
   - Delete `window._FRAME_REGISTER`
   - Prevents new module registration

4. **Initialize integrity lock:**
   - `FRAME.initIntegrityLock()`
   - Captures boot state root
   - Captures boot code hashes

5. **Run verification checks:**
   - `FRAME_STATE_ROOT.selfTest()` — State root determinism
   - `FRAME.replayDeterminismCheck()` — Replay determinism
   - `FRAME.reconstructAndVerify()` — State reconstruction

**Dependencies:** All runtime modules

### Phase 3: System Modules

**Files load after runtime spine:**

```
system/workflowMemory.js
system/federatedMemory.js
system/capabilityReputation.js
system/intentMarket.js
system/creditLedger.js
system/peerRouter.js
system/capsule.js
system/capsuleStore.js
system/capabilityTable.js
system/capabilityAdvertiser.js
system/capsuleScheduler.js
system/capsuleTemplates.js
system/capsuleExecutor.js
system/capsuleNetwork.js
system/network.js
system/intentCapsule.js
system/lanDiscovery.js
system/peerAuth.js
system/p2pTransport.js
system/peerDiscovery.js
system/distributedStorage.js
system/computeWorker.js
system/aiRuntime.js
system/agentRuntime.js
system/agentReputation.js
system/agentPersistence.js
system/capabilityIndex.js
system/networkViz.js
system/aiBuilder.js
system/agentEvolution.js
system/capabilityRegistry.js
system/capabilityPermissions.js
system/dappInstaller.js
system/bootstrapNodes.js
system/relayConfig.js
system/tokenAdapters/ethereumAdapter.js
system/tokenAdapters/solanaAdapter.js
system/tokenAdapters/bitcoinAdapter.js
system/walletManager.js
system/nodeDeployment.js
system/agentTemplates/index.js
system/aiCommandRouter.js
system/networkAutoConfig.js
system/themeManager.js
system/layoutManager.js
system/layoutEngine.js
system/capabilityNetwork.js
system/contextEngine.js
system/contextMemory.js
system/contextAnalyzer.js
system/contextArchiveClient.js
system/widgetManager.js
system/workspaceManager.js
system/frameAPI.js
system/uiStateController.js
system/uiFeedback.js
system/widgetRegistry.js
system/uiPlanExecutor.js
system/agentPlanner.js
system/devReset.js
system/runtimeSelfCheck.js
system/activityFeed.js
system/conversationEngine.js
system/agentRegistry.js
system/agentEngine.js
system/agents/contextAnalyzerAgent.js
system/agents/capabilityDiscoveryAgent.js
system/agents/systemHealthAgent.js
system/agents/aiAssistantAgent.js
ai/intentRouter.js
ai/orchestrator.js
dev/create-dapp-template.js
dev/runtimeFlowTests.js
```

### Phase 4: Application Entry Point

**File:** `app.js`

**Actions:**
1. **Initialize UI:**
   - Set up command bar
   - Set up activity feed
   - Set up widget containers
   - Set up permission modals

2. **Set up event handlers:**
   - Command input handling
   - Preview debouncing
   - Permission dialogs
   - Widget rendering

3. **Ready for user input**

## Boot Verification Steps

### Step 1: Protocol Version Checks

**In invariants.js:**

```javascript
assertProtocolFrozen()
  ├─→ PROTOCOL_VERSION === "2.2.0"
  ├─→ RECEIPT_VERSION === 2
  ├─→ CAPABILITY_VERSION === 2
  └─→ STATE_ROOT_VERSION === 3
```

### Step 2: Receipt Field Checks

**In invariants.js:**

```javascript
assertReceiptFields()
  ├─→ RECEIPT_FIELDS has 16 fields
  ├─→ Fields in canonical order
  └─→ RECEIPT_FIELDS frozen
```

### Step 3: State Root Schema Checks

**In invariants.js:**

```javascript
assertStateRootSchema()
  ├─→ STATE_ROOT_KEYS has 6 keys
  ├─→ Keys in canonical order
  ├─→ Includes receiptChainCommitment
  └─→ STATE_ROOT_KEYS frozen
```

### Step 4: Capability Schema Checks

**In invariants.js:**

```javascript
assertCapabilitySchemaIntegrity()
  ├─→ CAPABILITY_SCHEMA frozen
  ├─→ All capabilities have descriptions
  └─→ validateCapabilities() works correctly
```

### Step 5: Runtime Surface Checks

**In invariants.js:**

```javascript
assertFrozenSurfaces()
  ├─→ FRAME frozen
  ├─→ FRAME_STORAGE frozen
  ├─→ FRAME_IDENTITY frozen
  ├─→ FRAME_BACKUP frozen
  ├─→ FRAME_STATE_ROOT frozen
  └─→ FRAME_SNAPSHOT frozen
```

### Step 6: Integrity Lock Initialization

**In invariants.js:**

```javascript
FRAME.initIntegrityLock()
  ├─→ Compute boot state root
  ├─→ Store as _bootStateRoot
  ├─→ Compute boot code hashes
  ├─→ Store as _bootCodeHashes
  └─→ Set _integrityLockReady = true
```

### Step 7: Replay Determinism Check

**In invariants.js:**

```javascript
FRAME.replayDeterminismCheck(identity)
  ├─→ Load receipt chain
  ├─→ If chain empty: skip
  ├─→ Replay last receipt
  ├─→ Verify state root matches
  └─→ Detect nondeterminism
```

### Step 8: State Reconstruction

**In invariants.js:**

```javascript
FRAME.reconstructAndVerify(identity)
  ├─→ Load receipt chain
  ├─→ Replay all receipts
  ├─→ Verify state root matches
  └─→ Detect corruption
```

## Boot Failure Handling

### Invariant Violation

**If any assertion fails:**

- Runtime initialization aborted
- Error thrown
- Boot fails
- User sees error message

### Replay Determinism Failure

**If replay produces different state root:**

- Boot fails
- Safe mode entered
- Error logged
- User must fix nondeterminism

### State Reconstruction Failure

**If state reconstruction fails:**

- Boot fails
- Safe mode entered
- Error logged
- User must restore from snapshot

## Boot Success Criteria

**Boot is successful if:**

1. ✅ All invariants pass
2. ✅ Runtime surfaces frozen
3. ✅ Registration hook removed
4. ✅ Integrity lock initialized
5. ✅ Replay determinism check passes
6. ✅ State reconstruction passes
7. ✅ `app.js` loads
8. ✅ UI ready for input

## Boot Timing

**Typical boot sequence:**

- HTML load: ~10ms
- Runtime spine: ~50-100ms
- System modules: ~200-500ms
- Invariants: ~50-100ms
- Verification: ~100-500ms (depends on chain length)
- Application: ~50ms

**Total:** ~500ms - 1.5s (depending on chain length)

## Related Documentation

- [Runtime Invariants](runtime_invariants.md) - Boot assertions
- [Kernel Runtime](kernel_runtime.md) - Engine initialization
- [Integration Flows](integration_flows.md) - Module interactions
