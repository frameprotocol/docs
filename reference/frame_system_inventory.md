# FRAME System Inventory — Complete Architectural Analysis

This document is an exhaustive technical inventory of the FRAME codebase. It is intended for engineers designing the next major subsystem (Capsules) and must not omit important infrastructure.

---

## 1. CORE RUNTIME ARCHITECTURE

### 1.1 Kernel (Engine)

**File:** `ui/runtime/engine.js`

**Role:** The kernel is the spine of FRAME. All user-facing execution flows through `handleIntent`. The engine enforces: intent validation, capability resolution, permission checks, integrity lock, code hash verification, receipt building, and chain append. It does **not** implement business logic; it delegates to the router and built-in handlers.

**Exposed on window:**
- `window.FRAME` — Object with `version`, `protocolVersion`, `_expectedCoreHash`, `RECEIPT_VERSION`, `CAPABILITY_VERSION`, `CAPABILITY_SCHEMA`, `mode` (getter), `_coreHash` (getter/setter). `handleIntent` is assigned later (see below).
- `window.FRAME.handleIntent` — Set at end of engine.js to the main async `handleIntent` function.
- `window.FRAME._engine` — Internal engine API (Object.freeze): `installAsyncBlockers`, `installDeterministicSandbox`, `enterReconstructionMode`, `exitReconstructionMode`, `getExecutionCapLog`, `resetExecutionCapLog`, `getNetworkResponseLog`, `resetNetworkResponseLog`, `setNetworkExecutionTimestamp`, `BLOCKED_ASYNC_GLOBALS`, etc.

**State maintained:**
- `_mode` — One of `'normal'`, `'safe'`, `'restoring'`, `'replay'`.
- `_reconstructionMode` — Boolean for deterministic replay.
- `_coreHash` — SHA-256 of runtime file contents (computed asynchronously).
- `_bootStateRoot` — State root at boot; compared before each execution for integrity lock.
- `_bootCodeHashes` — Map dappId → codeHash at boot.
- `_integrityLockReady` — Boolean.
- `_router` — Reference to the router module (set via `registerModule('router', module)`).
- Permissions stored via `FRAME_STORAGE`: key `permissions:${identity}`, value `{ [dappId]: [capability, ...] }`.
- Receipt chain: key `chain:${identity}`, value array of receipt objects.

**Key functions (internal):**
- `validateIntent(intent)` — Requires `intent.action`, `intent.timestamp`, `intent.raw` (all present and correct types).
- `handleBuiltIn(intent)` — Handles all `SYSTEM_INTENTS`: system.explain, system.empty, system.unknown, system.verify, system.safe, system.corehash, system.integrity, system.stateroot, system.export, system.import, system.permissions, system.revoke, system.recover, system.replay, identity.create.
- `kernelNormalize(input)` — String → intent for system commands (e.g. "verify chain" → `{ action: 'system.verify', ... }`).
- `normalizeViaAiDApp(input)` — Calls router.resolve with synthetic `ai.parse_intent` intent, router.execute with AI dApp, returns parsed intent from result.
- `buildScopedApi(dappId, declaredCapabilities)` — Returns `{ api, declaredCapabilities }`. `api` is a frozen object keyed by capability namespace (storage, identity, wallet, contacts, messages, display, network, bridge). Each namespace has methods that call `capabilityGuard(dappId, cap, caps)` then delegate to Tauri or FRAME_STORAGE/FRAME_IDENTITY.
- `capabilityGuard(dappId, requestedCapability, declaredCapabilities)` — Throws if `requestedCapability` not in `declaredCapabilities`.
- `buildReceipt(opts)` — Builds receipt with version, timestamp, identity, dappId, intent, inputHash, inputPayload (canonicalized), capabilitiesDeclared, capabilitiesUsed, previousStateRoot, nextStateRoot, resultHash, previousReceiptHash, receiptHash (sha256 of signable payload). Optionally signs via `__FRAME_INVOKE__('sign_data', { data: receiptHash })`.
- `appendReceipt(identity, receipt)` — Reads `chain:${identity}` from FRAME_STORAGE, appends receipt, writes back; updates `_bootStateRoot` via `FRAME_STATE_ROOT.computeStateRoot`.
- `getPermissions`, `grantPermission`, `hasPermission`, `revokePermission` — All async, use FRAME_STORAGE with `permissions:${identity}`.
- `checkIntegrityLock()` — Compares current state root to `_bootStateRoot`; if mismatch sets mode to safe and returns false.
- `verifyDAppCodeHash(dappId)` — Compares current dApp code hash to `_bootCodeHashes[dappId]`; if mismatch sets safe mode and returns false.
- `installAsyncBlockers()` — Replaces `setTimeout`, `setInterval`, `fetch`, `WebSocket`, etc. with stubs that throw. Returns restore function.
- `installDeterministicSandbox(timestampSeconds, receiptHash)` — Overrides `Date`, `Math.random`, and async globals for replay.

**Interactions:**
- Depends on `window._FRAME_CANONICAL` (canonicalize, sha256, canonicalHash) from `runtime/canonical.js`.
- Uses `window.__FRAME_INVOKE__` for Tauri: get_current_identity, sign_data, get_identity_public_key, storage_read, storage_write (chain and permissions are in JS layer using FRAME_STORAGE which itself uses Tauri storage_read/storage_write).
- Uses `window.FRAME_STORAGE` for chain and permissions.
- Uses `window.FRAME_STATE_ROOT` for computeStateRoot, loadInstalledDApps, computeCodeHash.
- Uses `window.FRAME_BACKUP` for export/import identity (system.export, system.import).
- Registers router via `registerModule('router', routerModule)`; gets it with `getRouter()`.
- handleIntent flow: normalize input to intent → validate → if system intent handleBuiltIn → else router.resolve(intent) → permission check → integrity lock → code hash verify → buildScopedApi → router.execute(match, intent, buildScopedApi) → buildReceipt → appendReceipt → return.

**Call flow (handleIntent):**
```
handleIntent(input)
  → intent = kernelNormalize(input) || normalizeViaAiDApp(input) || { action: 'system.unknown', ... }
  → validateIntent(intent)
  → if SYSTEM_INTENTS: return handleBuiltIn(intent)
  → match = router.resolve(intent)
  → if safe mode and match: return safe message
  → if replay mode: return error
  → if !match: return unknown intent message
  → identity = __FRAME_INVOKE__('get_current_identity')
  → for each required capability: if !hasPermission(identity, match.id, cap) return permission_request
  → checkIntegrityLock()
  → verifyDAppCodeHash(match.id)
  → previousStateRoot = FRAME_STATE_ROOT.computeStateRoot(identity)
  → resetExecutionCapLog(); resetNetworkResponseLog(); setNetworkExecutionTimestamp(intent.timestamp)
  → execResult = router.execute(match, intent, buildScopedApi)
  → capabilitiesUsed = getExecutionCapLog()
  → nextStateRoot = FRAME_STATE_ROOT.computeStateRoot(identity)
  → chain = FRAME_STORAGE.read(chainKey); previousReceiptHash = chain[chain.length-1].receiptHash
  → receipt = buildReceipt({ intent, identity, dappId, capabilitiesDeclared, capabilitiesUsed, previousStateRoot, nextStateRoot, resultHash, previousReceiptHash, result: execResult })
  → appendReceipt(identity, receipt)
  → return { type: 'dapp', dappId, receipt, ...execResult }
```

---

### 1.2 Router

**File:** `ui/runtime/router.js`

**Role:** Resolves an intent to a dApp (match) and executes the dApp's `run(intent, api)` with a scoped API. Does not touch receipts or permissions; the engine does that. Validates manifest (name, intents array, capabilities array, and FRAME.validateCapabilities(capabilities)).

**Not on window.** The engine exposes `window._FRAME_REGISTER(type, module)`; at the end of its IIFE the router calls `window._FRAME_REGISTER('router', { resolve, execute, queryPreviews })`, so the engine's internal `_router` is set when router.js runs (script order: engine.js then router.js). The engine calls `getRouter()` to obtain the router for resolve and execute.

**State:**
- `dappsBase` — URL base for dapps (e.g. same-origin/dapps/).
- `_previewModuleCache` — Cache of dynamically imported dApp modules for preview.

**Key functions:**
- `getDAppBase()` — Returns `new URL('./dapps/', window.location.href).href`.
- `validateManifest(manifest)` — Requires manifest.name (string), manifest.intents (array of strings), manifest.capabilities (array of strings), and `window.FRAME.validateCapabilities(manifest.capabilities)` (engine validates against CAPABILITY_SCHEMA for runtime-backed capabilities; UI-layer capabilities like ui.theme.apply are not in CAPABILITY_SCHEMA so validation may fail for dApps that only declare UI intents — see note below).
- `loadManifests()` — Calls `window.__FRAME_INVOKE__('list_dapps')` to get dApp ids, then fetches `dapps/${id}/manifest.json` for each, validates, sets manifest.id = id, returns array of manifests.
- `resolve(intent)` — If intent.action starts with `ai.`, uses `FRAME_CAPABILITY_REGISTRY.getBestCapability(action, context)` and finds manifest with that dapp id; else iterates loadManifests() and matches intent.action to manifest.intents (exact or prefix match). Returns `{ id, modulePath, capabilities }` or null.
- `resolveManifestCapabilities(dappId)` — Fetches manifest for dappId, validates, returns sorted capabilities array.
- `execute(match, intent, buildScopedApi)` — Validates match.modulePath is under getDAppBase() and no `..`. Fetches verified caps via resolveManifestCapabilities(match.id). Dynamic import of match.modulePath. Gets `run` from module. Builds scoped API via buildScopedApi(match.id, verifiedCaps). Freezes intent. Calls `window.FRAME._engine.installAsyncBlockers()`. Calls `runFn(intent, built.api)`. Returns `{ type: 'dapp', dappId, ...result }`. Restores async blockers in finally.
- `queryPreviews(fragment)` — Used for live intent preview; loads modules, calls classify/preview; AI_CLASSIFY_TIMEOUT_MS and PREVIEW_TIMEOUT_MS apply.

**Interactions:**
- `list_dapps` is implemented in Tauri (`src-tauri/src/dapps.rs`): reads filesystem under resource dir or cwd for `ui/dapps/*` directories that contain `manifest.json`, returns sorted list of directory names.
- Router does not register itself on window; engine imports router and calls `registerModule('router', routerModule)` (engine.js loads router via script tag order; the router IIFE runs and exports nothing to window; engine.js has logic that expects a module register — see engine.js around line 1232: `if (type === 'router') { _router = Object.freeze(module); }`. So the router must be passed to the engine. In the codebase, engine.js and router.js are separate scripts; engine does not import router. So _router is set elsewhere. Grep shows: engine.js has `getRouter()` that returns _router; _router is set in `registerModule`. So there must be a place that calls registerModule with the router. In engine.js, at the end, we have `window.FRAME.handleIntent = handleIntent` and the engine's init or bootstrap. The router is loaded as a script after the engine. So the engine runs first and _router is null until something sets it. Looking at engine.js again: there is no explicit registerModule call in the snippet we have. So either it's in a different part of engine.js or the router registers itself with the engine. Checking: router.js does not reference FRAME or engine in a way that registers. So engine must get the router reference somehow. In index.html the order is: engine.js then router.js. So when engine.js runs, the router script has not run yet. When router.js runs, it's an IIFE that doesn't attach to window. So the engine's getRouter() would throw "Router module not registered" unless _router is set. So there must be code in engine that loads or receives the router. Reading engine.js full: it might be that the router is assigned in a different way — e.g. engine exports something that the router calls, or the router is passed in via a global. I'll document: "The router module is loaded after the engine (script order). The engine's internal _router is set when the router registers with the engine; the exact registration point is in engine.js (e.g. registerModule or similar)." After re-checking: in engine.js line 1232 we have `if (type === 'router') { _router = Object.freeze(module); }` inside what looks like a function that receives type and module. So something calls that with type 'router' and the router module. The engine might use a dynamic import or a global. For the inventory we can state: "Router is registered with the engine via an internal registration mechanism (see engine.js); the engine calls getRouter() to obtain the router for resolve and execute."
- For AI intents (action starting with `ai.`), router uses FRAME_CAPABILITY_REGISTRY.getBestCapability(action, context) to pick a dApp, then finds that dApp in the manifests from list_dapps. So the AI dApp must be in list_dapps (i.e. present under ui/dapps/ai/) to be resolved.

**Note on validateCapabilities:** The engine's VALID_CAPABILITIES are the frozen list from CAPABILITY_SCHEMA (storage.read, storage.write, identity.read, wallet.send, wallet.read, wallet.balance, contacts.read, contacts.import, messages.send, messages.list, display.timerTick, network.request, bridge.mint, bridge.burn). A dApp that only declares intents like ui.theme.apply or context.query does not have those in CAPABILITY_SCHEMA. So validateManifest in the router calls window.FRAME.validateCapabilities(manifest.capabilities). If the engine's validateCapabilities only allows VALID_CAPABILITIES, then UI-only dApps would fail validation when loaded via loadManifests. So either: (1) the engine allows any string in capabilities for validation, or (2) UI dApps are not in list_dapps and are only registered via FRAME_CAPABILITY_REGISTRY.registerDapp (from dappInstaller or app.js). From router.js: "if (typeof window.FRAME.validateCapabilities === 'function') { if (!window.FRAME.validateCapabilities(manifest.capabilities)) return false; }". So the engine must expose validateCapabilities. In engine.js we have validateCapabilities(caps) which checks every cap is in CAPABILITY_SCHEMA. So dApps whose manifests list only runtime capabilities pass; dApps that list ui.theme.apply would fail. So theme, layout, context, contextArchive, systemOverview are not in list_dapps for the Tauri build (they are in the repo under ui/dapps/) but list_dapps returns directories under the app's resource path. So when running in dev, list_dapps might return all dirs under ui/dapps including theme, layout, etc. In that case their manifests would need to pass validateManifest — and their capabilities arrays might need to be subsets of VALID_CAPABILITIES or the engine would need to allow "unknown" capabilities for UI. This is an important nuance: the frozen CAPABILITY_SCHEMA is for receipt-tracked, permission-gated capabilities. UI intents might be intended to be permission-free and not in the schema. The inventory should state this clearly.

---

### 1.3 Storage (Runtime)

**File:** `ui/runtime/storage.js`

**Role:** Thin wrapper over Tauri `storage_read` / `storage_write`. Enforces canonical JSON (no NaN, Infinity, Date, RegExp, Map, Set, circular refs, etc.). Deep-freezes written values. All mutations go through this layer for deterministic replay and state root.

**Exposed:** `window.FRAME_STORAGE` = Object.freeze({ read, write, validateCanonicalJson }).

**Functions:**
- `read(key)` — `window.__FRAME_INVOKE__('storage_read', { key })`.
- `write(key, value)` — Validates value with validateCanonicalJson, deep-clones and freezes, then `window.__FRAME_INVOKE__('storage_write', { key, value })`.
- `validateCanonicalJson(value, path, seen)` — Throws if value contains non-canonical types.

**State:** None; stateless wrapper.

**Interactions:** Called by engine for permissions and receipt chain; by stateRoot for gathering storage data; by backup/replay. Tauri implements actual persistence in `src-tauri/src/storage.rs`.

---

### 1.4 Identity (Vault)

**File:** `ui/runtime/identity.js`

**Role:** Thin wrapper over Tauri identity commands. No state; all operations delegate to `__FRAME_INVOKE__`.

**Exposed:** `window.FRAME_IDENTITY` = Object.freeze({ list, getCurrent, switch, create, getCapabilities }).

**Functions:**
- `list()` — list_identities
- `getCurrent()` — get_current_identity
- `switch(identity_id)` — switch_identity
- `create()` — create_identity
- `getCapabilities()` — get_identity_capabilities

**Interactions:** Engine uses getCurrent for permission and receipt chain identity; buildScopedApi exposes identity.read as getCurrent/list. Tauri: `src-tauri/src/identity.rs`.

---

### 1.5 Canonical

**File:** `ui/runtime/canonical.js`

**Role:** Deterministic serialization and hashing. Single source of truth for canonicalize and sha256 used in receipts and state root.

**Exposed:** `window._FRAME_CANONICAL` = Object.freeze({ canonicalize, sha256, canonicalHash }). Documented as "deleted by invariants.js at boot" — so it may be removed from window after boot for security.

**Functions:**
- `canonicalize(value)` — Recursively sorts object keys and canonicalizes values; arrays and plain objects only.
- `sha256(str)` — Returns Promise of hex string (crypto.subtle.digest).
- `canonicalHash(value)` — sha256(JSON.stringify(canonicalize(value))).

---

### 1.6 State Root

**File:** `ui/runtime/stateRoot.js`

**Role:** Computes a single SHA-256 hash representing the entire state: identity, installed dApps (with code hash), storage, receipt chain commitment. Used for integrity lock and receipt fields (previousStateRoot, nextStateRoot).

**Exposed:** `window.FRAME_STATE_ROOT` — Object with computeStateRoot, loadInstalledDApps, computeCodeHash, and related. Used by engine for receipt building and integrity check.

**State:** None persistent; computes on demand.

**Key functions:**
- `computeCodeHash(dappId)` — Fetches dapps/${dappId}/index.js, returns sha256(source).
- `loadInstalledDApps()` — list_dapps, then for each fetches manifest and computeCodeHash, returns array of { id, manifest, codeHash }.
- `gatherStorageData()` — __FRAME_INVOKE__('storage_list_keys') then storage_read_by_basename for each.
- `computeStateRoot(identityOverride)` — Builds object with version, identityPublicKey, installedDApps (with codeHash), storage (from gatherStorageData), receiptChainCommitment (from chain), then canonicalize and sha256.

**Interactions:** Engine calls computeStateRoot before/after execution and for verifyChain; uses loadInstalledDApps in initIntegrityLock; uses computeCodeHash in verifyDAppCodeHash.

---

### 1.7 Capability Registry

**File:** `ui/system/capabilityRegistry.js`

**Role:** Scans installed dApps (via list_dapps), reads manifest capabilitySchemas, builds indexes by intent and type, scores capabilities (trust, latency, cost, workflow memory, federated memory, reputation). Used by router for AI intent routing (getBestCapability) and by UI/context archive client for context.archive.

**Exposed:** `window.FRAME_CAPABILITY_REGISTRY` = Object.freeze({ scan, rescan, registerDapp, getCapabilities, getCapability, getCapabilitySync, getCapabilitiesByIntent, getCapabilitiesByType, getBestCapability, getBestCapabilitySync, getProvidersForCapability, getCapabilityGraph, getCapabilityGraphSync, searchCapabilities, getCapabilityContextString, computeScore, getIntents, getTypes, VALID_TYPES, estimatePromptTokens, computePreferenceScore }).

**State:**
- `_index` — id → capability entry.
- `_list` — Array of all entries.
- `_intentIndex` — intent string → array of entries.
- `_typeIndex` — type → array of entries.
- `_graph` — { nodes, edges } for capability graph.
- `_ready`, `_scanPromise` — Scan state.

**Key functions:**
- `invoke(name, args)` — __FRAME_INVOKE__(name, args). Used for list_dapps in scan.
- `fetchManifest(dappId)` — Fetches dapps/${dappId}/manifest.json (relative to current origin).
- `buildEntry(cap, manifest, dappId)` — Builds { id, dapp, dappName, description, type, intent, input, output, examples, cost, latency, trust_score, locality, compute_capability }.
- `scan()` — invoke('list_dapps'), then for each id fetchManifest, for each capabilitySchema buildEntry, fill _index, _list, _intentIndex, _typeIndex, _graph. Sets _ready = true.
- `registerDapp(manifest)` — For each capabilitySchemas entry, buildEntry and push to _index, _list, _intentIndex (by intent and by id), _typeIndex. Rebuilds _graph. Does not call list_dapps; used to add capabilities from dynamically installed dApps (e.g. from URL or system overview).
- `getBestCapability(intent, options)` — ensureScanned(), then candidates = _intentIndex[intent], scoreCapability for each, return best.
- `scoreCapability(cap, intent, options)` — Uses computePreferenceScore (AI preference for local), trust_score, latency, cost, FRAME_WORKFLOW_MEMORY (success rate), FRAME_FEDERATED_MEMORY, FRAME_CAPABILITY_REPUTATION.

**Interactions:** Router uses getBestCapability for ai.* intents. Context archive client uses getBestCapability('context.archive'). dappInstaller calls registerDapp after install. app.js fetches systemOverview manifest and registerDapp at boot.

---

### 1.8 Intent Router (AI)

**File:** `ui/system/aiCommandRouter.js`

**Role:** Universal AI-to-system interface. Parses natural language with COMMAND_PATTERNS (regex); if no pattern matches, infers AI capability (e.g. ai.parse_intent, ai.chat, ai.summarize) and calls FRAME.handleIntent with that action and payload { text }. If pattern matches, runs cmd* handlers (wallet, deploy worker, store data, build dapp, show network, etc.). All execution goes through FRAME.handleIntent for capability intents.

**Exposed:** `window.FRAME_AI_COMMANDS` = Object.freeze({ execute, parseCommand, inferAiCapability, AI_CAPABILITIES }).

**State:** None persistent.

**Key functions:**
- `parseCommand(text)` — Runs COMMAND_PATTERNS in order; returns { command, args, originalText } or null.
- `inferAiCapability(text)` — Returns one of ai.summarize, ai.code_generate, ai.image_generate, ai.embed, ai.chat, or ai.parse_intent based on keywords.
- `execute(text)` — If !parseCommand(text), returns executeFrameIntent(inferAiCapability(text), { text }, text). Else switch on parsed.command and run cmdWalletCreate, cmdWalletSend, cmdBuildDapp, etc. executeFrameIntent calls FRAME.handleIntent(action, args).
- Command handlers use FRAME_WALLET, FRAME_NODE_DEPLOY, FRAME_DSTORE, FRAME_AGENT_TEMPLATES, FRAME_AGENTS, FRAME_PEER_ROUTER, FRAME_P2P, FRAME_PEER_DISCOVERY, FRAME_CREDIT_LEDGER, FRAME_AI_BUILDER, FRAME.handleIntent.

**Interactions:** app.js handleCommand calls `router.execute(trimmed)` where router is FRAME_AI_COMMANDS. So user command → FRAME_AI_COMMANDS.execute(text) → either direct cmd* or FRAME.handleIntent(ai.*, { text }).

---

### 1.9 Peer Router

**File:** `ui/system/peerRouter.js`

**Role:** Maintains a peer table (nodeId, address, capabilities, reputation, latency, lastSeen, routeSuccesses, routeFailures). Classifies peers into locality rings (ring1–ring4). Routes intents to nearest capable peers. Does not execute intents; execution stays local via FRAME.handleIntent. Stores peers in localStorage (frame_peer_router).

**Exposed:** `window.FRAME_PEER_ROUTER` — Object with getPeers, registerPeer, updatePeer, getPeer, removePeer, clearPeers, routeIntent, handleIntentResult, and ring/locality helpers.

**State:**
- Peer table in localStorage key `frame_peer_router` (max MAX_PEERS 32).
- `_routingRings` — ring1..ring4 arrays, recalculated from peer table.
- Route history in localStorage `frame_route_history` (max 500).

**Interactions:** Used by widgetManager (network.status), app.js (status bar), networkViz, p2pTransport (handleIntentResult). Does not perform network I/O itself; peer table is populated by peerDiscovery and p2pTransport. Intent routing to remote peers would require p2p send of intent and receive of result — that flow exists in peerRouter (routeIntent) and p2pTransport (intent_route, intent_result).

---

### 1.10 P2P Transport

**File:** `ui/system/p2pTransport.js`

**Role:** WebRTC DataChannels for peer-to-peer messaging. Supports capsule_announce, capsule_request, capsule_data, intent_route, intent_result, auction_bid, swarm_update, peer_exchange, ping/pong, auth, storage_*. Manages connections (_connections), rate limiting, send/receive. When intent_result is received, calls peerRouter.handleIntentResult. Signaling is manual (copy/paste) for prototype; bootstrap nodes use wss URLs.

**Exposed:** `window.FRAME_P2P` — Object with getConnectionCount, getStats, send, connect, disconnect, onMessage, etc.

**State:**
- `_connections` — peerId → PeerConnection.
- `_listeners` — Message listeners.
- `_stats` — sent, received, dropped.

**Interactions:** Peer discovery may use p2p to establish connections. PeerRouter may use p2p to send intent_route and receive intent_result. NetworkViz subscribes to p2p events.

---

### 1.11 Activity System

**File:** `ui/system/activityFeed.js`

**Role:** Real-time event stream for the UI. Push entries (type, message, timestamp), render to a feed DOM, optional chat area for user/assistant messages. Hooks into agent runtime, intent market, network viz, wallet. Each pushEntry also pushes to FRAME_CONTEXT_MEMORY (addEvent, addRollingEvent).

**Exposed:** `window.FRAME_ACTIVITY` = Object.freeze({ mount, mountChat, pushEntry, renderAll, getEntries, clear, hookSystems, startPolling, onEvent, addUserMessage, addAssistantMessage, updateAssistantMessage, addResultMessage, addErrorMessage }).

**State:**
- `_entries` — Array of entries (max MAX_ENTRIES 50).
- `_containerEl`, `_chatEl` — DOM refs.
- `_listeners` — Event listeners for 'entry'.
- `_pollTimer` — Polling timer.

**Interactions:** pushEntry calls FRAME_CONTEXT_MEMORY.addEvent and addRollingEvent. hookSystems subscribes to FRAME_AGENTS, FRAME_WIDGETS, FRAME_INTENT_MARKET, FRAME_NET_VIZ, FRAME_WALLET to push entries. app.js mounts activity feed and chat to DOM.

---

### 1.12 Widget System

**File:** `ui/system/widgetManager.js`

**Role:** Creates, stores, renders, removes, and persists widgets. Widgets have id, title, type, size, priority, dataSourceId/dataSource, refreshInterval, panel, config, region. Data sources are functions returning { items: [{ label, value }] }. Widgets are rendered into region containers (dock, sidebar, workspace, statusbar, or dynamic #frame-region-*). Persistence in localStorage key `frame_widgets`.

**Exposed:** `window.FRAME_WIDGETS` = Object.freeze({ createWidget, createBuildingWidget, updateWidgetStep, finishBuild, removeWidget, getWidgets, getWidget, refreshWidget, refreshAll, mount, renderAll, rehydrate, installDefaults, expandWidgetPreview, onEvent, offEvent, registerDataSource, renderWidgetCard, resolveDataSource, BUILD_STEPS, DATA_SOURCES }).

**State:**
- `_widgets` — Array of widget objects.
- `_containerEl` — Legacy single container.
- `_containers` — { dock, sidebar, workspace, statusbar } DOM refs.
- `_refreshTimers` — widget id → setInterval id.
- `_listeners` — created, removed, refreshed listeners.

**Key functions:**
- `createWidget(opts)` — Normalizes opts (region default 'workspace'), creates widget object, pushes to _widgets, FRAME_LAYOUT.ensureWidgetInLayout(widgetId, region), save(), emit('created'), startRefresh if interval.
- `removeWidget(id)` — FRAME_LAYOUT.removeWidgetFromLayout(id), stopRefresh, splice, save(), emit('removed').
- `mount(containerElOrContainers)` — If object, _containers = { dock, sidebar, workspace, statusbar }; else _containerEl = containerElOrContainers.
- `getContainerForRegion(regionId)` — _containers[regionId] or document.getElementById('frame-region-' + regionId).
- `getWidgetsByRegion()` — FRAME_LAYOUT.getLayout() and listRegions(); for each region, widget ids from layout.regions[region].widgets; unassigned widgets go to workspace.
- `renderAll()` — For each region in listRegions(), get container via getContainerForRegion, get widget ids for that region, sort by priority, clear container, append renderWidgetCard for each.
- DATA_SOURCES: wallet.balances, network.status, agents.list, credits.balance, system.status, brain.status, local.ai, theme.manager, notes.recent. AI dApp registers ai.local.status.

**Interactions:** FRAME_LAYOUT for ensureWidgetInLayout, removeWidgetFromLayout, getLayout, listRegions. FRAME_ACTIVITY for pushEntry on step update. Workspace manager for openPanel. Theme manager for theme widget. app.js mounts containers and calls rehydrate, installDefaults, onEvent(renderAll).

---

### 1.13 Layout System

**File:** `ui/system/layoutManager.js`

**Role:** Single source of truth for region layout. Regions: dock, sidebar, workspace, statusbar (built-in) plus any dynamic regions. Each region has id, position, widgets (array of widget ids), type (panel | dock | workspace), createdAt. Persisted in localStorage key `frame_layout`. Notifies listeners on setLayout, updateLayout, createRegion, removeRegion.

**Exposed:** `window.FRAME_LAYOUT` = Object.freeze({ getLayout, setLayout, updateLayout, resetLayout, getRegionForWidget, moveWidgetToRegion, ensureWidgetInLayout, removeWidgetFromLayout, createRegion, removeRegion, listRegions, cleanupEmptyDynamicRegions, onChange, offChange, BUILTIN_REGIONS, DEFAULT_LAYOUT }).

**State:**
- `_layout` — In-memory layout object (regions keyed by id). Loaded from localStorage on init.
- `_changeListeners` — Array of functions called on emitChange().

**Key functions:**
- `normalizeLayout(layout)` — Ensures built-in regions have id, position, widgets, type, createdAt; dynamic regions from layout.regions get defaultRegionConfig and id, createdAt.
- `createRegion(regionId, config)` — For non-built-in id, sets layout.regions[id] = defaultRegionConfig with id and createdAt, save, emitChange.
- `ensureWidgetInLayout(widgetId, region)` — If region missing (and not built-in), creates region; appends widgetId to region.widgets, save.
- `removeRegion(regionId)` — Moves region's widgets to workspace, deletes region, save, emitChange.
- `cleanupEmptyDynamicRegions(minAgeMs)` — Removes dynamic regions with 0 widgets and createdAt older than minAgeMs.

**Interactions:** Widget manager uses getLayout, listRegions, ensureWidgetInLayout, removeWidgetFromLayout, moveWidgetToRegion. app.js ensureRegionContainers() creates #frame-region-* DOM for dynamic regions and subscribes to onChange to re-run and renderAll.

---

### 1.14 Context Engine

**File:** `ui/system/contextEngine.js`

**Role:** Collects situational state for adaptive UI and relevance scoring. No persistence; getContext() aggregates from other systems.

**Exposed:** `window.FRAME_CONTEXT_ENGINE` = Object.freeze({ getContext }).

**getContext() returns:**
- activeWidgets — from FRAME_WIDGETS.getWidgets() (id, title, type, dataSourceId, region).
- installedDapps — from FRAME_DAPP_INSTALLER.listInstalledDapps().
- recentActivity — last 10 from FRAME_ACTIVITY.getEntries() (type, message slice, timestamp).
- rollingActivity — FRAME_CONTEXT_MEMORY.getRollingContext().
- systemMetrics — { cpu: 0, memory: 0, network: 'idle'|'active' } (network from peer router + p2p connection count).
- timeOfDay — 'morning'|'afternoon'|'night' from current hour.

**Interactions:** Used by uiPlanner (generateUIPlan), app.js executeAdaptiveUIPlan.

---

### 1.15 Context Memory (Tiered)

**File:** `ui/system/contextMemory.js`

**Role:** Three tiers: (1) Short-term: RAM, max 50 entries, addEvent, getRecent, clear. (2) Rolling: localStorage key `frame_context_rolling`, max 200 events, 10 MB; events older than 24h are summarized (type: 'summary', description, startTime, endTime); addRollingEvent, getRollingContext, compressOldEvents, trimRollingToSize. (3) Archive pointer: localStorage key `frame_context_archive`, { node, capability, lastSync }; setArchiveNode, getArchiveNode.

**Exposed:** `window.FRAME_CONTEXT_MEMORY` = Object.freeze({ shortTerm, rolling, archivePointer, addEvent, getRecent, clear, addRollingEvent, getRollingContext, compressOldEvents, setArchiveNode, getArchiveNode, trimRollingToSize }).

**Interactions:** Activity feed pushEntry calls addEvent and addRollingEvent. Context engine getContext uses getRollingContext. Archive client uses getArchiveNode (optional). Context query and context.archive handlers read getRollingContext.

---

### 1.16 Context Archive Client

**File:** `ui/system/contextArchiveClient.js`

**Role:** Queries remote or local context.archive capability. Uses FRAME_CAPABILITY_REGISTRY.getBestCapability('context.archive'); if found, calls FRAME.handleIntent('context.archive', { query: { timeRange, limit } }). Returns result (summary, type) or null. No raw HTTP.

**Exposed:** `window.FRAME_CONTEXT_ARCHIVE_CLIENT` = Object.freeze({ queryArchive }).

**Interactions:** app.js context query flow: if local context insufficient (FRAME_CONTEXT_ANALYZER.hasSufficientLocalContext), calls queryArchive({ timeRange: 'today' }); if result.summary, display it; else fallback to local context.query.

---

### 1.17 Additional Runtime Modules

- **canonical.js** — See 1.5.
- **stateRoot.js** — See 1.6.
- **backup.js** — Export/import identity (used by system.export, system.import). `window.FRAME_BACKUP`.
- **replay.js** — Replay execution log (deterministic re-execution). Used by system.replay.
- **federation.js** — Federation-related logic (referenced in codebase).
- **contracts.js** — Contract runtime (referenced in codebase).
- **snapshot.js** — Snapshot/restore (referenced in engine).
- **invariants.js** — Boot-time invariant checks; may delete _FRAME_CANONICAL from window.

---

## 2. INTENT / CAPABILITY EXECUTION PIPELINE

### 2.1 User Command Entry

**File:** `ui/app.js`

- User types in `#ai-input` and presses Enter.
- `handleCommand(text)` is called.
- `FRAME_ACTIVITY.addUserMessage(trimmed)` and `pushEntry({ type: 'intent', message: 'Command: ' + trimmed.slice(0, 60) })`.

### 2.2 Pre-Intent Short-Circuits (UI Layer)

**File:** `ui/app.js`

Before calling the AI command router, handleCommand checks:

1. **isOverviewQuery(trimmed)** — e.g. "show what is happening on my frame", "system overview". If true: addAssistantMessage('Showing system overview...'), `executeAdaptiveUIPlan(trimmed)`, refreshWidgets, updateStatusBar, return.
2. **isContextQuery(trimmed)** — e.g. "what happened earlier today". If true: addAssistantMessage('Checking context...'), get rolling context, run FRAME_CONTEXT_ANALYZER.hasSufficientLocalContext(rolling). If insufficient, `FRAME_CONTEXT_ARCHIVE_CLIENT.queryArchive({ timeRange: 'today' })`. If archive returns summary, use it; else `FRAME.handleIntent('context.query', {})`. Update assistant message with summary, return.
3. **isBuildCommand(trimmed)** — Handle build dApp flow separately.

### 2.3 AI Command Router (Parse or Route)

**File:** `ui/system/aiCommandRouter.js`

- `execute(trimmed)` is called.
- If `parseCommand(trimmed)` returns null (no regex match), then `executeFrameIntent(inferAiCapability(text), { text: text }, text)` — i.e. `FRAME.handleIntent('ai.parse_intent' | 'ai.chat' | 'ai.summarize' | ..., { text })`.
- If parseCommand matched, switch on command and run cmd* (e.g. cmdWalletSend, cmdBuildDapp). Those either call FRAME.handleIntent with a specific action (e.g. contacts.resolve, wallet.send) or call other system APIs (FRAME_WALLET, FRAME_AI_BUILDER, etc.) and return a result object.

### 2.4 Intent Normalization (Kernel)

**File:** `ui/runtime/engine.js`

- `handleIntent(input)` is invoked (either from app.js wrapper or from executeFrameIntent).
- **app.js wrapper:** Before calling original handleIntent, if action is `ui.layout.apply`, `ui.region.create`, `context.query`, or `context.archive`, the wrapper handles them and returns a Promise (no router). So those never hit the kernel's router.
- For all other intents: intent = kernelNormalize(input) || normalizeViaAiDApp(input) || { action: 'system.unknown', ... }.
- **kernelNormalize:** Regex on raw string → system intents (system.verify, system.safe, revoke, export, import, permissions, recover, replay, identity.create, etc.).
- **normalizeViaAiDApp:** If kernelNormalize didn't match, build synthetic intent { action: 'ai.parse_intent', payload: { text: raw }, ... }, resolve via router (AI dApp), execute AI dApp, take result.intent and return it (so handleIntent then re-runs with that parsed intent).

### 2.5 Validation and Built-Ins

**File:** `ui/runtime/engine.js`

- `validateIntent(intent)` — action, timestamp, raw required.
- If `intent.action` is in SYSTEM_INTENTS, return `handleBuiltIn(intent)` (no router, no receipt).

### 2.6 Capability Resolution

**File:** `ui/runtime/router.js` + `ui/system/capabilityRegistry.js`

- `router.resolve(intent)`:
  - If action starts with `ai.`: `FRAME_CAPABILITY_REGISTRY.getBestCapability(action, context)` → get best dApp id; loadManifests(), find manifest with that id → return { id, modulePath, capabilities }.
  - Else: loadManifests() (list_dapps + fetch each manifest), match intent.action to manifest.intents (exact or prefix) → return match or null.

### 2.7 Permission Check

**File:** `ui/runtime/engine.js`

- For each capability in match.capabilities, engine calls `hasPermission(identity, match.id, cap)`. If any missing, return `{ type: 'permission_request', dapp, capability, identity, intent }`. UI shows permission modal; on allow, grantPermission is called and handleIntent can be retried.

### 2.8 Integrity and Code Hash

**File:** `ui/runtime/engine.js`

- `checkIntegrityLock()` — current state root vs _bootStateRoot; if mismatch, setMode('safe'), return false.
- `verifyDAppCodeHash(match.id)` — current code hash vs _bootCodeHashes[dappId]; if mismatch, setMode('safe'), return false.

### 2.9 Execution

**File:** `ui/runtime/router.js` + `ui/runtime/engine.js`

- Engine: `previousStateRoot = FRAME_STATE_ROOT.computeStateRoot(identity)`, resetExecutionCapLog, setNetworkExecutionTimestamp.
- `execResult = router.execute(match, intent, buildScopedApi)`.
- **router.execute:** Fetch manifest for match.id to get verified caps. Dynamic import(match.modulePath). module.run(intent, built.api). built.api is from engine.buildScopedApi(match.id, verifiedCaps) — only includes namespaces for which the dApp declared capabilities. capabilityGuard runs on every scoped API call. installAsyncBlockers() during run. Return run result.

### 2.10 Receipt and Chain

**File:** `ui/runtime/engine.js`

- capabilitiesUsed = getExecutionCapLog(), nextStateRoot = FRAME_STATE_ROOT.computeStateRoot(identity), chain = FRAME_STORAGE.read(chainKey), previousReceiptHash = chain[chain.length-1].receiptHash, receipt = buildReceipt({ ... }), appendReceipt(identity, receipt). Return { type: 'dapp', dappId, receipt, ...execResult }.

### 2.11 UI Updates and Activity Log

**File:** `ui/app.js` + `ui/system/activityFeed.js`

- handleCommand receives result from router.execute (which is the result of the AI command router, not necessarily a single handleIntent — the AI command router may have called handleIntent and gotten back a result). app.js updates the assistant message with result.message or result.summary, pushes activity entry. Activity feed already receives entries from pushEntry (from various hooks). Context memory receives entries from activity feed pushEntry (addEvent, addRollingEvent).

**Summary table:**

| Stage | File(s) | Function / mechanism |
|-------|---------|----------------------|
| User command | app.js | handleCommand(text), input #ai-input keydown |
| Pre-intent UI | app.js | isOverviewQuery → executeAdaptiveUIPlan; isContextQuery → archive or context.query |
| Parse or route | aiCommandRouter.js | execute(text) → parseCommand or executeFrameIntent(ai.*, { text }) |
| Intent normalization | engine.js | handleIntent → kernelNormalize, normalizeViaAiDApp |
| Validation | engine.js | validateIntent(intent) |
| Built-ins | engine.js | handleBuiltIn(intent) for SYSTEM_INTENTS |
| Resolve | router.js, capabilityRegistry.js | loadManifests(), resolve(intent), getBestCapability for ai.* |
| Permissions | engine.js | hasPermission(identity, dappId, cap) |
| Integrity | engine.js | checkIntegrityLock, verifyDAppCodeHash |
| Execute | router.js, engine.js | buildScopedApi, router.execute(match, intent, buildScopedApi) |
| Receipt | engine.js | buildReceipt, appendReceipt |
| UI / activity | app.js, activityFeed.js | updateAssistantMessage, pushEntry, FRAME_CONTEXT_MEMORY |

---

## 3. DAPP SYSTEM

### 3.1 Manifest Format

- **Location:** Each dApp lives under `ui/dapps/<id>/` with `manifest.json` and `index.js`.
- **Required (router validateManifest):** `name` (string), `intents` (array of strings), `capabilities` (array of strings). Engine's validateCapabilities checks capabilities against CAPABILITY_SCHEMA (so runtime-backed caps must be in the frozen list).
- **Common fields:** `id`, `version`, `capabilitySchemas` (array of { id, type, intent, description, input, output, examples, cost, latency, trust_score, locality }).
- **dApp Installer (dynamic install from URL):** Requires `name`, `version`, `capabilities`, `entry` (string path). Stores in localStorage `frame_installed_dapps` and registers with capability registry; does not add to list_dapps (Tauri filesystem).

### 3.2 Capability Declaration

- **Runtime (engine):** CAPABILITY_SCHEMA is frozen: storage.read, storage.write, identity.read, wallet.send, wallet.read, wallet.balance, contacts.read, contacts.import, messages.send, messages.list, display.timerTick, network.request, bridge.mint, bridge.burn. A dApp manifest's capabilities array is validated against this for runtime-backed capabilities.
- **UI-layer intents:** ui.theme.apply, ui.layout.apply, ui.region.create, context.query, context.archive, system.overview are handled in app.js handleIntent wrapper or by dApps that are not in list_dapps but registered via registerDapp (e.g. systemOverview at boot). Their capability names are not in CAPABILITY_SCHEMA; they are not permission-gated by the engine.

### 3.3 Intent Handling

- dApp module must export `run(intent, api)`. Intent is frozen; api is the scoped API from buildScopedApi (only namespaces for declared capabilities). run may be async.

### 3.4 Sandbox Execution

- **Async blockers:** During router.execute, engine.installAsyncBlockers() replaces setTimeout, setInterval, fetch, WebSocket, etc. with stubs that throw. So dApp run() cannot do arbitrary async I/O except through the scoped API (storage.read/write, identity, wallet, etc. all go through Tauri invoke or FRAME_STORAGE).
- **Capability guard:** Every scoped API method calls capabilityGuard(dappId, cap, caps) before delegating. So a dApp can only use capabilities it declared.

### 3.5 Installation Process

- **Filesystem (Tauri):** list_dapps reads directories under resource or cwd `ui/dapps/` that contain manifest.json. No install step; drop-in dApps.
- **Dynamic (URL):** FRAME_DAPP_INSTALLER.installDappFromUrl(url) fetches manifest, validates (name, version, capabilities, entry), stores in localStorage frame_installed_dapps, calls FRAME_CAPABILITY_REGISTRY.registerDapp(manifest). Removed via removeDapp(dappId).

### 3.6 Discovery Process

- **Router:** loadManifests() uses list_dapps (Tauri) only. So discovered dApps are only those in the filesystem. Dynamically installed dApps are in the capability registry (so getBestCapability can return them) but are not in loadManifests() — so router.resolve by intent string won't find them unless they're also in list_dapps. So for a URL-installed dApp to be executed by the router, it would need to be in list_dapps or the router would need to merge in installed dApps from frame_installed_dapps. Currently the router only uses list_dapps.
- **Capability registry:** scan() uses list_dapps; registerDapp() adds without list_dapps. So the registry can have more capabilities than the router's manifests. Router resolve for ai.* uses registry getBestCapability then finds that dapp in loadManifests() — so the AI dApp must be in list_dapps. For context.archive, getBestCapability is used by the archive client; execution is via handleIntent('context.archive', payload), which is handled by the app.js wrapper (so no router resolve needed for context.archive when run locally).

### 3.7 How Runtime Loads and Executes a dApp

1. User or system invokes FRAME.handleIntent(action, payload).
2. Engine normalizes to intent, validates, resolves via router.resolve(intent). resolve calls loadManifests() (list_dapps + fetch manifests), matches action to manifest.intents (or for ai.*, getBestCapability then find manifest by dapp id).
3. Engine checks permissions, integrity, code hash; builds scoped API; calls router.execute(match, intent, buildScopedApi). execute fetches manifest again (resolveManifestCapabilities), dynamic import(match.modulePath), gets module.run, freezes intent, installAsyncBlockers(), run(intent, built.api), restore blockers, return result.
4. Engine builds receipt, appends to chain, returns.

### 3.8 Built-In dApps (Present in ui/dapps/)

| id | intents / purpose |
|----|-------------------|
| ai | ai.parse_intent, ai.chat, ai.summarize, ai.explain_request, ai.summarize_result |
| theme | ui.theme.apply |
| layout | ui.layout.apply (manifest; handler in app.js) |
| context | context.query |
| contextArchive | context.archive (manifest; handler in app.js + dApp run) |
| systemOverview | system.overview |
| wallet | wallet.* |
| notes | notes.* |
| timer | timer.* |
| contacts | contacts.* |
| messages | messages.* |
| bridge | bridge.* |
| contracts | contracts.* |
| compute | compute.* |
| maps | maps.* |

(list_dapps returns whatever directories exist under the app's dapps path; the above list is from the repo layout.)

---

## 4. UI SYSTEM

### 4.1 Widget Manager

**File:** `ui/system/widgetManager.js`. See 1.12.

- **Create:** createWidget(opts) — id, title, type, size, priority, region (default workspace), dataSourceId or dataSource function, refreshInterval, panel, config. ensureWidgetInLayout. Persist to frame_widgets.
- **Render:** renderAll() uses FRAME_LAYOUT.getLayout() and listRegions(); getWidgetsByRegion(); for each region, getContainerForRegion(regionId), clear container, sort widgets by priority, append renderWidgetCard(widget). Workspace grid uses --workspace-cols from layout.regions.workspace.gridColumns.
- **Move:** moveWidget is on FRAME_API: FRAME_LAYOUT.moveWidgetToRegion(widgetId, region); then renderAll(). Layout manager moveWidgetToRegion updates layout and persist.
- **Destroy:** removeWidget(id); layout removeWidgetFromLayout; stopRefresh; splice; save; emit.
- **Persist:** localStorage frame_widgets; serializes id, title, type, size, priority, dataSourceId, refreshInterval, panel, config, createdAt, region. rehydrate() loads and createWidget for each.

### 4.2 Layout Manager

**File:** `ui/system/layoutManager.js`. See 1.13.

- **Regions:** dock, sidebar, workspace, statusbar (built-in); any other id is dynamic. Each has id, position, widgets[], type, createdAt. workspace has gridColumns.
- **Dynamic region creation:** createRegion(regionId, config). app.js ensureRegionContainers() creates DOM #frame-region-<id> in the appropriate slot (frame-regions-left, -right, -center, -top, -bottom) based on position, and subscribes to FRAME_LAYOUT.onChange to re-run.

### 4.3 Regions and Containers

- **HTML:** #frame-sidebar, #frame-workspace (contains #widget-grid), #frame-dock, #frame-statusbar (contains #frame-statusbar-static and #frame-statusbar-widgets). #frame-dynamic-regions with slots for top, left, center, right, bottom. Dynamic regions get #frame-region-<id>.
- **Mount (app.js):** FRAME_WIDGETS.mount({ dock: #frame-dock, sidebar: #frame-sidebar, workspace: #widget-grid, statusbar: #frame-statusbar-widgets }).

### 4.4 AI UI Planner

**File:** `ui/dapps/ai/uiPlanner.js` (ESM)

- **generateUIPlan(prompt, context):** scoreWidgets(context) from relevance.js, filter by MIN_SCORE, take top MAX_PLAN_WIDGETS, return { loadingWidget: { title, text }, region: { id, position }, widgets: [{ type }, ...] }.
- **getSuggestions(context):** isHeavyActivity(context) → suggest system monitor, inspect tasks; else suggest open notes, run AI task, view network.

**File:** `ui/dapps/ai/relevance.js`

- **scoreWidgets(context):** Returns object type → score (0–1). Rules: recent AI activity boosts recent_tasks, ai_status; network active boosts network_activity, brain_status; many widgets boosts recent_tasks, system_status. Sorted descending.

### 4.5 Adaptive UI Execution

**File:** `ui/system/uiPlanExecutor.js`

- **executeUIPlan(plan):** FRAME_UI_STATE.setMode('analysis'), show loading widget (from plan.loadingWidget), FRAME_LAYOUT.createRegion(plan.region.id, { position }), for each plan.widgets createWidget with 150ms delay (resolve options via FRAME_WIDGET_REGISTRY or fallback map), remove loading widget, setMode('normal'), setActivePlan(plan), showSuccess. Widgets created with region = plan.region.id.

**File:** `ui/app.js`

- executeAdaptiveUIPlan(prompt): dynamic import uiPlanner, getContext(), generateUIPlan(prompt, context). If FRAME_UI_PLANNER exists, planner.executeUIPlan(plan); else inline create region and widgets via FRAME_WIDGET_REGISTRY and FRAME_API.

### 4.6 How AI Dynamically Generates UI Surfaces

1. User says "show what is happening on my frame" → isOverviewQuery true.
2. executeAdaptiveUIPlan(trimmed): import uiPlanner, context = FRAME_CONTEXT_ENGINE.getContext(), plan = generateUIPlan(prompt, context).
3. FRAME_UI_PLANNER.executeUIPlan(plan): set mode analysis, create loading widget in workspace, create region (e.g. system_overview right), create widgets in that region with delay, remove loading widget, set mode normal. Widget definitions from FRAME_WIDGET_REGISTRY (cpu_monitor, network_activity, etc.).

---

## 5. CONTEXT MEMORY SYSTEM

### 5.1 Short-Term Memory

- **Storage:** RAM only (array, max 50).
- **API:** addEvent(event), getRecent(limit), clear.
- **Entry shape:** { type, action, message, timestamp } (normalized from activity entry).
- **Source:** FRAME_ACTIVITY.pushEntry → FRAME_CONTEXT_MEMORY.addEvent.

### 5.2 Rolling Context

- **Storage:** localStorage key `frame_context_rolling`. Max 200 events; max 10 MB (trimRollingToSize).
- **API:** addRollingEvent(event), getRollingContext(), compressOldEvents(), trimRollingToSize().
- **Compression:** When count > 200 or any event older than 24h, events older than 24h are grouped by type and replaced with one summary record { type: 'summary', description: 'User activity: N type1s, M type2s', startTime, endTime }.
- **Source:** Same as short-term; pushEntry also calls addRollingEvent.

### 5.3 Archive Pointer

- **Storage:** localStorage key `frame_context_archive`. Value { node, capability: 'context.archive', lastSync }.
- **API:** setArchiveNode(nodeId), getArchiveNode(). No network implementation; pointer only.

### 5.4 Archive Queries

- **Client:** FRAME_CONTEXT_ARCHIVE_CLIENT.queryArchive({ timeRange, limit }) → getBestCapability('context.archive'), then FRAME.handleIntent('context.archive', { query }). Returns { type, summary, timestamps } or null.
- **Handler (local):** app.js handleIntent wrapper: if action === 'context.archive', read FRAME_CONTEXT_MEMORY.getRollingContext(), filter by timeRange (today/yesterday/week), build summary string, return { type: 'context.archive', summary, timestamps }.
- **dApp:** ui/dapps/contextArchive/ run() does the same (getRollingContext, filter, summarize). Used when context.archive is resolved to this dApp (e.g. on archive node).

### 5.5 Context Query Pipeline

1. User: "what happened earlier today" → isContextQuery true.
2. rolling = FRAME_CONTEXT_MEMORY.getRollingContext(), sufficient = FRAME_CONTEXT_ANALYZER.hasSufficientLocalContext(rolling).
3. If !sufficient: archiveRes = FRAME_CONTEXT_ARCHIVE_CLIENT.queryArchive({ timeRange: 'today' }). If archiveRes.summary, use it.
4. Else (or if no archive): FRAME.handleIntent('context.query', {}). Handler (app.js or context dApp) builds summary from getRollingContext() (summaries + recent events), returns { summary }.
5. Display summary in assistant message.

### 5.6 Context Analyzer

**File:** `ui/system/contextAnalyzer.js`

- **hasSufficientLocalContext(rollingContext):** Returns false if length < 3; false if only summary entries and all older than 24h; else true. Used to decide whether to call archive.

---

## 6. NETWORK LAYER

### 6.1 Peer Router

**File:** `ui/system/peerRouter.js`. See 1.9.

- **Implemented:** Peer table (localStorage), locality rings, registerPeer, updatePeer, getPeers, routeIntent (logic to pick peer for an intent), handleIntentResult (record success/failure). Route history in localStorage.
- **Not implemented:** Actual network send of intent to peer (that would be in p2p or a relay).

### 6.2 Capability Routing

- **Peer router** has routeIntent(intent, options) — selects peer by capability match and ring. The actual send would be via p2p or relay; the code may stub or call p2p.send.
- **Intent market** (intentMarket.js): publishIntent, bid, complete intent; storage-backed. "Intent exchange could occur via local network broadcast, federation peers, or relay servers. Not implemented yet."

### 6.3 Node Discovery

**File:** `ui/system/peerDiscovery.js`

- **Implemented:** Bootstrap nodes list (wss URLs), getNodeId, getLocalCapabilities, getLocalReputation, buildAnnouncement, signaling server connection (WebSocket to bootstrap), peer exchange, health check timers. start(), stop(), isRunning(). Registers discovered peers with peer router; attempts WebRTC connect via p2pTransport.
- **Note:** Bootstrap addresses are placeholders (bootstrap1.frame.network etc.); signaling may be stubbed or require real server.

### 6.4 Message Passing

**File:** `ui/system/p2pTransport.js`

- **Implemented:** WebRTC (RTC_CONFIG with STUN/TURN), PeerConnection wrapper, DataChannel, send(msg) with rate limit, onMessage dispatch. VALID_MSG_TYPES include capsule_announce, intent_route, intent_result, etc. connect(peerId, offerOrAnswer) for signaling. handleIntentResult called when intent_result received.
- **Signaling:** Manual copy/paste for prototype. Relay server not implemented in repo.

### 6.5 Intent Forwarding

- **peerRouter.routeIntent:** Picks peer, could call p2p.send(peerId, { type: 'intent_route', intent, ... }). p2pTransport would send to that peer; remote runs handleIntent and sends back intent_result. handleIntentResult updates peer stats. So the wiring exists; end-to-end depends on signaling and both sides running compatible code.

### 6.6 Summary: What Is Implemented vs Stubbed

| Component | Status |
|-----------|--------|
| Peer table (storage, rings) | Implemented |
| Peer discovery (bootstrap, announce) | Implemented (addresses may be placeholders) |
| P2P transport (WebRTC, send/receive) | Implemented |
| Signaling | Manual / stubbed |
| Intent route/result messages | Implemented in p2p; routing logic in peerRouter |
| Capability-based peer selection | Implemented |
| Intent market (publish, bid) | Implemented (storage); no network broadcast |
| LAN discovery | lanDiscovery.js present |
| Relay config | relayConfig.js present |

---

## 7. AI SYSTEM

### 7.1 Local AI dApp

**File:** `ui/dapps/ai/index.js` (ESM)

- **Exports:** run(intent), preview(fragment), classify(fragment). Registers data source 'ai.local.status' with FRAME_WIDGETS.
- **run:** Dispatches to handleParseIntent (payload.text → inference.parseIntent), handleChat, handleSummarize, handleExplainRequest, handleSummarizeResult. Uses enqueue/drainQueue for serialized execution. Pushes activity entries (ai_inference_started, ai_inference_completed).
- **Intent parsing:** inference.parseIntent(text) uses rules + optional model (modelLoader). Returns { intent: { action, payload, raw, timestamp }, source, confidence }.

**File:** `ui/dapps/ai/inference.js` — parseIntent, chat, summarize, explainRequest, summarizeResult (rules and/or model).

**File:** `ui/dapps/ai/modelLoader.js` — Loads Transformers.js model lazily; getStats(); preload scheduled after 2s.

### 7.2 Intent Parsing Flow

- User text → FRAME_AI_COMMANDS.execute(text) → if no command pattern, handleIntent('ai.parse_intent', { text }) → router resolves to ai dApp (if in list_dapps) → run(intent) → handleParseIntent → inference.parseIntent(text) → returns { intent }. Engine’s normalizeViaAiDApp uses the same path to turn raw string into structured intent.

### 7.3 AI UI Planner and Relevance

- **uiPlanner.js:** generateUIPlan(prompt, context), getSuggestions(context). Uses relevance.scoreWidgets(context).
- **relevance.js:** scoreWidgets(context) → { [type]: score }; rules based on recentActivity (AI), systemMetrics.network, activeWidgets.length.

### 7.4 Context Integration

- getContext() includes recentActivity, rollingActivity, systemMetrics, timeOfDay. Used by generateUIPlan and getSuggestions. No direct call from AI inference to context; context is used at the UI plan level.

### 7.5 Multiple AI Models as Capability Providers

- **Capability registry:** getBestCapability('ai.parse_intent', context) returns the best provider (by score). Router then resolves to that dApp’s manifest (from list_dapps). So multiple dApps could advertise ai.parse_intent with different locality/cost; the registry picks one. Only one AI dApp is in the repo (ai); the architecture allows more.

### 7.6 AI Intent Router and Orchestrator (ui/ai/)

- **ui/ai/intentRouter.js:** Tokenizes user text, scores capabilities from FRAME_CAPABILITY_REGISTRY; parseIntent, resolveIntent, selectCapability, buildWorkflow. Execution still via FRAME.handleIntent(). Loaded in index.html after capabilityRegistry.
- **ui/ai/orchestrator.js:** Multi-step workflow orchestration using the intent router; execution via FRAME.handleIntent.

---

## 8. STORAGE SYSTEM

| Data | Location | Key / mechanism |
|------|----------|------------------|
| Receipt chain | Tauri storage (via FRAME_STORAGE) | chain:${identity} |
| Permissions | Tauri storage | permissions:${identity} |
| Layout | localStorage | frame_layout |
| Widgets | localStorage | frame_widgets |
| Context rolling | localStorage | frame_context_rolling |
| Archive pointer | localStorage | frame_context_archive |
| Theme | localStorage | frame_theme (themeManager) |
| Installed dApps (dynamic) | localStorage | frame_installed_dapps |
| Peer router table | localStorage | frame_peer_router |
| Route history | localStorage | frame_route_history |
| Intent market | localStorage | frame_intent_market |
| Credit ledger | localStorage | frame_credit_ledger |
| Activity feed | Not persisted | RAM only (_entries) |
| Short-term context | Not persisted | RAM only |

Tauri storage (storage_read, storage_write) is used for chain and permissions; the rest are localStorage in the UI layer.

---

## 9. SECURITY MODEL

### 9.1 Identity Model

- **Tauri:** create_identity, list_identities, get_current_identity, switch_identity, sign_data, get_identity_public_key, get_identity_capabilities, grant_capability, revoke_capability. Identity/vault is the security principal; keys and signing are in native code.

### 9.2 Capability Permissions

- **Engine:** Before execution, for each capability in match.capabilities, hasPermission(identity, dappId, cap). If missing, returns permission_request; UI can prompt and grantPermission(identity, dappId, capability). Permissions stored in FRAME_STORAGE under permissions:${identity}.

### 9.3 Sandbox Boundaries

- **Scoped API:** dApp only receives api object with namespaces it declared (storage, identity, wallet, contacts, messages, display, network, bridge). Each method is wrapped with capabilityGuard(dappId, cap, caps).
- **Async blockers:** During run(), setTimeout, setInterval, fetch, WebSocket, etc. are replaced with stubs that throw. So no arbitrary I/O.
- **Deterministic sandbox (replay):** Date and Math.random overridden; async globals stubbed.

### 9.4 Intent Validation

- validateIntent requires action (string), timestamp (number), raw (string). Malformed intent returns error before resolve.

### 9.5 What Prevents Malicious dApps

- **Code hash:** At boot, code hashes of each dApp are stored; before execution, current hash is compared. If drift, safe mode.
- **Integrity lock:** State root at boot vs after each execution; mismatch → safe mode.
- **Permission grant:** User must grant capabilities per dApp; revoke available (system.revoke).
- **No privileged dApps:** AI is a normal dApp; no backdoor. Capability schema is frozen; no runtime capability injection.

---

## 10. ECONOMIC / TOKEN LAYER

### 10.1 What Exists

- **Credit ledger** (`ui/system/creditLedger.js`): balance, minted, spent, burned, history in localStorage (frame_credit_ledger). getBalance(), mintCredits(), spendCredits(), canAffordPublish(), chargePublishFee(). PUBLISH_FEE, MIN_REWARD. Used by intent market (publish fee, bid costs).
- **Intent market** (`ui/system/intentMarket.js`): BID_SUBMISSION_COST, MIN_REWARD_MULTIPLIER, ledger.chargePublishFee(), canAffordPublish(). publishIntent charges fee; bids can cost credits.
- **Token adapters** (`ui/system/tokenAdapters/`): ethereumAdapter, solanaAdapter, bitcoinAdapter — wallet/chain integration.
- **Wallet (Tauri):** get_balance, send_payment — actual balance and send.

### 10.2 What Does Not Exist

- No FRAME credits on-chain implementation in this repo (ledger is local).
- No node compensation or resource pricing protocol.
- No marketplace for buying/selling compute or capabilities beyond the intent market’s reward field and bid costs.

---

## 11. MISSING SYSTEMS

*(Capsules are now implemented; see §13 and docs/capsule_system.md.)*

| System | Why needed |
|--------|------------|
| **Distributed compute** | Compute worker / node deployment is referenced (nodeDeployment, computeWorker, deploy worker command) but full distributed job scheduling and execution across nodes is not present. |
| **Payment system** | Wallet and ledger exist; no standardized payment flow for “pay for capability” or escrow. |
| **Resource marketplace** | Intent market has publish/bid; no discovery of resource providers (CPU, storage) or pricing. |
| **Trust / reputation** | capabilityReputation, agentReputation, federatedMemory provide scoring and memory; no global or cross-node trust graph. |
| **Global capability discovery** | Capability registry is local (list_dapps + registerDapp). No protocol to discover capabilities from other nodes or a directory. |

---

## 13. CAPSULE SYSTEM

Capsules are portable execution units (payload + requested capability + execution/payment/privacy policy + origin signature). Implemented in:

- **ui/system/capsule.js** — Canonical format: createCapsule, signCapsule, hashCapsule, verifyCapsule, validateCapsule. Signable payload excludes id/signature; id = sha256(canonical payload). Uses __FRAME_INVOKE__('sign_data') for signing.
- **ui/system/capsuleStore.js** — localStorage key frame_capsules; addCapsule, getCapsule, listCapsules, setStatus (pending | announced | executing | completed | failed), removeCapsule, pruneCapsules.
- **ui/system/capsuleNetwork.js** — FRAME_CAPSULE_NETWORK: announceCapsule, requestCapsule, sendCapsule, receiveCapsule. FRAME_CAPSULES (for p2pTransport): receiveCapsuleAnnounce, handleCapsuleRequest, importCapsule, announceCapsules. Flow: announce → peers request → origin sends capsule_data → peer importCapsule (verify, store, optionally execute).
- **ui/system/capsuleExecutor.js** — canExecuteCapsule (verify, validate, getBestCapability(requestedCapability), privacy restricted ⇒ origin match, payment check). executeCapsule: build intent from capsule.payload (action, payload), FRAME.handleIntent(intent) only; on success set status completed and creditLedger.mintCredits(paymentRule.amount) for executor. No bypass of router or permissions.

**Integration:** p2pTransport.js already had handleCapsuleAnnounce, handleCapsuleRequest, handleCapsuleData; they use window.FRAME_CAPSULES. capabilityRegistry.getBestCapability for capability match; creditLedger for payment; peerRouter for preferredLocality (execution policy). Widget: data source capsule.activity, default_capsule_activity in sidebar. See docs/capsule_system.md.

---

## 12. CURRENT SYSTEM DIAGRAM

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              USER (command input)                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  app.js (handleCommand)                                                          │
│  • isOverviewQuery → executeAdaptiveUIPlan (UI planner + executor)                │
│  • isContextQuery → archive client or context.query → assistant message           │
│  • isBuildCommand → handleBuildCommand                                           │
│  • else → FRAME_AI_COMMANDS.execute(text)                                         │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
          ┌─────────────────────────────┼─────────────────────────────┐
          ▼                             ▼                             ▼
┌──────────────────┐         ┌──────────────────────┐       ┌─────────────────────┐
│ FRAME_UI_PLANNER │         │ FRAME_AI_COMMANDS    │       │ FRAME.handleIntent   │
│ executeUIPlan    │         │ execute(text)        │       │ (app.js wrapper)    │
│ • setMode        │         │ • parseCommand or     │       │ ui.layout.apply,     │
│ • loading widget │         │   handleIntent(ai.*)  │       │ ui.region.create,   │
│ • createRegion   │         │ • cmd* handlers       │       │ context.query,       │
│ • createWidgets │         └───────────┬──────────┘       │ context.archive      │
└──────────────────┘                    │                  └──────────┬──────────┘
          │                             │                             │
          │                             ▼                             │
          │              ┌──────────────────────────────┐             │
          │              │ ui/runtime/engine.js          │             │
          │              │ handleIntent                  │             │
          │              │ • kernelNormalize             │             │
          │              │ • normalizeViaAiDApp (→ AI)   │             │
          │              │ • validateIntent               │             │
          │              │ • handleBuiltIn (system.*)    │             │
          │              │ • router.resolve(intent)      │             │
          │              │ • permissions, integrity      │             │
          │              │ • router.execute(match,..)   │             │
          │              │ • buildReceipt, appendReceipt │             │
          │              └──────────────┬───────────────┘             │
          │                             │                             │
          ▼                             ▼                             ▼
┌──────────────────┐         ┌──────────────────────┐       ┌─────────────────────┐
│ FRAME_LAYOUT     │         │ ui/runtime/router.js │       │ FRAME_STORAGE        │
│ FRAME_WIDGETS    │         │ resolve: loadManifests│      │ (chain, permissions) │
│ FRAME_UI_STATE   │         │   list_dapps, match  │       │ FRAME_STATE_ROOT     │
└──────────────────┘         │ execute: import run() │       └─────────────────────┘
          │                  │   buildScopedApi     │
          │                  └──────────┬──────────┘
          │                             │
          │                             ▼
          │                  ┌──────────────────────┐
          │                  │ dApp run(intent, api) │
          │                  │ • capabilityGuard     │
          │                  │ • async blockers on   │
          │                  └──────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  CONTEXT & MEMORY                                                                │
│  FRAME_CONTEXT_ENGINE.getContext() → activeWidgets, installedDapps, recentActivity│
│  FRAME_CONTEXT_MEMORY → shortTerm (RAM), rolling (localStorage), archive pointer │
│  FRAME_CONTEXT_ARCHIVE_CLIENT.queryArchive → getBestCapability('context.archive') │
│  FRAME_ACTIVITY.pushEntry → addEvent, addRollingEvent                            │
└─────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│  NETWORK (coordination layer)                                                    │
│  FRAME_PEER_ROUTER (peer table, rings, routeIntent)                              │
│  FRAME_P2P (WebRTC, send/recv intent_route, intent_result, capsule_*)            │
│  FRAME_PEER_DISCOVERY (bootstrap, announce)                                      │
│  FRAME_INTENT_MARKET (publish, bid, localStorage)                               │
└─────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│  CAPSULE SYSTEM                                                                  │
│  FRAME_CAPSULE (create, sign, verify, hash, validate)                            │
│  FRAME_CAPSULE_STORE (frame_capsules localStorage, status lifecycle)             │
│  FRAME_CAPSULE_NETWORK (announce, request, send, receive) / FRAME_CAPSULES      │
│  FRAME_CAPSULE_EXECUTOR (execute via handleIntent only, payment, privacy)       │
│  Widget: capsule.activity (pending, executing, completed)                        │
└─────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│  TAURI (native)                                                                   │
│  list_dapps, storage_read, storage_write, list_identities, get_current_identity   │
│  sign_data, get_identity_public_key, send_payment, get_balance, etc.             │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

*End of FRAME System Inventory. This document should be updated when adding or changing subsystems. Capsule system: see §13 and docs/capsule_system.md.*
