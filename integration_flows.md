# FRAME Integration Flows

This document shows how modules interact in FRAME. Integration flows expose broken connections between modules and help identify where failures occur.

## Intent Execution Flow

**Complete flow from user input to receipt:**

```
User Input (text or structured)
  ↓
app.js (handleCommand)
  ↓
conversationEngine.js (processUserMessage)
  ↓
conversationEngine.js (resolveIntent)
  ├─→ kernelNormalize (system intents)
  ├─→ normalizeViaAiDApp (AI parsing)
  └─→ ai.chat (fallback)
  ↓
Intent Resolved: { action, payload, raw, timestamp }
  ↓
agentPlanner.js (planExecution)
  ├─→ Maps action to canonical capability
  ├─→ Builds execution graph
  └─→ Returns: { executionGraph: [{ id, canonical, dependsOn, args }] }
  ↓
agentPlanner.js (executeGraph)
  ├─→ Topological sort (dependency order)
  ├─→ For each step:
  │     ├─→ capabilityRegistry.getBestCapability (provider selection)
  │     ├─→ engine.handleIntent (step execution)
  │     └─→ Dependency injection ($deps, $prev)
  └─→ Returns: last step result or error
  ↓
engine.js (handleIntent)
  ├─→ kernelNormalize (normalize input)
  ├─→ validateIntent (validate structure)
  ├─→ handleBuiltIn (system intents)
  ├─→ router.resolve (find dApp)
  ├─→ Permission checks (hasPermission)
  ├─→ checkIntegrityLock (state root check)
  ├─→ verifyDAppCodeHash (code hash check)
  ├─→ computeStateRoot (previous state root)
  ├─→ installDeterministicSandbox (freeze time, seed random)
  ├─→ installAsyncBlockers (block nondeterministic APIs)
  ├─→ router.execute (run dApp)
  │     ├─→ resolveManifestCapabilities (verify capabilities)
  │     ├─→ import(modulePath) (load dApp)
  │     ├─→ buildScopedApi (create scoped API)
  │     ├─→ dApp.run(intent, api) (execute dApp)
  │     └─→ Restore async blockers
  ├─→ getExecutionCapLog (capabilities used)
  ├─→ computeStateRoot (next state root)
  ├─→ buildReceipt (create receipt)
  ├─→ appendReceipt (append to chain)
  └─→ Return: { type: 'dapp', dappId, receipt, ...result }
  ↓
Receipt Generated: { version, timestamp, identity, dappId, intent, inputHash, inputPayload, capabilitiesDeclared, capabilitiesUsed, previousStateRoot, nextStateRoot, resultHash, previousReceiptHash, receiptHash, publicKey, signature }
  ↓
UI Planning (optional)
  ├─→ uiPlanner.planUI (AI-driven planning)
  ├─→ uiPlanExecutor.executeUIPlan (apply layout, widgets)
  └─→ widgetManager.renderAll (render widgets)
  ↓
Display Response to User
```

## Replay Verification Flow

**Complete flow for replaying receipt chain:**

```
Load Receipt Chain
  ↓
FRAME_STORAGE.read('chain:' + identity)
  ↓
Enter Reconstruction Mode
  ↓
engine.enterReconstructionMode()
  ├─→ _reconstructionMode = true
  └─→ No receipts appended, no state mutations
  ↓
For Each Receipt (in order):
  ├─→ Verify Chain Link
  │     ├─→ receipt[n].previousReceiptHash == receipt[n-1].receiptHash
  │     └─→ First receipt: previousReceiptHash == null
  ├─→ Verify Receipt Hash
  │     ├─→ signable = receiptSignablePayload(receipt)
  │     ├─→ recomputedHash = sha256(JSON.stringify(signable))
  │     └─→ recomputedHash == receipt.receiptHash
  ├─→ Verify Signature
  │     ├─→ key = importKey(receipt.publicKey)
  │     ├─→ verify(signature, receiptHash, key)
  │     └─→ Signature valid
  ├─→ Verify Input Hash
  │     ├─→ recomputedInputHash = canonicalHash(receipt.inputPayload)
  │     └─→ recomputedInputHash == receipt.inputHash
  ├─→ Install Deterministic Sandbox
  │     ├─→ installDeterministicSandbox(receipt.timestamp, receipt.previousReceiptHash)
  │     ├─→ Freeze Date.now() to receipt.timestamp
  │     └─→ Seed Math.random() from receipt hash
  ├─→ Inject Network Responses
  │     ├─→ If receipt.inputPayload.networkResponses exists
  │     └─→ Inject sealed responses (no live network)
  ├─→ Re-execute Intent
  │     ├─→ handleIntent(receipt.inputPayload)
  │     └─→ Result computed
  ├─→ Verify Result Hash
  │     ├─→ recomputedResultHash = canonicalHash(result)
  │     └─→ recomputedResultHash == receipt.resultHash
  ├─→ Compute State Root
  │     ├─→ computedRoot = computeStateRoot(identity)
  │     └─→ computedRoot == receipt.nextStateRoot
  └─→ Continue to next receipt
  ↓
Exit Reconstruction Mode
  ↓
engine.exitReconstructionMode()
  ├─→ _reconstructionMode = false
  └─→ Normal execution restored
  ↓
Replay Complete
```

## Snapshot Restore Flow

**Complete flow for restoring from snapshot:**

```
Load Snapshot
  ↓
snapshot.js (importSnapshot)
  ├─→ Validate snapshot structure
  ├─→ Check protocol versions match
  ├─→ Check receipt version match
  ├─→ Check capability version match
  └─→ Check state root version match
  ↓
Save Current State
  ↓
gatherStorageSnapshot()
  ├─→ Read all storage keys
  └─→ Deep clone storage state
  ↓
Clear Storage
  ↓
FRAME_STORAGE.clear()
  ↓
Restore Storage
  ↓
For each key in snapshot.storage:
  ├─→ FRAME_STORAGE.write(key, value)
  └─→ Validate canonical JSON
  ↓
Restore Receipt Chain
  ↓
FRAME_STORAGE.write('chain:' + identity, snapshot.receiptChain)
  ↓
Verify State Root
  ↓
computedRoot = computeStateRoot(identity)
  ├─→ If computedRoot !== snapshot.stateRoot:
  │     ├─→ Rollback storage
  │     ├─→ restoreStorageSnapshot(preImportSnapshot)
  │     └─→ Throw error
  └─→ If match: Continue
  ↓
Restore Complete
```

## Federation Sync Flow

**Complete flow for syncing with peer:**

```
Export Attestation
  ↓
replay.js (exportAttestation)
  ├─→ Get current identity
  ├─→ Get public key
  ├─→ Compute state root
  ├─→ Get receipt chain commitment
  ├─→ Build attestation payload
  ├─→ Sign payload
  └─→ Return attestation
  ↓
Send Attestation to Peer
  ↓
Receive Peer Attestation
  ↓
Compare State Roots
  ↓
replay.js (compareStateRoots)
  ├─→ Verify both attestations (signatures)
  ├─→ Compare state roots
  │     ├─→ If match: Synchronized
  │     └─→ If differ: Divergence detected
  └─→ Return: { synchronized, localAhead, remoteAhead, divergence }
  ↓
If Divergence Detected:
  ├─→ Request Receipt Chain from Peer
  ├─→ Receive Receipt Chain
  ├─→ Verify Receipt Chain
  │     ├─→ Structural verification
  │     ├─→ Signature verification
  │     └─→ Chain link verification
  ├─→ Find Divergence Index
  │     ├─→ federation.js (findDivergenceIndex)
  │     ├─→ Find common ancestor
  │     └─→ Extract divergent receipts
  ├─→ Replay Remote Receipts
  │     ├─→ Enter reconstruction mode
  │     ├─→ Replay divergent receipts
  │     └─→ Verify state root matches peer's
  ├─→ Merge Chains
  │     ├─→ federation.js (mergeReceiptChains)
  │     ├─→ Append remote receipts to local chain
  │     └─→ Write merged chain
  └─→ Sync Complete
```

## Composite Execution Flow

**Complete flow for multi-step atomic execution:**

```
Execute Composite Intent
  ↓
engine.js (executeCompositeIntent)
  ├─→ Validate steps (at least 2)
  ├─→ Check mode (not safe, not replay)
  └─→ Snapshot State
  ↓
gatherStorageSnapshot()
  ├─→ Deep clone storage
  ├─→ Capture state root
  └─→ Capture receipt chain
  ↓
For Each Step (in dependency order):
  ├─→ Build Step Intent
  │     ├─→ action = step.canonical
  │     ├─→ payload = step.args
  │     └─→ Inject dependencies ($deps, $prev)
  ├─→ Execute Step
  │     ├─→ handleStructuredIntent(stepIntent)
  │     └─→ Result returned
  ├─→ Check Result
  │     ├─→ If error: Stop execution
  │     └─→ If success: Store result
  └─→ Continue to next step
  ↓
If All Steps Succeed:
  ├─→ Generate Receipt
  │     ├─→ buildReceipt({ intent: 'composite', ... })
  │     └─→ Receipt includes all step results
  ├─→ Append Receipt
  └─→ Return: { success: true, results: [...] }
  ↓
If Any Step Fails:
  ├─→ Rollback Storage
  │     ├─→ restoreStorageSnapshot(snapshotStorage)
  │     └─→ Restore receipt chain
  ├─→ Return Error
  └─→ No receipt generated
```

## Permission Grant Flow

**Complete flow for granting capabilities:**

```
Intent Execution Requested
  ↓
engine.js (handleIntent)
  ├─→ router.resolve (find dApp)
  └─→ Check Permissions
  ↓
For Each Required Capability:
  ├─→ hasPermission(identity, dappId, capability)
  │     ├─→ Read permissions:<identity>
  │     ├─→ Check dApp permissions
  │     └─→ Return: true/false
  └─→ If Missing Permission:
        ├─→ Return: { type: 'permission_request', dappId, capability }
        └─→ Stop execution
  ↓
UI Shows Permission Modal
  ↓
User Grants Permission
  ↓
engine.js (grantPermission)
  ├─→ Read permissions:<identity>
  ├─→ Add capability to dApp permissions
  └─→ Write permissions:<identity>
  ↓
Retry Intent Execution
  ↓
Permission Check Passes
  ↓
Execution Continues
```

## Capability Resolution Flow

**Complete flow for resolving capabilities to dApps:**

```
Intent Action Received
  ↓
router.js (resolve)
  ├─→ Load Manifests
  │     ├─→ list_dapps() (get dApp IDs)
  │     ├─→ Fetch manifest.json for each
  │     └─→ Validate manifests
  └─→ Resolution Strategies (in order):
        ├─→ Strategy 1: Canonical Capability Lookup
        │     ├─→ capabilityRegistry.getBestCapabilityByCanonical(action)
        │     └─→ If found: Return provider
        ├─→ Strategy 2: AI Intent Routing (if ai.*)
        │     ├─→ capabilityRegistry.getBestCapability(action, context)
        │     └─→ If found: Return AI provider
        └─→ Strategy 3: Manifest Intent Matching
              ├─→ Exact match: action === declaredIntent
              ├─→ Prefix match: action.startsWith(declaredIntent + '.')
              └─→ If found: Return dApp match
  ↓
Match Found: { id, modulePath, capabilities }
  ↓
router.js (execute)
  ├─→ Validate Module Path
  │     ├─→ Must be under dapps/
  │     └─→ No path traversal (..)
  ├─→ Resolve Manifest Capabilities
  │     ├─→ Fetch manifest
  │     ├─→ Validate capabilities
  │     └─→ Return verified capabilities
  ├─→ Dynamic Import
  │     ├─→ import(modulePath)
  │     └─→ Get run function
  ├─→ Build Scoped API
  │     ├─→ buildScopedApi(dappId, capabilities)
  │     └─→ Only declared capabilities included
  ├─→ Freeze Intent
  ├─→ Install Async Blockers
  ├─→ Execute dApp
  │     ├─→ run(intent, scopedApi)
  │     └─→ Result returned
  └─→ Restore Async Blockers
  ↓
Result Returned to Engine
```

## State Root Computation Flow

**Complete flow for computing deterministic state root:**

```
Compute State Root Requested
  ↓
stateRoot.js (computeStateRoot)
  ├─→ Get Identity
  │     ├─→ get_current_identity()
  │     └─→ Get public key
  ├─→ Load Installed dApps
  │     ├─→ list_dapps()
  │     ├─→ For each dApp:
  │     │     ├─→ Fetch manifest.json
  │     │     ├─→ Compute codeHash (sha256 of index.js)
  │     │     └─→ Canonicalize manifest
  │     └─→ Sort by ID
  ├─→ Gather Storage Data
  │     ├─→ storage_list_keys()
  │     ├─→ Filter excluded prefixes
  │     ├─→ Read each key
  │     └─→ Deep sort for determinism
  ├─→ Compute Receipt Chain Commitment
  │     ├─→ Read receipt chain
  │     ├─→ If empty: commitment = null
  │     └─→ If not empty: commitment = lastReceipt.receiptHash
  └─→ Build Payload
        ├─→ version: STATE_ROOT_VERSION
        ├─→ capabilityVersion: CAPABILITY_VERSION
        ├─→ identityPublicKey: publicKey
        ├─→ installedDApps: [{ id, manifest, codeHash }]
        ├─→ storage: { key: value, ... }
        └─→ receiptChainCommitment: commitment
  ↓
Canonicalize Payload
  ↓
canonicalize(payload)
  ├─→ Sort object keys
  ├─→ Deep sort nested structures
  └─→ Canonical JSON
  ↓
Hash Payload
  ↓
sha256(JSON.stringify(canonicalizedPayload))
  ↓
State Root: hex SHA-256 hash
```

## Boot Sequence Flow

**Complete flow from index.html to runtime ready:**

```
index.html Loads
  ↓
Script Loading (in order):
  ├─→ runtime/canonical.js
  │     └─→ Creates window._FRAME_CANONICAL
  ├─→ runtime/storage.js
  │     └─→ Creates window.FRAME_STORAGE
  ├─→ runtime/identity.js
  │     └─→ Creates window.FRAME_IDENTITY
  ├─→ runtime/stateRoot.js
  │     └─→ Creates window.FRAME_STATE_ROOT
  ├─→ runtime/backup.js
  │     └─→ Creates window.FRAME_BACKUP
  ├─→ runtime/engine.js
  │     ├─→ Creates window.FRAME
  │     ├─→ Creates window._FRAME_REGISTER hook
  │     └─→ Exposes handleIntent
  ├─→ runtime/router.js
  │     ├─→ Registers with engine via _FRAME_REGISTER
  │     └─→ Exposes resolve, execute, queryPreviews
  ├─→ runtime/replay.js
  │     └─→ Exposes replay functions on FRAME
  ├─→ runtime/federation.js
  │     └─→ Exposes federation functions on FRAME
  ├─→ runtime/contracts.js
  │     └─→ Exposes contract functions on FRAME
  ├─→ runtime/snapshot.js
  │     └─→ Creates window.FRAME_SNAPSHOT
  └─→ runtime/invariants.js
        ├─→ Runs all boot assertions
        ├─→ Freezes runtime surfaces
        ├─→ Removes _FRAME_REGISTER hook
        └─→ Initializes integrity lock
  ↓
System Modules Load (in order)
  ├─→ workflowMemory.js
  ├─→ federatedMemory.js
  ├─→ capabilityReputation.js
  ├─→ ... (all system modules)
  └─→ conversationEngine.js
  ↓
app.js Loads
  ├─→ Sets up UI event handlers
  ├─→ Initializes command bar
  └─→ Ready for user input
  ↓
Boot Complete
  ├─→ Runtime ready
  ├─→ Integrity lock initialized
  ├─→ All invariants verified
  └─→ Runtime surfaces frozen
```

## Integration Points

### Critical Integration Points

**Engine ↔ Router:**
- Engine calls `router.resolve()` for intent resolution
- Engine calls `router.execute()` for dApp execution
- Router registers via `_FRAME_REGISTER('router', module)`

**Engine ↔ Storage:**
- Engine reads/writes receipt chain via `FRAME_STORAGE`
- Engine reads/writes permissions via `FRAME_STORAGE`
- Storage validates canonical JSON

**Engine ↔ State Root:**
- Engine calls `computeStateRoot()` before/after execution
- State root includes receipt chain commitment
- State root computed deterministically

**Engine ↔ Identity:**
- Engine gets current identity via `__FRAME_INVOKE__('get_current_identity')`
- Engine signs receipts via `__FRAME_INVOKE__('sign_data')`
- Identity isolated per vault

**Router ↔ Capability Registry:**
- Router uses registry for provider selection
- Registry scans manifests and builds indexes
- Registry scores providers

**Agent Planner ↔ Engine:**
- Planner calls `engine.handleIntent()` for each step
- Planner builds execution graphs
- Planner handles dependency injection

## Failure Points

**Common failure points in integration:**

1. **Router resolution fails** → Intent unknown, execution blocked
2. **Permission check fails** → Permission request returned
3. **Integrity lock fails** → Safe mode entered
4. **Code hash mismatch** → Safe mode entered
5. **State root mismatch** → Safe mode entered
6. **Receipt chain corruption** → Replay fails
7. **dApp module load fails** → Execution fails
8. **Manifest validation fails** → dApp not loaded

## Related Documentation

- [Runtime Pipeline](runtime_pipeline.md) - Detailed execution flow
- [Kernel Runtime](kernel_runtime.md) - Engine implementation
- [Router and Intents](router_and_intents.md) - Router implementation
- [Failure Modes](failure_modes.md) - Failure scenarios
