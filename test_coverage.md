# FRAME Test Coverage

This document maps every system component to tests and identifies missing test coverage.

## Current Test Status

### ✅ Existing Tests

**File:** `ui/dev/runtimeFlowTests.js`

**Entry:** Run via command bar: `frame test` or `frame.test`

**Function:** `runFrameFlowTests()`

**Tests:**

1. **show wallet** — `wallet.balance` intent
   - Intent resolution ✓
   - Execution graph ✓
   - Execution ✓
   - UI plan ✓
   - Receipt generation ✓

2. **send 5 frame to Alice** — `wallet.send` intent
   - Intent resolution ✓
   - Execution graph ✓
   - Execution ✓
   - Receipt optional (may need vault) ✓

3. **create note "hello world"** — `notes.create` intent
   - Intent resolution ✓
   - Execution ✓
   - Receipt generation ✓

4. **list contacts** — `contacts.search` intent
   - Intent resolution ✓
   - Execution ✓

5. **system overview** — `system.overview` intent
   - Intent resolution ✓
   - Empty graph allowed ✓
   - UI plan ✓
   - Receipt optional ✓

**Per-test checks:**
- Intent resolved (action set) ✓
- Execution graph built ✓
- Execution completed ✓
- UI plan generated ✓
- Receipt produced (when applicable) ✓

## Test Coverage by Component

### Runtime Modules

#### canonical.js
- [ ] `canonicalize()` — Array canonicalization
- [ ] `canonicalize()` — Object key sorting
- [ ] `canonicalize()` — Nested structures
- [ ] `sha256()` — Hash computation
- [ ] `canonicalHash()` — Canonical hash computation

#### storage.js
- [ ] `validateCanonicalJson()` — Safe integer validation
- [ ] `validateCanonicalJson()` — Float rejection
- [ ] `validateCanonicalJson()` — NaN/Infinity rejection
- [ ] `validateCanonicalJson()` — Function rejection
- [ ] `validateCanonicalJson()` — Date rejection
- [ ] `validateCanonicalJson()` — Circular reference detection
- [ ] `deepCloneAndFreeze()` — Deep freeze behavior
- [ ] `read()` — Storage read
- [ ] `write()` — Storage write with validation

#### identity.js
- [ ] `list()` — List identities
- [ ] `getCurrent()` — Get current identity
- [ ] `switch()` — Switch identity
- [ ] `create()` — Create identity
- [ ] `getCapabilities()` — Get identity capabilities

#### stateRoot.js
- [ ] `computeStateRoot()` — State root computation
- [ ] `computeCodeHash()` — Code hash computation
- [ ] `loadInstalledDApps()` — dApp loading
- [ ] `gatherStorageData()` — Storage enumeration
- [ ] `computeReceiptChainCommitment()` — Chain commitment
- [ ] `selfTest()` — Determinism self-test

#### backup.js
- [ ] `exportIdentity()` — Identity export
- [ ] `importIdentity()` — Identity import
- [ ] AES-256-GCM encryption
- [ ] PBKDF2 key derivation
- [ ] Password validation

#### engine.js
- [ ] `handleIntent()` — Intent execution
- [ ] `kernelNormalize()` — Kernel normalization
- [ ] `normalizeViaAiDApp()` — AI normalization
- [ ] `validateIntent()` — Intent validation
- [ ] `handleBuiltIn()` — System intents
- [ ] `buildScopedApi()` — Scoped API construction
- [ ] `capabilityGuard()` — Capability enforcement
- [ ] `buildReceipt()` — Receipt creation
- [ ] `appendReceipt()` — Receipt appending
- [ ] `executeCompositeIntent()` — Composite execution
- [ ] `installDeterministicSandbox()` — Sandbox installation
- [ ] `installAsyncBlockers()` — Async blocker installation
- [ ] `checkIntegrityLock()` — Integrity lock check
- [ ] `verifyDAppCodeHash()` — Code hash verification

#### router.js
- [ ] `loadManifests()` — Manifest loading
- [ ] `validateManifest()` — Manifest validation
- [ ] `resolve()` — Intent resolution (exact match)
- [ ] `resolve()` — Intent resolution (prefix match)
- [ ] `resolve()` — Intent resolution (AI routing)
- [ ] `execute()` — dApp execution
- [ ] `queryPreviews()` — Preview queries
- [ ] Path security (no `..` traversal)
- [ ] Module path validation

#### replay.js
- [ ] `replayExecutionLog()` — Full replay
- [ ] `reconstructStateFromReceipts()` — State reconstruction
- [ ] `verifyReceiptChain()` — Chain verification
- [ ] `verifyReceiptSignatureWithKey()` — Signature verification
- [ ] `exportAttestation()` — Attestation export
- [ ] `verifyAttestation()` — Attestation verification
- [ ] `exportStateAttestation()` — State attestation export
- [ ] `verifyStateAttestation()` — State attestation verification
- [ ] `exportGenesis()` — Genesis export
- [ ] Network response injection during replay

#### federation.js
- [ ] `exportReceiptChain()` — Chain export
- [ ] `findDivergenceIndex()` — Divergence detection
- [ ] `mergeReceiptChains()` — Chain merge
- [ ] `importReceiptChain()` — Chain import
- [ ] Replay-before-merge verification

#### contracts.js
- [ ] `createContract()` — Contract creation
- [ ] `proposeContract()` — Contract proposal
- [ ] `counterContract()` — Contract counter
- [ ] `signContract()` — Contract signing
- [ ] `activateContract()` — Contract activation
- [ ] `updateDriverLocation()` — Driver location update
- [ ] `updateRiderLocation()` — Rider location update
- [ ] `transitionRideStatus()` — Ride status transition
- [ ] Contract revision increments
- [ ] Ride state machine transitions

#### snapshot.js
- [ ] `exportSnapshot()` — Snapshot export
- [ ] `importSnapshot()` — Snapshot import
- [ ] Version compatibility checks
- [ ] Rollback on failure
- [ ] State root verification

#### invariants.js
- [ ] Protocol version checks
- [ ] Receipt field checks
- [ ] State root schema checks
- [ ] Capability schema checks
- [ ] Runtime surface checks
- [ ] Module registration removal
- [ ] All boot assertions

### dApps

#### ai
- [ ] `run()` — Intent parsing
- [ ] `run()` — Chat responses
- [ ] `run()` — Text summarization
- [ ] `preview()` — Preview responses
- [ ] `classify()` — Intent classification
- [ ] Request queue handling
- [ ] Model loading (if implemented)

#### wallet
- [ ] `run()` — Balance check
- [ ] `run()` — Send payment
- [ ] `run()` — Mint frame
- [ ] `run()` — Burn frame
- [ ] `run()` — Escrow lock
- [ ] `run()` — Escrow release
- [ ] `run()` — Escrow refund
- [ ] `preview()` — Payment preview
- [ ] Total supply tracking
- [ ] Escrow balance calculation

#### bridge
- [ ] `run()` — Deposit
- [ ] `run()` — Withdraw
- [ ] Proof canonicalization
- [ ] Duplicate prevention
- [ ] Supply updates
- [ ] Deterministic timestamping

#### contracts
- [ ] Contract lifecycle
- [ ] Ride state machine
- [ ] Location updates
- [ ] Status transitions

### System Modules

#### conversationEngine.js
- [ ] `resolveIntent()` — Intent resolution
- [ ] `processUserMessage()` — Message processing
- [ ] AI fallback behavior

#### agentPlanner.js
- [ ] `planExecution()` — Execution graph planning
- [ ] `executeGraph()` — Graph execution
- [ ] Topological sort
- [ ] Dependency injection
- [ ] Parallel execution
- [ ] Retry logic
- [ ] Provider fallback

#### capabilityRegistry.js
- [ ] Manifest scanning
- [ ] Capability indexing
- [ ] Provider scoring
- [ ] Best provider selection

## Missing Test Coverage

### Critical Missing Tests

#### Determinism Tests
- [ ] Deterministic replay with frozen time
- [ ] Deterministic replay with seeded randomness
- [ ] Network response sealing and replay
- [ ] State root determinism (same state → same root)

#### Security Tests
- [ ] Capability escalation attempts
- [ ] Undeclared capability access
- [ ] Permission bypass attempts
- [ ] Path traversal attempts
- [ ] Receipt tampering detection
- [ ] Signature forgery attempts

#### Protocol Tests
- [ ] Receipt version mismatch handling
- [ ] State root version mismatch handling
- [ ] Capability version mismatch handling
- [ ] Protocol version mismatch handling

#### Error Handling Tests
- [ ] dApp module load failure
- [ ] Manifest validation failure
- [ ] Receipt chain corruption
- [ ] State root mismatch detection
- [ ] Code hash mismatch detection

#### Integration Tests
- [ ] Full intent execution flow
- [ ] Composite execution with rollback
- [ ] Snapshot export/import
- [ ] Federation sync
- [ ] Replay verification

#### Edge Cases
- [ ] Empty receipt chain
- [ ] Single receipt chain
- [ ] Large receipt chains
- [ ] Concurrent execution (if supported)
- [ ] Identity switching during execution

## Test Infrastructure Needs

### Missing Infrastructure

- [ ] Unit test framework (Jest, Mocha, etc.)
- [ ] Integration test framework
- [ ] Mock/stub infrastructure for Tauri backend
- [ ] Test data fixtures
- [ ] Test utilities (canonical helpers, hash helpers)
- [ ] CI/CD test execution

### Test Organization

**Recommended structure:**

```
tests/
├── unit/
│   ├── runtime/
│   ├── dapps/
│   └── system/
├── integration/
│   ├── flows/
│   └── federation/
└── e2e/
    └── scenarios/
```

## Test Priorities

### Priority 1: Protocol Correctness
- Receipt creation and verification
- State root computation
- Deterministic replay
- Capability enforcement

### Priority 2: Security
- Capability isolation
- Permission enforcement
- Path security
- Signature verification

### Priority 3: Integration
- Intent execution flow
- Composite execution
- Snapshot operations
- Federation sync

### Priority 4: Edge Cases
- Error handling
- Failure recovery
- Edge case scenarios

## Related Documentation

- [Runtime Invariants](runtime_invariants.md) - What must be true
- [Failure Modes](failure_modes.md) - What can break
- [Integration Flows](integration_flows.md) - How modules connect
