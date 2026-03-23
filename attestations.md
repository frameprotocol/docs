# FRAME Attestations

FRAME supports cryptographic attestations for cross-instance verification and state proof.

**File:** `ui/runtime/replay.js`

## Attestation Overview

Attestations are cryptographic proofs of state root and receipt chain commitment, signed by identity. They enable:

- **Cross-instance verification** — Compare state roots across instances
- **State proof** — Prove current state without sharing full chain
- **Federation sync** — Detect divergence and sync

## Attestation Types

### Receipt Attestation

**Function:** `exportAttestation(identity)`

**Structure:**

```javascript
{
  version: 1,
  protocolVersion: "2.2.0",
  receiptVersion: 2,
  capabilityVersion: 2,
  stateRootVersion: 3,
  identityPublicKey: "hex...",
  stateRoot: "hex...",
  receiptChainCommitment: "hex...",
  receiptCount: 100,
  timestamp: 1234567890,
  signature: "hex..."
}
```

**Fields:**

- `version` — Attestation format version (1)
- `protocolVersion` — FRAME protocol version
- `receiptVersion` — Receipt schema version
- `capabilityVersion` — Capability schema version
- `stateRootVersion` — State root schema version
- `identityPublicKey` — Ed25519 public key (hex)
- `stateRoot` — Current state root
- `receiptChainCommitment` — Hash of latest receipt
- `receiptCount` — Number of receipts in chain
- `timestamp` — Attestation timestamp
- `signature` — Ed25519 signature of attestation hash

**Verification:** `verifyAttestation(attestation, publicKeyHex)`

### State Attestation

**Function:** `exportStateAttestation()`

**Structure:**

```javascript
{
  identityPublicKey: "hex...",
  stateRoot: "hex...",
  chainHash: "hex...",
  protocolVersion: "2.2.0",
  receiptVersion: 2,
  capabilityVersion: 2,
  timestamp: 1234567890,
  signature: "hex..."
}
```

**Fields:**

- `identityPublicKey` — Ed25519 public key (hex)
- `stateRoot` — Current state root
- `chainHash` — SHA-256 of canonicalized receipt chain
- `protocolVersion` — FRAME protocol version
- `receiptVersion` — Receipt schema version
- `capabilityVersion` — Capability schema version
- `timestamp` — Attestation timestamp
- `signature` — Ed25519 signature of payload hash

**Verification:** `verifyStateAttestation(attestation, publicKeyHex)`

### Genesis Attestation

**Function:** `exportGenesis()`

**Structure:**

```javascript
{
  identityPublicKey: "hex...",
  protocolVersion: "2.2.0",
  receiptVersion: 2,
  capabilityVersion: 2,
  stateRootVersion: 3,
  timestamp: 1234567890,
  signature: "hex..."
}
```

**Purpose:** Initial attestation for new identity (no state root or chain yet).

## Attestation Process

### Export

1. **Gather state:**
   - Current identity
   - Public key
   - State root (via `computeStateRoot()`)
   - Receipt chain commitment

2. **Build payload:**
   ```javascript
   var payload = canonicalize({
     identityPublicKey: publicKey,
     stateRoot: stateRoot,
     chainHash: chainHash,
     protocolVersion: protocolVersion,
     receiptVersion: receiptVersion,
     capabilityVersion: capabilityVersion,
     timestamp: timestamp
   });
   ```

3. **Hash payload:**
   ```javascript
   var payloadHash = await sha256(JSON.stringify(payload));
   ```

4. **Sign:**
   ```javascript
   var signature = await __FRAME_INVOKE__('sign_data', { data: payloadHash });
   ```

5. **Return attestation:**
   ```javascript
   return Object.freeze({
     ...payload,
     signature: signature
   });
   ```

### Verification

**Process:**

1. **Extract payload** (exclude signature)
2. **Canonicalize payload**
3. **Hash payload**
4. **Verify signature:**
   ```javascript
   var keyBuf = hexDecode(publicKeyHex);
   var sigBuf = hexDecode(attestation.signature);
   var msgBuf = hexDecode(payloadHash);
   var key = await crypto.subtle.importKey('raw', keyBuf, { name: 'Ed25519' }, false, ['verify']);
   var valid = await crypto.subtle.verify({ name: 'Ed25519' }, key, sigBuf, msgBuf);
   ```

5. **Verify protocol versions** match
6. **Return:** `{ valid: true }` or `{ valid: false, reason: "..." }`

## Cross-Instance Verification

**Function:** `compareStateRoots(localAttestation, remoteAttestation)`

**Process:**

1. **Verify both attestations** (signatures valid)
2. **Compare state roots:**
   - If match → Instances synchronized
   - If differ → Divergence detected

3. **Compare receipt counts:**
   - If local < remote → Local behind
   - If local > remote → Local ahead
   - If equal but roots differ → Divergence

**Result:**

```javascript
{
  synchronized: true | false,
  localAhead: boolean,
  remoteAhead: boolean,
  divergence: boolean,
  reason?: string
}
```

## Attestation Properties

**Attestations are:**

- **Public** — No private data included
- **Verifiable** — Can be verified without trust
- **Deterministic** — Same state produces same attestation
- **Signed** — Ed25519 signature proves authenticity

**Attestations do NOT:**

- Include private keys
- Include full receipt chains
- Include storage data
- Mutate state

## Use Cases

### Federation Sync

1. **Export attestation** from local instance
2. **Send to peer** via network
3. **Receive peer attestation**
4. **Compare state roots** — If differ, sync receipts

### State Proof

1. **Export state attestation**
2. **Share with verifier**
3. **Verifier checks signature** and protocol versions
4. **Verifier trusts state root** without full chain

### Divergence Detection

1. **Compare attestations** from multiple instances
2. **Detect divergence** if state roots differ
3. **Investigate** receipt chains to find divergence point

## Invariants

**Attestation functions must:**

- Use `canonicalize()` for payload
- Use `sha256()` for hashing
- Use `sign_data` for signing
- Use `get_identity_public_key` for public key
- Use `computeStateRoot()` for state root
- **NOT** write to storage
- **NOT** append receipts
- **NOT** mutate state

## Related Documentation

- [Replay and Verification](replay_and_verification.md) - Attestation functions
- [Federation and Sync](federation_and_sync.md) - Cross-instance sync
- [State Root](state_root.md) - State root computation
- [Receipt Chain](receipt_chain.md) - Receipt chain commitment
