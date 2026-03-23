# FRAME Failure Modes

This document lists every possible runtime failure, how to detect it, how the system responds, and how to recover.

## Failure Categories

### 1. Capability Failures

#### Undeclared Capability

**Failure:** dApp attempts to use capability not declared in manifest.

**Detection:**
- `capabilityGuard()` throws error
- Error message: "Capability denied: dappId has not declared capability"

**Response:**
- Execution immediately stopped
- Error returned to caller
- No receipt generated

**Recovery:**
- Update manifest to declare capability
- Re-execute intent

#### Permission Denied

**Failure:** dApp attempts to use capability without permission grant.

**Detection:**
- `hasPermission()` returns false
- Check happens before execution

**Response:**
- Execution blocked
- Permission request returned: `{ type: 'permission_request', dappId, capability }`
- UI shows permission modal

**Recovery:**
- User grants permission
- Intent execution retried
- Permission stored in `permissions:<identity>`

#### Unknown Capability

**Failure:** Manifest declares capability not in `CAPABILITY_SCHEMA`.

**Detection:**
- `validateCapabilities()` returns false
- Manifest validation fails

**Response:**
- Manifest rejected
- dApp not loaded
- Router skips dApp during resolution

**Recovery:**
- Remove invalid capability from manifest
- Or add capability to `CAPABILITY_SCHEMA` (requires protocol version bump)

### 2. Protocol Failures

#### Receipt Version Mismatch

**Failure:** Receipt version doesn't match `RECEIPT_VERSION`.

**Detection:**
- Receipt verification checks `receipt.version`
- Comparison: `receipt.version !== RECEIPT_VERSION`

**Response:**
- Receipt verification fails
- Chain import rejected
- Replay fails

**Recovery:**
- Cannot recover (version incompatibility)
- Must upgrade runtime or downgrade receipts

#### State Root Version Mismatch

**Failure:** State root version doesn't match `STATE_ROOT_VERSION`.

**Detection:**
- State root computation checks version
- Comparison: `stateRoot.version !== STATE_ROOT_VERSION`

**Response:**
- State root computation fails
- Snapshot import rejected

**Recovery:**
- Cannot recover (version incompatibility)
- Must upgrade runtime or use compatible snapshot

#### Capability Version Mismatch

**Failure:** Capability schema version doesn't match `CAPABILITY_VERSION`.

**Detection:**
- Boot-time check in `invariants.js`
- Comparison: `CAPABILITY_VERSION !== 2`

**Response:**
- Runtime initialization fails
- Boot aborted

**Recovery:**
- Fix capability version constant
- Reboot runtime

#### Protocol Version Mismatch

**Failure:** Protocol version doesn't match `PROTOCOL_VERSION`.

**Detection:**
- Attestation verification
- Snapshot import verification

**Response:**
- Attestation verification fails
- Snapshot import rejected
- Federation sync fails

**Recovery:**
- Cannot recover (version incompatibility)
- Must use compatible protocol version

### 3. Receipt Chain Failures

#### Chain Link Broken

**Failure:** Receipt's `previousReceiptHash` doesn't match previous receipt's hash.

**Detection:**
- Chain verification: `receipt[n].previousReceiptHash !== receipt[n-1].receiptHash`
- First receipt: `previousReceiptHash !== null`

**Response:**
- Chain verification fails
- Replay stops at broken link
- Safe mode entered

**Recovery:**
- Find last valid receipt
- Truncate chain to last valid receipt
- Re-execute from that point

#### Receipt Hash Mismatch

**Failure:** Receipt hash doesn't match recomputed hash.

**Detection:**
- Chain verification recomputes hash
- Comparison: `recomputedHash !== receipt.receiptHash`

**Response:**
- Receipt verification fails
- Chain verification fails
- Safe mode entered

**Recovery:**
- Receipt corrupted
- Must restore from backup or truncate chain

#### Signature Invalid

**Failure:** Receipt signature doesn't verify with public key.

**Detection:**
- Ed25519 signature verification fails
- `crypto.subtle.verify()` returns false

**Response:**
- Receipt verification fails
- Chain verification fails
- Safe mode entered

**Recovery:**
- Receipt tampered or key mismatch
- Must restore from backup or truncate chain

#### Input Hash Mismatch

**Failure:** Input hash doesn't match canonicalized input payload.

**Detection:**
- Chain verification recomputes input hash
- Comparison: `recomputedInputHash !== receipt.inputHash`

**Response:**
- Receipt verification fails
- Chain verification fails

**Recovery:**
- Input payload corrupted
- Must restore from backup

#### Result Hash Mismatch

**Failure:** Result hash doesn't match canonicalized result during replay.

**Detection:**
- Replay recomputes result hash
- Comparison: `recomputedResultHash !== receipt.resultHash`

**Response:**
- Replay fails
- Nondeterminism detected
- Safe mode entered

**Recovery:**
- Execution not deterministic
- Must fix nondeterministic code
- Re-execute from that point

### 4. State Root Failures

#### State Root Mismatch

**Failure:** Computed state root doesn't match receipt's state root.

**Detection:**
- Before execution: `computedRoot !== receipt.previousStateRoot`
- After execution: `computedRoot !== receipt.nextStateRoot`
- During replay: `computedRoot !== receipt.nextStateRoot`

**Response:**
- Integrity lock check fails
- Safe mode entered
- Execution blocked

**Recovery:**
- State corruption detected
- Restore from snapshot
- Or replay from last valid state root

#### Code Hash Mismatch

**Failure:** dApp code hash doesn't match boot-time hash.

**Detection:**
- `verifyDAppCodeHash()` compares current vs boot hash
- Comparison: `currentHash !== bootHash`

**Response:**
- Safe mode entered
- dApp execution blocked

**Recovery:**
- Code drift detected
- Restore original dApp code
- Or update boot hash (if intentional)

#### Integrity Lock Failure

**Failure:** Current state root doesn't match boot-time state root.

**Detection:**
- `checkIntegrityLock()` compares state roots
- Comparison: `currentRoot !== bootRoot`

**Response:**
- Safe mode entered
- State-changing operations disabled

**Recovery:**
- State corruption detected
- Restore from snapshot
- Or replay from boot state

### 5. Determinism Failures

#### Date.now() Used During Execution

**Failure:** dApp uses `Date.now()` during execution (not frozen).

**Detection:**
- Nondeterministic execution
- Replay produces different result hash
- State root mismatch during replay

**Response:**
- Replay fails
- Nondeterminism detected
- Safe mode entered

**Recovery:**
- Fix dApp to use frozen timestamp
- Re-execute from that point

#### Math.random() Used Directly

**Failure:** dApp uses `Math.random()` directly (not seeded).

**Detection:**
- Nondeterministic execution
- Replay produces different result hash
- State root mismatch during replay

**Response:**
- Replay fails
- Nondeterminism detected
- Safe mode entered

**Recovery:**
- Fix dApp to use seeded PRNG
- Re-execute from that point

#### Network Request Without Sealing

**Failure:** Network request made but response not sealed in receipt.

**Detection:**
- Replay makes live network call
- Different response received
- Result hash mismatch

**Response:**
- Replay fails
- Nondeterminism detected

**Recovery:**
- Fix dApp to use `network.request` capability
- Re-execute from that point

#### Async API Used

**Failure:** dApp uses blocked async API (`setTimeout`, `fetch`, etc.).

**Detection:**
- Async blocker throws error
- Error message: "setTimeout blocked during deterministic execution"

**Response:**
- Execution immediately stopped
- Error returned
- No receipt generated

**Recovery:**
- Fix dApp to avoid async APIs
- Re-execute intent

### 6. Storage Failures

#### Non-Canonical JSON

**Failure:** Attempt to store non-canonical JSON value.

**Detection:**
- `validateCanonicalJson()` throws error
- Error message: "Non-canonical JSON value rejected"

**Response:**
- Storage write rejected
- Error thrown
- No state mutation

**Recovery:**
- Fix value to be canonical JSON
- Retry storage write

#### Storage Corruption

**Failure:** Storage data corrupted or invalid.

**Detection:**
- Storage read returns invalid data
- State root computation fails
- Receipt chain read fails

**Response:**
- State root mismatch
- Safe mode entered

**Recovery:**
- Restore from snapshot
- Or restore from backup

### 7. dApp Failures

#### Module Load Failure

**Failure:** dApp module fails to load.

**Detection:**
- `import(modulePath)` throws error
- Error: "Failed to load module"

**Response:**
- Execution fails
- Error returned
- No receipt generated

**Recovery:**
- Fix module path
- Fix module syntax errors
- Re-execute intent

#### Manifest Validation Failure

**Failure:** dApp manifest fails validation.

**Detection:**
- `validateManifest()` returns false
- Error: "Invalid manifest"

**Response:**
- Manifest rejected
- dApp not loaded
- Router skips dApp

**Recovery:**
- Fix manifest structure
- Fix capability declarations
- Re-install dApp

#### Path Traversal Attempt

**Failure:** dApp attempts to load module outside dapps/ directory.

**Detection:**
- Path validation: `modulePath.indexOf('..') !== -1`
- Path validation: `modulePath not under dapps/`

**Response:**
- Execution fails
- Error: "Path traversal detected"
- No receipt generated

**Recovery:**
- Fix module path
- Re-execute intent

#### dApp Execution Error

**Failure:** dApp's `run()` function throws error.

**Detection:**
- `dApp.run()` throws exception
- Error caught by router

**Response:**
- Execution fails
- Error returned
- No receipt generated

**Recovery:**
- Fix dApp code
- Re-execute intent

### 8. Federation Failures

#### Divergence Detected

**Failure:** Local and remote state roots differ.

**Detection:**
- `compareStateRoots()` detects mismatch
- Divergence: `localRoot !== remoteRoot`

**Response:**
- Sync fails
- Divergence logged

**Recovery:**
- Find divergence point
- Merge chains if possible
- Or choose one chain to keep

#### Chain Import Failure

**Failure:** Imported receipt chain fails verification.

**Detection:**
- Chain verification fails
- Structural or signature verification fails

**Response:**
- Import rejected
- Local chain unchanged

**Recovery:**
- Fix remote chain
- Or use different peer

#### Merge Failure

**Failure:** Chain merge fails after replay.

**Detection:**
- Replay produces different state root
- State root doesn't match peer's

**Response:**
- Merge rejected
- Local chain unchanged

**Recovery:**
- Cannot merge (nondeterminism)
- Keep local chain
- Or investigate divergence cause

### 9. Snapshot Failures

#### Snapshot Version Mismatch

**Failure:** Snapshot version doesn't match expected version.

**Detection:**
- `importSnapshot()` checks version
- Comparison: `snapshot.version !== 1`

**Response:**
- Import rejected
- Error: "Unsupported snapshot version"

**Recovery:**
- Use compatible snapshot version
- Or upgrade snapshot format

#### Snapshot Protocol Mismatch

**Failure:** Snapshot protocol version doesn't match runtime.

**Detection:**
- Protocol version comparison
- Capability version comparison
- Receipt version comparison

**Response:**
- Import rejected
- Error: "Protocol version mismatch"

**Recovery:**
- Use compatible snapshot
- Or upgrade runtime

#### Snapshot State Root Mismatch

**Failure:** Restored state root doesn't match snapshot's state root.

**Detection:**
- After restore: `computedRoot !== snapshot.stateRoot`

**Response:**
- Rollback triggered
- Storage restored to pre-import state
- Error thrown

**Recovery:**
- Snapshot corrupted
- Use different snapshot
- Or restore from receipt chain

### 10. Boot Failures

#### Invariant Violation

**Failure:** Boot-time invariant check fails.

**Detection:**
- `invariants.js` assertion fails
- Error: "Invariant: ..."

**Response:**
- Runtime initialization fails
- Boot aborted

**Recovery:**
- Fix invariant violation
- Reboot runtime

#### Replay Determinism Check Failure

**Failure:** Replay determinism check fails at boot.

**Detection:**
- `replayDeterminismCheck()` detects nondeterminism
- Last receipt replay produces different state root

**Response:**
- Boot fails
- Safe mode entered

**Recovery:**
- Fix nondeterministic code
- Or truncate chain to last deterministic receipt
- Reboot

#### State Reconstruction Failure

**Failure:** State reconstruction fails at boot.

**Detection:**
- `reconstructAndVerify()` fails
- Receipt chain verification fails

**Response:**
- Boot fails
- Safe mode entered

**Recovery:**
- Fix receipt chain corruption
- Or restore from snapshot
- Reboot

## Failure Response Matrix

| Failure | Detection | Response | Recovery |
|---------|-----------|----------|----------|
| Undeclared capability | capabilityGuard() | Execution stopped | Update manifest |
| Permission denied | hasPermission() | Permission request | Grant permission |
| Receipt hash mismatch | Chain verification | Safe mode | Restore from backup |
| State root mismatch | Integrity lock | Safe mode | Restore from snapshot |
| Code hash mismatch | verifyDAppCodeHash() | Safe mode | Restore code |
| Nondeterminism | Replay verification | Safe mode | Fix code |
| Storage corruption | State root check | Safe mode | Restore from backup |
| Module load failure | import() | Execution fails | Fix module |
| Chain divergence | compareStateRoots() | Sync fails | Merge or choose |

## Safe Mode Behavior

**When safe mode is entered:**

- State-changing operations disabled
- Only diagnostic operations allowed
- Receipt generation blocked
- dApp execution blocked (except system intents)

**Safe mode triggers:**

- Core hash mismatch
- State root mismatch
- Code hash mismatch
- Receipt chain corruption
- Replay verification failure
- Integrity lock failure

**Exiting safe mode:**

- Restore from snapshot
- Fix corruption
- Reboot runtime

## Related Documentation

- [Runtime Invariants](runtime_invariants.md) - What must be true
- [Test Coverage](test_coverage.md) - Test coverage
- [Debugging Guide](debugging_guide.md) - How to debug failures
