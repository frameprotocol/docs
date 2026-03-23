# FRAME Snapshots and Backups

FRAME supports full state snapshots for backup, migration, and recovery.

**File:** `ui/runtime/snapshot.js`

## Snapshot Overview

A snapshot is a complete export of FRAME state:

- Receipt chain
- Storage state
- Protocol versions
- State root
- Identity metadata

Snapshots enable:
- **Backup** — Save state for recovery
- **Migration** — Move state between instances
- **Recovery** — Restore from backup
- **Verification** — Verify state integrity

## Snapshot Structure

**File:** `ui/runtime/snapshot.js` — `exportSnapshot()`

```javascript
{
  version: 1,                          // Snapshot format version
  protocolVersion: "2.2.0",          // FRAME protocol version
  receiptVersion: 2,                  // Receipt schema version
  capabilityVersion: 2,               // Capability schema version
  stateRootVersion: 3,                 // State root schema version
  identity: "identity_id",            // Identity ID
  publicKey: "hex...",                // Ed25519 public key
  stateRoot: "hex...",                // Current state root
  totalSupply: 1000,                  // frame total supply
  receiptChain: [...],                // Full receipt chain
  storage: {                          // Storage state
    key1: value1,
    key2: value2,
    // ...
  }
}
```

## Exporting Snapshots

**Process:**

1. **Gather identity:**
   ```javascript
   var identity = await get_current_identity();
   var publicKey = await get_identity_public_key();
   ```

2. **Load receipt chain:**
   ```javascript
   var chain = await FRAME_STORAGE.read('chain:' + identity);
   ```

3. **Gather storage:**
   ```javascript
   var keys = await storage_list_keys();
   var storageData = {};
   for (var key of keys) {
     storageData[key] = await FRAME_STORAGE.read(key);
   }
   ```

4. **Compute state root:**
   ```javascript
   var stateRoot = await computeStateRoot(identity);
   ```

5. **Gather total supply:**
   ```javascript
   var totalSupply = await FRAME_STORAGE.read('wallet:totalSupply');
   ```

6. **Build snapshot:**
   ```javascript
   return {
     version: 1,
     protocolVersion: window.FRAME.protocolVersion,
     receiptVersion: window.FRAME.RECEIPT_VERSION,
     capabilityVersion: window.FRAME.CAPABILITY_VERSION,
     stateRootVersion: window.FRAME_STATE_ROOT.STATE_ROOT_VERSION,
     identity: identity,
     publicKey: publicKey,
     stateRoot: stateRoot,
     totalSupply: totalSupply,
     receiptChain: chain,
     storage: storageData
   };
   ```

## Importing Snapshots

**File:** `ui/runtime/snapshot.js` — `importSnapshot(snapshot)`

### Validation

**Version checks:**

```javascript
if (snapshot.version !== 1) {
  throw new Error('Unsupported snapshot version');
}
if (snapshot.protocolVersion !== window.FRAME.protocolVersion) {
  throw new Error('Protocol version mismatch');
}
if (snapshot.receiptVersion !== window.FRAME.RECEIPT_VERSION) {
  throw new Error('Receipt version mismatch');
}
if (snapshot.capabilityVersion !== window.FRAME.CAPABILITY_VERSION) {
  throw new Error('Capability version mismatch');
}
```

**Structure checks:**

```javascript
if (!snapshot.storage || typeof snapshot.storage !== 'object') {
  throw new Error('Storage data missing');
}
if (!Array.isArray(snapshot.receiptChain)) {
  throw new Error('Receipt chain must be an array');
}
```

### Import Process

1. **Save current state:**
   ```javascript
   var preImportSnapshot = await gatherStorageSnapshot();
   ```

2. **Clear storage:**
   ```javascript
   await FRAME_STORAGE.clear();
   ```

3. **Write storage:**
   ```javascript
   for (var key of Object.keys(snapshot.storage)) {
     await FRAME_STORAGE.write(key, snapshot.storage[key]);
   }
   ```

4. **Write receipt chain:**
   ```javascript
   await FRAME_STORAGE.write('chain:' + identity, snapshot.receiptChain);
   ```

5. **Verify state root:**
   ```javascript
   var computedRoot = await computeStateRoot(identity);
   if (computedRoot !== snapshot.stateRoot) {
     // Rollback
     await restoreStorageSnapshot(preImportSnapshot);
     throw new Error('State root mismatch');
   }
   ```

6. **Rollback on error:**
   ```javascript
   try {
     // Import process
   } catch (e) {
     await restoreStorageSnapshot(preImportSnapshot);
     throw e;
   }
   ```

## Snapshot Storage

Snapshots are stored as JSON files:

- **Export:** `snapshot.json`
- **Import:** Load from file
- **Backup:** Save to external storage

**Security:**
- Snapshots contain sensitive data
- Encrypt at rest (Tauri backend)
- Verify signatures before import

## Backup Strategy

### Full Backup

Export complete snapshot:

```javascript
var snapshot = await exportSnapshot();
await saveToFile(snapshot, 'backup.json');
```

### Incremental Backup

Export only new receipts:

```javascript
var lastBackupHash = getLastBackupReceiptHash();
var newReceipts = chain.filter(r => r.receiptHash !== lastBackupHash);
var incrementalSnapshot = {
  ...baseSnapshot,
  receiptChain: newReceipts
};
```

### Scheduled Backups

Automated backups:

- Daily full backups
- Hourly incremental backups
- On-demand backups

## Recovery Process

### From Snapshot

1. **Load snapshot:**
   ```javascript
   var snapshot = await loadFromFile('backup.json');
   ```

2. **Verify snapshot:**
   ```javascript
   await verifySnapshot(snapshot);
   ```

3. **Import snapshot:**
   ```javascript
   await importSnapshot(snapshot);
   ```

4. **Verify state:**
   ```javascript
   var stateRoot = await computeStateRoot(identity);
   if (stateRoot !== snapshot.stateRoot) {
     throw new Error('Recovery failed');
   }
   ```

### From Receipt Chain

1. **Load receipt chain:**
   ```javascript
   var chain = await loadReceiptChain();
   ```

2. **Replay receipts:**
   ```javascript
   await replayExecutionLog(identity, chain);
   ```

3. **Verify state:**
   ```javascript
   var stateRoot = await computeStateRoot(identity);
   ```

## Snapshot Verification

**File:** `ui/runtime/snapshot.js` — `verifySnapshot(snapshot)`

**Checks:**

1. **Version compatibility**
2. **Receipt chain integrity**
3. **Storage structure**
4. **State root consistency**

**On failure:**
- Snapshot is invalid
- Import rejected
- Error logged

## Migration

**Moving state between instances:**

1. **Export snapshot** from source instance
2. **Transfer snapshot** to target instance
3. **Import snapshot** on target instance
4. **Verify state** matches source

**Requirements:**
- Same protocol versions
- Same identity (or identity migration)
- Compatible dApps

## Performance Considerations

**Snapshot export:**
- Reads all storage keys
- Computes state root
- Serializes large data structures

**Snapshot import:**
- Writes all storage keys
- Verifies state root
- Replays receipt chain (optional)

**Optimizations:**
- Incremental snapshots
- Compressed storage
- Parallel processing

## Security Considerations

**Snapshots contain:**
- Receipt chains (public)
- Storage state (may contain sensitive data)
- Identity metadata (public keys)

**Protection:**
- Encrypt at rest
- Verify signatures
- Secure transfer

## Summary

FRAME snapshots enable:

- **Backup** — Save state for recovery
- **Migration** — Move state between instances
- **Recovery** — Restore from backup
- **Verification** — Verify state integrity

**Snapshot guarantees:**

- Complete state export
- Version compatibility
- Integrity verification
- Recovery safety

## Related Documentation

- [Receipt Chain](receipt_chain.md) - Receipt structure
- [State Root](state_root.md) - State root computation
- [Replay and Verification](replay_and_verification.md) - Replay process
- [Storage Model](storage_model.md) - Storage structure
