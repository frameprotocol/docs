# FRAME Protocol Versions

This document defines the canonical protocol constants for FRAME. These versions control compatibility and must be consistent across all FRAME instances.

## Protocol Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `PROTOCOL_VERSION` | `"2.2.0"` | Overall FRAME protocol version controlling runtime behavior and compatibility |
| `RECEIPT_VERSION` | `2` | Version of the execution receipt schema |
| `CAPABILITY_VERSION` | `2` | Version of the capability taxonomy schema |
| `STATE_ROOT_VERSION` | `3` | Version of the deterministic state root schema |

## Protocol Compatibility

FRAME instances must reject synchronization or federation attempts if:

- Protocol versions differ
- Receipt versions differ
- Capability schema versions differ
- State root schema versions differ

These constraints ensure deterministic replay across instances.

## Version Bump Rules

### PROTOCOL_VERSION

**Must be incremented if any of the following change:**
- Receipt format
- State root structure
- Canonicalization rules
- Deterministic execution model
- Kernel behavior affecting replay

**Example:** Changing receipt field structure requires a protocol version bump.

### RECEIPT_VERSION

**Must be incremented if:**
- New fields are added to receipts
- Receipt hash computation changes
- Signature semantics change
- Field ordering changes

**Current version:** 2

**Receipt v2 fields:**
- `version`, `timestamp`, `identity`, `dappId`, `intent`
- `inputHash`, `inputPayload`, `capabilitiesDeclared`, `capabilitiesUsed`
- `previousStateRoot`, `nextStateRoot`, `resultHash`
- `previousReceiptHash`, `receiptHash`, `publicKey`, `signature`

### CAPABILITY_VERSION

**Must be incremented if:**
- Capability names change
- Capability semantics change
- Schema structure changes

**Note:** Adding new capabilities without changing existing ones does not require a version bump.

**Current version:** 2

**Capability v2 includes:**
- `bridge.burn`, `bridge.mint`
- `storage.read`, `storage.write`
- `identity.read`
- `wallet.send`, `wallet.read`, `wallet.balance`
- `contacts.read`, `contacts.import`
- `messages.send`, `messages.list`
- `display.timerTick`
- `network.request`

### STATE_ROOT_VERSION

**Must be incremented if:**
- Fields included in state root change
- Canonicalization rules change
- Field ordering changes
- Storage inclusion rules change

**Current version:** 3

**State root v3 fields:**
- `version`, `capabilityVersion`
- `identityPublicKey`
- `installedDApps` (with codeHash)
- `storage`
- `receiptChainCommitment`

## Runtime Enforcement

These versions are enforced at boot via `invariants.js`:

```javascript
if (window.FRAME.CAPABILITY_VERSION !== 2) {
  throw new Error('Protocol freeze: CAPABILITY_VERSION must be 2');
}
if (window.FRAME.RECEIPT_VERSION !== 2) {
  throw new Error('Protocol freeze: RECEIPT_VERSION must be 2');
}
if (window.FRAME_STATE_ROOT.STATE_ROOT_VERSION !== 3) {
  throw new Error('Protocol freeze: STATE_ROOT_VERSION must be 3');
}
```

## Version Checking

**Receipt verification:**

```javascript
if (receipt.version !== RECEIPT_VERSION) {
  throw new Error('Receipt version mismatch');
}
```

**State root verification:**

```javascript
if (stateRoot.version !== STATE_ROOT_VERSION) {
  throw new Error('State root version mismatch');
}
```

**Attestation verification:**

```javascript
if (attestation.receiptVersion !== RECEIPT_VERSION ||
    attestation.capabilityVersion !== CAPABILITY_VERSION ||
    attestation.stateRootVersion !== STATE_ROOT_VERSION) {
  throw new Error('Protocol version mismatch');
}
```

## Related Documentation

- [Kernel Runtime](kernel_runtime.md) - Protocol version enforcement
- [Receipt Chain](receipt_chain.md) - Receipt version details
- [State Root](state_root.md) - State root version details
- [Capability System](capability_system.md) - Capability version details
