# FRAME Protocol Compliance Checklist

This is a developer audit checklist for verifying FRAME protocol compliance. Use this to audit code changes and ensure protocol correctness.

## Receipt Compliance

### Receipt Structure
- [ ] Receipt has exactly 16 fields
- [ ] Fields are in canonical order: `capabilitiesDeclared, capabilitiesUsed, dappId, identity, inputHash, inputPayload, intent, nextStateRoot, previousReceiptHash, previousStateRoot, publicKey, receiptHash, resultHash, signature, timestamp, version`
- [ ] `version` equals `RECEIPT_VERSION` (2)
- [ ] `timestamp` is normalized (seconds, not milliseconds)
- [ ] `inputHash` matches canonicalized `inputPayload`
- [ ] `resultHash` matches canonicalized result
- [ ] `previousReceiptHash` links to previous receipt (null for first)
- [ ] `receiptHash` is SHA-256 of signable payload
- [ ] `signature` is Ed25519 signature of `receiptHash`
- [ ] `publicKey` is Ed25519 public key (hex)

### Receipt Generation
- [ ] Receipt built via `buildReceipt()` only
- [ ] Signable payload excludes `receiptHash` and `signature`
- [ ] Receipt canonicalized before hashing
- [ ] Signature computed after hash
- [ ] Receipt appended via `appendReceipt()` only

## State Root Compliance

### State Root Structure
- [ ] State root has exactly 6 keys
- [ ] Keys in canonical order: `capabilityVersion, identityPublicKey, installedDApps, receiptChainCommitment, storage, version`
- [ ] `version` equals `STATE_ROOT_VERSION` (3)
- [ ] `capabilityVersion` equals `CAPABILITY_VERSION` (2)
- [ ] `identityPublicKey` is Ed25519 public key (hex)
- [ ] `installedDApps` includes `codeHash` for each dApp
- [ ] `receiptChainCommitment` is hash of latest receipt (not full chain)
- [ ] `storage` excludes ephemeral keys
- [ ] All keys deep-sorted for determinism

### State Root Computation
- [ ] State root computed via `computeStateRoot()` only
- [ ] Computed before and after each execution
- [ ] Payload canonicalized before hashing
- [ ] Deterministic (same state → same root)
- [ ] Self-test passes at boot

## Capability Compliance

### Capability Schema
- [ ] `CAPABILITY_SCHEMA` is frozen (`Object.freeze()`)
- [ ] Schema cannot be modified after boot
- [ ] All capabilities have string descriptions
- [ ] Unknown capabilities rejected during manifest validation
- [ ] `CAPABILITY_VERSION` equals 2

### Capability Declaration
- [ ] dApps declare capabilities in manifest
- [ ] Capabilities validated against schema
- [ ] Invalid capabilities cause manifest rejection
- [ ] Capabilities sorted in receipt

### Capability Permission
- [ ] Permissions stored per identity: `permissions:<identity>`
- [ ] Permission checked before execution
- [ ] Missing permission triggers request (not error)
- [ ] Permission grants persist across restarts

### Capability Usage
- [ ] All privileged operations pass `capabilityGuard()`
- [ ] Capability usage logged in `_executionCapLog`
- [ ] `capabilitiesUsed` included in receipt
- [ ] Only declared capabilities can be used

## Determinism Compliance

### Time Freezing
- [ ] `Date.now()` frozen during execution
- [ ] Frozen to execution-start timestamp
- [ ] Restored after execution
- [ ] Replay uses receipt timestamp

### Randomness Seeding
- [ ] `Math.random()` replaced with seeded PRNG
- [ ] Seeded from receipt hash or previous receipt hash
- [ ] Restored after execution
- [ ] Replay uses same seed

### Async API Blocking
- [ ] `setTimeout` blocked during execution
- [ ] `setInterval` blocked during execution
- [ ] `fetch` blocked during execution
- [ ] `WebSocket` blocked during execution
- [ ] `EventSource` blocked during execution
- [ ] Blockers restored after execution

### Network Request Sealing
- [ ] Network requests use `network.request` capability
- [ ] Responses stored in `inputPayload.networkResponses`
- [ ] Replay uses sealed responses (no live network)
- [ ] Network log reset before each execution

### Canonical Data
- [ ] All storage values are canonical JSON
- [ ] Only safe integers (no floats, NaN, Infinity)
- [ ] No functions, classes, Dates, RegExp
- [ ] No circular references
- [ ] Deep freeze after write

## Receipt Chain Compliance

### Chain Linking
- [ ] Receipts linked via `previousReceiptHash`
- [ ] First receipt has `previousReceiptHash = null`
- [ ] Chain links verified during replay
- [ ] Chain links verified during import

### Chain Storage
- [ ] Chain stored in `chain:<identity>`
- [ ] Chain is array of receipts
- [ ] Chain append-only (no deletions)
- [ ] Chain canonicalized for commitment

### Chain Verification
- [ ] Structural verification (fields, order)
- [ ] Hash verification (receipt hash)
- [ ] Signature verification (Ed25519)
- [ ] Input hash verification
- [ ] Result hash verification (during replay)

## State Root Compliance

### State Root Consistency
- [ ] State root computed before execution
- [ ] State root computed after execution
- [ ] `previousStateRoot` matches computed root
- [ ] `nextStateRoot` matches computed root
- [ ] Integrity lock checks state root

### State Root Determinism
- [ ] Same state always produces same root
- [ ] Deep sort ensures deterministic ordering
- [ ] Canonicalization ensures deterministic JSON
- [ ] Self-test verifies determinism

### Code Hash Binding
- [ ] Code hash computed for each dApp
- [ ] Code hash included in state root
- [ ] Code hash verified at boot
- [ ] Code hash verified before execution

## Runtime Surface Compliance

### Frozen Surfaces
- [ ] `FRAME` frozen after boot
- [ ] `FRAME_STORAGE` frozen after boot
- [ ] `FRAME_IDENTITY` frozen after boot
- [ ] `FRAME_BACKUP` frozen after boot
- [ ] `FRAME_STATE_ROOT` frozen after boot
- [ ] `FRAME_SNAPSHOT` frozen after boot

### Module Registration
- [ ] `_FRAME_REGISTER` hook removed after boot
- [ ] No new modules can register after boot
- [ ] Router registers via hook during boot
- [ ] All modules loaded before invariants.js

### Capability Isolation
- [ ] `buildScopedApi` not exposed on `FRAME`
- [ ] `capabilityGuard` not exposed on `FRAME`
- [ ] No privileged dApp modules on window
- [ ] dApps only access scoped API

## Boot Sequence Compliance

### Script Loading Order
- [ ] canonical.js loads first
- [ ] storage.js loads second
- [ ] identity.js loads third
- [ ] stateRoot.js loads fourth
- [ ] backup.js loads fifth
- [ ] engine.js loads sixth
- [ ] router.js loads seventh
- [ ] replay.js loads eighth
- [ ] federation.js loads ninth
- [ ] contracts.js loads tenth
- [ ] snapshot.js loads eleventh
- [ ] invariants.js loads last

### Boot Assertions
- [ ] Protocol version checks pass
- [ ] Receipt field checks pass
- [ ] State root schema checks pass
- [ ] Capability schema checks pass
- [ ] Runtime surface checks pass
- [ ] Module registration removed
- [ ] Integrity lock initialized

### Boot Verification
- [ ] Replay determinism check runs
- [ ] State reconstruction runs
- [ ] Self-test runs
- [ ] All checks pass before runtime ready

## Composite Execution Compliance

### Atomicity
- [ ] All steps succeed or all fail
- [ ] State snapshot before execution
- [ ] Rollback on any step failure
- [ ] Single receipt for entire composite

### Dependency Ordering
- [ ] Steps execute in dependency order
- [ ] Topological sort resolves order
- [ ] Cycles detected before execution
- [ ] Missing dependencies detected

### Dependency Injection
- [ ] `$deps` map injected into step args
- [ ] `$prev` single dependency injected
- [ ] Results stored by step ID
- [ ] Dependencies available when needed

## Federation Compliance

### Attestation
- [ ] Attestation includes protocol versions
- [ ] Attestation includes state root
- [ ] Attestation includes receipt chain commitment
- [ ] Attestation signed with Ed25519
- [ ] Attestation verified before use

### Chain Import
- [ ] Chain verified before import
- [ ] Structural verification passes
- [ ] Signature verification passes
- [ ] Replay verification passes
- [ ] State root matches after import

### Merge
- [ ] Common ancestor found
- [ ] Divergent receipts extracted
- [ ] Remote receipts replayed
- [ ] State root matches peer's
- [ ] Merge only after verification

## Snapshot Compliance

### Export
- [ ] Snapshot includes protocol versions
- [ ] Snapshot includes receipt chain
- [ ] Snapshot includes storage state
- [ ] Snapshot includes state root
- [ ] Snapshot versioned

### Import
- [ ] Version compatibility checked
- [ ] Protocol version match checked
- [ ] Storage restored correctly
- [ ] Receipt chain restored correctly
- [ ] State root verified after restore
- [ ] Rollback on failure

## Error Handling Compliance

### Error Detection
- [ ] Errors detected at appropriate points
- [ ] Error messages descriptive
- [ ] Errors logged for debugging

### Error Response
- [ ] Safe mode entered on critical errors
- [ ] Execution stopped on capability errors
- [ ] Permission requests returned (not errors)
- [ ] Rollback on composite execution failure

### Error Recovery
- [ ] Recovery paths documented
- [ ] Recovery operations available
- [ ] State can be restored from backup
- [ ] State can be restored from snapshot

## Testing Compliance

### Test Coverage
- [ ] Protocol invariants tested
- [ ] Receipt creation tested
- [ ] State root computation tested
- [ ] Deterministic replay tested
- [ ] Capability enforcement tested

### Test Execution
- [ ] Tests run at boot (self-test)
- [ ] Tests run during development
- [ ] Tests verify protocol compliance
- [ ] Tests catch protocol violations

## Documentation Compliance

### Protocol Documentation
- [ ] Protocol spec matches implementation
- [ ] Receipt schema documented
- [ ] State root schema documented
- [ ] Capability schema documented
- [ ] Version constants documented

### Code Documentation
- [ ] Runtime modules documented
- [ ] dApps documented
- [ ] Integration flows documented
- [ ] Failure modes documented

## Audit Process

### Pre-Commit Checklist
1. Run protocol compliance checklist
2. Verify all invariants pass
3. Run self-tests
4. Verify receipt structure
5. Verify state root structure
6. Verify capability enforcement

### Pre-Release Checklist
1. Complete protocol compliance audit
2. Run full test suite
3. Verify all documentation updated
4. Verify version constants correct
5. Verify protocol spec matches code

## Related Documentation

- [Runtime Invariants](runtime_invariants.md) - Detailed invariants
- [FRAME Protocol Spec](FRAME_PROTOCOL_SPEC.md) - Protocol specification
- [Test Coverage](test_coverage.md) - Test requirements
