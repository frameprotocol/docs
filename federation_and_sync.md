# FRAME Federation and Sync

FRAME supports peer-to-peer synchronization via receipt chain exchange.

**File:** `ui/runtime/federation.js`

## Federation Overview

Federation enables:

- **Cross-instance sync** — Sync receipts between instances
- **Divergence detection** — Detect state differences
- **Merge** — Merge receipt chains
- **Verification** — Verify peer state

## Sync Process

### Step 1: Export Attestation

**File:** `ui/runtime/replay.js` — `exportAttestation(identity)`

**Attestation structure:**

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

### Step 2: Exchange Attestations

Peers exchange attestations:

1. **Send attestation** to peer
2. **Receive attestation** from peer
3. **Compare state roots** — If match, instances are synchronized

### Step 3: Receipt Chain Exchange

If state roots differ:

1. **Request receipt chain** from peer
2. **Receive receipt chain**
3. **Verify chain** — Structural and signature verification
4. **Replay receipts** — Replay receipts in order
5. **Compare state roots** — Must match peer's state root

### Step 4: Merge

**File:** `ui/runtime/federation.js` — `mergeReceiptChains(localChain, remoteChain, identity)`

**Process:**

1. **Find common ancestor** — Last matching receipt hash
2. **Extract divergent receipts** — Receipts after common ancestor
3. **Verify remote chain** — Structural and signature verification
4. **Replay remote receipts** — Replay in reconstruction mode
5. **Compare state roots** — Must match peer's state root
6. **Merge chains** — Append remote receipts to local chain

## Divergence Detection

**Divergence occurs when:**

- State roots differ
- Receipt chains differ
- Receipt hashes don't match

**Detection:**

```javascript
if (localStateRoot !== remoteStateRoot) {
  // Divergence detected
  // Merge receipts
}
```

## Verification

**Receipt chain verification:**

- Structural verification (field count, order)
- Chain link verification (previousReceiptHash)
- Signature verification (Ed25519)
- Capability validation

**State root verification:**

- Recompute state root after replay
- Compare with peer's state root
- Must match for sync success

## Network Transport

**Current:** Simulated (no actual networking)

**Future:** WebRTC-based P2P transport

**Modules:**

- `FRAME_PEER_ROUTER` — Peer routing
- `FRAME_P2P` — P2P transport
- `FRAME_PEER_DISCOVERY` — Peer discovery
- `FRAME_CAPABILITY_ADVERTISER` — Capability advertisement

## Related Documentation

- [Replay and Verification](replay_and_verification.md) - Replay process
- [Receipt Chain](receipt_chain.md) - Receipt structure
- [Networking Model](networking_model.md) - Network layer
