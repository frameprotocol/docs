# FRAME Networking Model

FRAME includes a networking layer for peer discovery, capability advertisement, and routing.

**File:** `ui/system/network.js`

## Network Overview

**Current:** Simulated (no actual networking)

**Future:** WebRTC-based P2P transport

## Components

### Peer Discovery

**Module:** `FRAME_PEER_DISCOVERY`

**Functions:**

- `start()` — Start peer discovery
- `getPeers()` — Get discovered peers

### Peer Router

**Module:** `FRAME_PEER_ROUTER`

**Functions:**

- `getPeers()` — Get connected peers
- `routeCapabilityRequest(provider, intent)` — Route request to provider

### P2P Transport

**Module:** `FRAME_P2P`

**Functions:**

- `getPeers()` — Get connected peers
- `send(peer, message)` — Send message to peer

### Capability Advertiser

**Module:** `FRAME_CAPABILITY_ADVERTISER`

**Functions:**

- `announceCapabilities()` — Advertise local capabilities

## Network API

**File:** `ui/system/network.js` — `FRAME_NETWORK`

**Functions:**

- `discoverPeers()` — Start peer discovery
- `getPeers()` — Get discovered peers
- `advertiseCapabilities()` — Advertise capabilities
- `findProvidersForCapability(canonicalName)` — Find providers for capability
- `routeCapabilityRequest(provider, intent)` — Route request to provider

## Capability Routing

**Process:**

1. **Find providers** — `findProvidersForCapability(canonicalName)`
2. **Select provider** — Best provider based on scoring
3. **Route request** — `routeCapabilityRequest(provider, intent)`
4. **Execute** — Execute intent on provider (local or remote)

## Network Request Capability

**Capability:** `network.request`

**Deterministic network requests:**

- Requests logged during execution
- Responses sealed in receipts
- Replay uses sealed responses

**Process:**

1. **Request made** — dApp calls `api.network.request(url, options)`
2. **Response received** — Response stored in network log
3. **Sealed in receipt** — Network responses included in `inputPayload.networkResponses`
4. **Replay uses sealed response** — No live network call during replay

## Related Documentation

- [Capability System](capability_system.md) - Network capability
- [Deterministic Execution](deterministic_execution.md) - Network determinism
- [Federation and Sync](federation_and_sync.md) - Federation sync
