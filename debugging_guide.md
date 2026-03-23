# FRAME Debugging Guide

This guide helps debug FRAME runtime issues. It provides step-by-step procedures for common debugging scenarios.

## Debugging Tools

### Runtime Self-Check

**File:** `ui/system/runtimeSelfCheck.js`

**Function:** `runSelfCheck()`

**Checks:**
- Runtime module availability
- Protocol version consistency
- Receipt chain integrity
- State root consistency

**Usage:**
```javascript
await window.FRAME_SELF_CHECK.runSelfCheck();
```

### Flow Tests

**File:** `ui/dev/runtimeFlowTests.js`

**Function:** `runFrameFlowTests()`

**Tests:**
- Intent resolution
- Execution graph building
- Execution completion
- UI plan generation
- Receipt creation

**Usage:**
- Command bar: `frame test` or `frame.test`
- Programmatic: `await runFrameFlowTests()`

### Dev Mode

**Enable:** `window.FRAME_DEV_MODE = true`

**Effects:**
- Console logging enabled
- Debug messages shown
- Additional diagnostics

## Common Debugging Scenarios

### State Root Mismatch

**Symptom:** State root doesn't match receipt's state root.

**Debugging Steps:**

1. **Recompute state root:**
   ```javascript
   var identity = await __FRAME_INVOKE__('get_current_identity');
   var computedRoot = await FRAME_STATE_ROOT.computeStateRoot(identity);
   console.log('Computed root:', computedRoot);
   ```

2. **Compare with receipt:**
   ```javascript
   var chain = await FRAME_STORAGE.read('chain:' + identity);
   var lastReceipt = chain[chain.length - 1];
   console.log('Receipt nextStateRoot:', lastReceipt.nextStateRoot);
   console.log('Match:', computedRoot === lastReceipt.nextStateRoot);
   ```

3. **Check storage keys:**
   ```javascript
   var keys = await __FRAME_INVOKE__('storage_list_keys');
   console.log('Storage keys:', keys);
   ```

4. **Check excluded prefixes:**
   ```javascript
   // Verify excluded keys are not included
   // Check: chain:, frame_integrity, frame_context_, etc.
   ```

5. **Inspect receipt chain commitment:**
   ```javascript
   var commitment = await FRAME._engine.computeReceiptChainCommitment(chain);
   console.log('Chain commitment:', commitment);
   ```

6. **Replay last receipt:**
   ```javascript
   await FRAME.replayExecutionLog(identity, [lastReceipt]);
   ```

**Common Causes:**
- Storage mutation outside kernel
- Nondeterministic execution
- Code hash mismatch
- Receipt chain corruption

### Capability Failure

**Symptom:** dApp cannot access capability.

**Debugging Steps:**

1. **Check manifest declaration:**
   ```javascript
   var manifest = await fetch('/dapps/<dappId>/manifest.json').then(r => r.json());
   console.log('Declared capabilities:', manifest.capabilities);
   ```

2. **Check permission grant:**
   ```javascript
   var identity = await __FRAME_INVOKE__('get_current_identity');
   var permissions = await FRAME_STORAGE.read('permissions:' + identity);
   console.log('Permissions:', permissions);
   ```

3. **Check capability schema:**
   ```javascript
   console.log('Capability schema:', FRAME.CAPABILITY_SCHEMA);
   console.log('Capability exists:', 'capability.name' in FRAME.CAPABILITY_SCHEMA);
   ```

4. **Check capability guard:**
   ```javascript
   // Enable dev mode to see guard errors
   window.FRAME_DEV_MODE = true;
   ```

5. **Check scoped API:**
   ```javascript
   // Verify scoped API includes capability
   // Check buildScopedApi() output
   ```

**Common Causes:**
- Capability not declared in manifest
- Permission not granted
- Capability not in schema
- Scoped API construction error

### Receipt Chain Corruption

**Symptom:** Receipt chain verification fails.

**Debugging Steps:**

1. **Verify chain structure:**
   ```javascript
   var chain = await FRAME_STORAGE.read('chain:' + identity);
   console.log('Chain length:', chain.length);
   ```

2. **Check chain links:**
   ```javascript
   for (var i = 0; i < chain.length; i++) {
     var receipt = chain[i];
     var expectedPrev = i === 0 ? null : chain[i - 1].receiptHash;
     if (receipt.previousReceiptHash !== expectedPrev) {
       console.error('Broken link at index', i);
       console.error('Expected:', expectedPrev);
       console.error('Got:', receipt.previousReceiptHash);
     }
   }
   ```

3. **Verify receipt hashes:**
   ```javascript
   for (var i = 0; i < chain.length; i++) {
     var receipt = chain[i];
     var signable = FRAME._engine.receiptSignablePayload(receipt);
     var recomputedHash = await FRAME._engine.sha256(JSON.stringify(signable));
     if (recomputedHash !== receipt.receiptHash) {
       console.error('Hash mismatch at index', i);
     }
   }
   ```

4. **Verify signatures:**
   ```javascript
   for (var i = 0; i < chain.length; i++) {
     var receipt = chain[i];
     var valid = await FRAME.verifyReceiptSignatureWithKey(receipt, receipt.publicKey);
     if (!valid) {
       console.error('Invalid signature at index', i);
     }
   }
   ```

5. **Find last valid receipt:**
   ```javascript
   var lastValid = 0;
   for (var i = 0; i < chain.length; i++) {
     try {
       await FRAME.verifyReceiptChain(chain.slice(0, i + 1), identity);
       lastValid = i;
     } catch (e) {
       break;
     }
   }
   console.log('Last valid receipt index:', lastValid);
   ```

**Common Causes:**
- Receipt tampering
- Storage corruption
- Signature mismatch
- Hash computation error

### Determinism Failure

**Symptom:** Replay produces different state root.

**Debugging Steps:**

1. **Check frozen time:**
   ```javascript
   // During execution, Date.now() should be frozen
   // Check installDeterministicSandbox() installation
   ```

2. **Check seeded randomness:**
   ```javascript
   // During execution, Math.random() should be seeded
   // Check installDeterministicSandbox() installation
   ```

3. **Check async blockers:**
   ```javascript
   // During execution, async APIs should be blocked
   // Check installAsyncBlockers() installation
   ```

4. **Check network requests:**
   ```javascript
   // Network responses should be sealed in receipt
   var receipt = chain[chain.length - 1];
   console.log('Network responses:', receipt.inputPayload.networkResponses);
   ```

5. **Replay with debugging:**
   ```javascript
   window.FRAME_DEV_MODE = true;
   await FRAME.replayExecutionLog(identity, chain);
   ```

**Common Causes:**
- Date.now() used directly
- Math.random() used directly
- Async API used
- Network request not sealed
- Non-canonical data stored

### dApp Execution Failure

**Symptom:** dApp execution fails or produces errors.

**Debugging Steps:**

1. **Check manifest:**
   ```javascript
   var manifest = await fetch('/dapps/<dappId>/manifest.json').then(r => r.json());
   console.log('Manifest:', manifest);
   ```

2. **Check module load:**
   ```javascript
   try {
     var module = await import('/dapps/<dappId>/index.js');
     console.log('Module loaded:', module);
   } catch (e) {
     console.error('Module load failed:', e);
   }
   ```

3. **Check scoped API:**
   ```javascript
   // Verify scoped API includes required capabilities
   // Check buildScopedApi() output
   ```

4. **Check intent structure:**
   ```javascript
   console.log('Intent:', intent);
   console.log('Intent frozen:', Object.isFrozen(intent));
   ```

5. **Check error details:**
   ```javascript
   window.FRAME_DEV_MODE = true;
   // Errors will be logged with details
   ```

**Common Causes:**
- Manifest validation failure
- Module load failure
- Missing capability in scoped API
- Intent structure invalid
- dApp code error

### Permission Request Loop

**Symptom:** Permission request keeps appearing.

**Debugging Steps:**

1. **Check permission storage:**
   ```javascript
   var identity = await __FRAME_INVOKE__('get_current_identity');
   var permissions = await FRAME_STORAGE.read('permissions:' + identity);
   console.log('Permissions:', permissions);
   ```

2. **Check permission grant:**
   ```javascript
   // Verify permission was actually granted
   var hasPerm = await FRAME.hasPermission(identity, dappId, capability);
   console.log('Has permission:', hasPerm);
   ```

3. **Check identity:**
   ```javascript
   var identity = await __FRAME_INVOKE__('get_current_identity');
   console.log('Current identity:', identity);
   ```

4. **Check capability declaration:**
   ```javascript
   var manifest = await fetch('/dapps/<dappId>/manifest.json').then(r => r.json());
   console.log('Declared:', manifest.capabilities);
   console.log('Requested:', capability);
   ```

**Common Causes:**
- Permission not actually granted
- Identity mismatch
- Capability not declared
- Storage write failure

## Debugging Utilities

### State Inspection

**Inspect current state:**
```javascript
var identity = await __FRAME_INVOKE__('get_current_identity');
var stateRoot = await FRAME_STATE_ROOT.computeStateRoot(identity);
var chain = await FRAME_STORAGE.read('chain:' + identity);
console.log('State root:', stateRoot);
console.log('Chain length:', chain.length);
console.log('Last receipt:', chain[chain.length - 1]);
```

### Receipt Inspection

**Inspect receipt:**
```javascript
var receipt = chain[chain.length - 1];
console.log('Receipt version:', receipt.version);
console.log('Receipt timestamp:', receipt.timestamp);
console.log('Receipt intent:', receipt.intent);
console.log('Capabilities used:', receipt.capabilitiesUsed);
console.log('State roots:', {
  previous: receipt.previousStateRoot,
  next: receipt.nextStateRoot
});
```

### Capability Inspection

**Inspect capabilities:**
```javascript
console.log('Capability schema:', FRAME.CAPABILITY_SCHEMA);
console.log('Capability version:', FRAME.CAPABILITY_VERSION);

var identity = await __FRAME_INVOKE__('get_current_identity');
var permissions = await FRAME_STORAGE.read('permissions:' + identity);
console.log('Permissions:', permissions);
```

### Storage Inspection

**Inspect storage:**
```javascript
var keys = await __FRAME_INVOKE__('storage_list_keys');
console.log('Storage keys:', keys);

for (var key of keys) {
  var value = await FRAME_STORAGE.read(key);
  console.log(key + ':', value);
}
```

## Debugging Checklist

### Before Reporting Issue

- [ ] Enable dev mode: `window.FRAME_DEV_MODE = true`
- [ ] Run self-check: `await FRAME_SELF_CHECK.runSelfCheck()`
- [ ] Run flow tests: `await runFrameFlowTests()`
- [ ] Check console for errors
- [ ] Verify protocol versions match
- [ ] Verify state root consistency
- [ ] Verify receipt chain integrity

### When Debugging

1. **Reproduce issue** — Can you reproduce it consistently?
2. **Check logs** — What errors appear in console?
3. **Verify state** — Is state root consistent?
4. **Check receipts** — Are receipts valid?
5. **Verify capabilities** — Are capabilities declared/granted?
6. **Check determinism** — Does replay work?

## Related Documentation

- [Runtime Invariants](runtime_invariants.md) - What must be true
- [Failure Modes](failure_modes.md) - Failure scenarios
- [Test Coverage](test_coverage.md) - Test requirements
