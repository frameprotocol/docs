# FRAME Security

This document explains FRAME's security model, including capability-based permissions, sandbox execution, cryptographic receipts, and identity isolation.

## Security Overview

FRAME enforces security through multiple layers:

1. **Capability-Based Permissions**: dApps only access granted capabilities
2. **Sandbox Execution**: dApps run in isolated environment
3. **Cryptographic Receipts**: All state transitions are signed
4. **Identity Isolation**: Each identity has separate storage and chains
5. **Code Hash Verification**: dApp code integrity verified
6. **State Root Verification**: State integrity verified

## Capability-Based Permissions

FRAME uses a capability-based permission system inspired by UCAN (User-Controlled Authorization Networks).

### Capability Taxonomy

Capabilities are defined in a frozen taxonomy:

**Location**: `ui/runtime/engine.js` → `CAPABILITY_SCHEMA`

**Example Capabilities**:
- `storage.read`: Read from scoped storage
- `storage.write`: Write to scoped storage
- `wallet.read`: Read wallet balance
- `wallet.send`: Send payments
- `identity.read`: Read identity information
- `network.request`: Make HTTP requests

**Frozen Schema**: Capability taxonomy cannot be modified after boot.

### Capability Declaration

dApps declare required capabilities in their manifest:

```json
{
  "name": "Wallet",
  "id": "wallet",
  "intents": ["wallet.balance", "wallet.send"],
  "capabilities": ["wallet.read", "wallet.send", "storage.read"]
}
```

### Capability Grants

Capabilities are granted per identity and per dApp:

**Storage Key**: `permissions:<identity>`

**Structure**:
```javascript
{
  "wallet": ["wallet.read", "wallet.send"],
  "notes": ["storage.read", "storage.write"]
}
```

**Grant Process**:
1. User receives permission request
2. User approves or denies
3. Grant stored in `permissions:<identity>`
4. Runtime checks grants before execution

### Capability Enforcement

Capabilities are enforced at multiple levels:

**1. Permission Check** (before execution):
```javascript
if (!hasPermission(identity, dappId, capability)) {
  return { type: 'permission_request', ... };
}
```

**2. Capability Guard** (during execution):
```javascript
function capabilityGuard(capability, fn) {
  return function(...args) {
    logCapabilityUse(capability);
    if (!hasPermission(identity, dappId, capability)) {
      throw new Error(`Capability ${capability} not granted`);
    }
    return fn(...args);
  };
}
```

**3. Scoped API** (only granted capabilities):
```javascript
function buildScopedApi(dappId, grantedCaps) {
  var api = {};
  if (grantedCaps.indexOf('storage.read') !== -1) {
    api.storage = { read: capabilityGuard('storage.read', storageRead) };
  }
  return api;
}
```

### Capability Logging

Used capabilities are logged during execution:

**Location**: `_executionCapLog` in `engine.js`

**Process**:
1. Each capability use logged
2. Logged capabilities included in receipt as `capabilitiesUsed`
3. Receipt certifies which capabilities were actually used

**Receipt Field**:
```javascript
{
  capabilitiesDeclared: ["wallet.read", "wallet.send"],
  capabilitiesUsed: ["wallet.read"]  // Only read was used
}
```

## Sandbox Execution

dApps execute in a deterministic sandbox that prevents unauthorized access:

### Sandbox Isolation

**Time Freezing**:
- `Date.now()` frozen to execution timestamp
- Prevents time-dependent nondeterminism

**Random Seeding**:
- `Math.random()` seeded deterministically
- Prevents random-dependent nondeterminism

**Async Blocking**:
- `fetch()`, `setTimeout()`, `WebSocket` blocked
- Prevents network-dependent nondeterminism

### Scoped API

dApps only receive scoped APIs with granted capabilities:

**Example**:
```javascript
// dApp receives:
{
  storage: {
    read: (key) => { /* capability guard */ }
  },
  wallet: {
    getBalance: () => { /* capability guard */ }
  }
}

// dApp cannot access:
window.localStorage  // Blocked
window.fetch         // Blocked
Date.now()           // Frozen
Math.random()        // Seeded
```

### Code Isolation

dApps are loaded via dynamic import:

```javascript
var dAppModule = await import(`./dapps/${dappId}/index.js`);
var result = await dAppModule.run(intent, scopedApi);
```

**Isolation Guarantees**:
- No direct access to global scope
- No access to runtime internals
- Only scoped API available

## Cryptographic Receipts

All state transitions are cryptographically signed:

### Receipt Signing

**Algorithm**: Ed25519 (Edwards-curve Digital Signature Algorithm)

**Process**:
1. Build receipt object
2. Compute signable payload (excludes `receiptHash`, `signature`)
3. Hash signable payload → `receiptHash`
4. Sign `receiptHash` with identity's private key
5. Include signature and public key in receipt

**Verification**:
```javascript
var signable = receiptSignablePayload(receipt);
var computedHash = await sha256(JSON.stringify(signable));
var isValid = await verifySignature(
  receipt.signature,
  receipt.publicKey,
  computedHash
);
```

### Receipt Chain Linking

Receipts form a hash chain:

**Structure**:
```
Receipt 0: previousReceiptHash = null
Receipt 1: previousReceiptHash = hash(Receipt 0)
Receipt 2: previousReceiptHash = hash(Receipt 1)
...
```

**Security**:
- Tampering detected via hash mismatch
- Chain integrity verified on replay
- Cannot modify past receipts without breaking chain

### Receipt Verification

Receipts are verified during replay:

**Checks**:
1. Receipt structure matches expected fields
2. `receiptHash` matches computed hash
3. Signature valid against public key
4. `previousReceiptHash` matches previous receipt
5. `inputHash` matches `inputPayload`
6. `resultHash` matches execution result

## Identity Isolation

Each identity has isolated storage and receipt chains:

### Identity Storage

**Per-Identity Storage**:
- Receipt chain: `chain_<identity>.log`
- Capability grants: `permissions:<identity>`
- Application state: `storage:<key>` (scoped by identity)

**Isolation**:
- Identities cannot access each other's storage
- Identities cannot access each other's receipt chains
- Identities cannot access each other's capability grants

### Identity Keys

**Key Generation**: Ed25519 keypairs generated per identity

**Storage**: Keys encrypted and stored in identity vault

**Usage**: Keys used for receipt signing and verification

**Isolation**: Keys never leave the Rust backend

## Code Hash Verification

dApp code integrity is verified:

### Code Hash Computation

**Process**:
1. Load dApp source code
2. Compute SHA-256 hash of source
3. Store hash in state root

**State Root Entry**:
```javascript
{
  installedDApps: [
    {
      id: "wallet",
      manifest: {...},
      codeHash: "abc123..."  // SHA-256 of source
    }
  ]
}
```

### Code Hash Verification

**Before Execution**:
1. Compute current dApp code hash
2. Compare to hash in state root
3. Return error if mismatch

**Security**:
- Detects code tampering
- Prevents execution of modified code
- Ensures deterministic replay

## State Root Verification

State integrity is verified via state root:

### State Root Computation

**Components**:
- Identity public key
- Installed dApps (with code hashes)
- Storage state (canonicalized)
- Receipt chain commitment

**Computation**:
```javascript
var stateRoot = {
  version: STATE_ROOT_VERSION,
  capabilityVersion: CAPABILITY_VERSION,
  identityPublicKey: publicKey,
  installedDApps: dapps,  // Sorted by id
  storage: storage,  // Sorted keys
  receiptChainCommitment: commitment
};
var hash = sha256(JSON.stringify(canonicalize(stateRoot)));
```

### Integrity Lock

**Boot-Time State Root**:
- Computed at boot
- Stored as `_bootStateRoot`
- Used for integrity verification

**Verification**:
- Before each execution, verify current state root matches boot state root
- If mismatch, enter safe mode
- Prevents silent state corruption

## Tamper Detection

FRAME detects tampering through multiple mechanisms:

### Receipt Chain Tampering

**Detection**:
- Hash chain verification
- Signature verification
- Receipt structure validation

**Response**:
- Replay fails if chain broken
- State root mismatch detected
- Safe mode triggered

### Code Tampering

**Detection**:
- Code hash verification
- State root code hash comparison

**Response**:
- Execution blocked if hash mismatch
- Safe mode triggered

### State Tampering

**Detection**:
- State root verification
- Integrity lock check

**Response**:
- Safe mode triggered
- Execution blocked

## Safe Mode

FRAME enters safe mode when integrity violations are detected:

### Triggers

- State root mismatch
- Code hash mismatch
- Receipt chain broken
- Runtime code tampering

### Behavior

- Execution blocked
- Only system intents allowed
- Recovery mode enabled
- User notified

### Recovery

- State reconstruction from receipt chain
- Code hash verification
- State root recomputation
- Integrity restoration

## Security Guarantees

FRAME provides the following security guarantees:

1. **Capability Isolation**: dApps only access granted capabilities
2. **Sandbox Execution**: dApps cannot escape sandbox
3. **Cryptographic Integrity**: All state transitions signed
4. **Identity Isolation**: Identities cannot access each other's data
5. **Code Integrity**: dApp code verified via hashes
6. **State Integrity**: State verified via state root
7. **Tamper Detection**: Tampering detected and prevented

These guarantees ensure that FRAME provides a secure execution environment for applications and agents.
