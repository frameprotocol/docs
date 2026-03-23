# FRAME Directory Structure

This document explains the repository layout and the purpose of each directory and key file.

## Root Structure

```
frame/
├── .gitignore
├── everythingwehaverightnow.md    # Complete architecture inventory (reference)
├── docs/                           # Documentation directory
├── src-tauri/                      # Rust backend (Tauri)
└── ui/                             # Frontend (JavaScript runtime + UI)
```

## Documentation (`docs/`)

Contains all FRAME documentation:

- `README.md` - Documentation entry point
- `architecture_overview.md` - High-level system architecture
- `core_principles.md` - Foundational invariants
- `runtime_pipeline.md` - Intent execution flow
- `kernel_runtime.md` - Kernel implementation details
- `capability_system.md` - Capability model
- `dapp_model.md` - dApp structure
- `agent_system.md` - Agent system
- `capsule_system.md` - Capsule system
- And more...

## Rust Backend (`src-tauri/`)

The Tauri backend provides trusted system operations.

```
src-tauri/
├── build.rs                        # Build script
├── Cargo.toml                      # Rust dependencies
├── Cargo.lock                      # Dependency lock file
├── tauri.conf.json                 # Tauri configuration
├── capabilities/
│   └── default.json                # Default capability definitions
├── icons/
│   └── icon.png                    # Application icon
├── gen/schemas/                    # Generated Tauri schemas
│   ├── acl-manifests.json
│   ├── capabilities.json
│   ├── desktop-schema.json
│   └── linux-schema.json
└── src/
    ├── main.rs                     # Tauri entry point
    ├── lib.rs                      # Command registration hub
    ├── identity.rs                 # Multi-identity vault system
    ├── storage.rs                  # Encrypted per-identity storage
    ├── wallet.rs                   # frame token balance & payments
    ├── contacts.rs                 # Contact management
    ├── messages.rs                 # Messaging system
    ├── crypto.rs                   # Ed25519 + AES-GCM primitives
    └── dapps.rs                    # dApp directory listing
```

### Key Backend Modules

- **`identity.rs`** - Multi-identity vault with Ed25519 keypairs, encrypted at rest
- **`storage.rs`** - Per-identity encrypted key-value storage
- **`wallet.rs`** - frame token operations (balance, send)
- **`crypto.rs`** - Cryptographic primitives (key generation, signing, encryption)

## Frontend (`ui/`)

The JavaScript runtime and UI layer.

```
ui/
├── index.html                      # Shell HTML — boot sequence, script loading
├── app.js                          # Shell controller — command bar, widget grid, boot
├── style.css                       # Full UI stylesheet
├── info.txt                        # Info file
├── SPEC.md                         # UI specification
├── THREAT_MODEL.md                 # Security threat model
├── runtime/                        # Core runtime modules
├── system/                         # System services
├── ai/                             # AI routing layer
├── dapps/                          # Sandboxed dApps
├── layouts/                        # Predefined layout configurations
├── themes/                         # Theme definitions
├── assets/                         # Static assets
├── dev/                            # Developer tooling
└── verifier/                       # Standalone chain verification tool
```

## Runtime Modules (`ui/runtime/`)

Core deterministic execution engine.

```
runtime/
├── canonical.js                    # Deterministic serialization & SHA-256
├── storage.js                      # Canonical JSON storage wrapper
├── identity.js                     # Tauri identity API wrapper
├── stateRoot.js                    # Deterministic state root computation
├── backup.js                       # Identity export/import
├── engine.js                       # KERNEL — handleIntent pipeline, receipts, chain
├── router.js                       # dApp resolution, module loading, previews
├── replay.js                       # Deterministic chain replay
├── federation.js                   # Federated state synchronization
├── contracts.js                    # Contract runtime (proposals, signing, escrow)
├── snapshot.js                     # State snapshots
└── invariants.js                   # Boot-time invariant checks
```

### Key Runtime Files

- **`engine.js`** - The kernel. Handles intents, enforces capabilities, builds receipts
- **`router.js`** - Resolves intents to dApps, loads modules, executes dApps
- **`stateRoot.js`** - Computes deterministic state root hash
- **`canonical.js`** - Canonical JSON serialization and SHA-256 hashing
- **`storage.js`** - Storage wrapper with canonical validation

## System Services (`ui/system/`)

System-level services and managers.

```
system/
├── conversationEngine.js           # User text → intent → execute → response
├── capabilityRegistry.js           # Capability scanning, indexing, scoring
├── capabilityIndex.js              # Capability search index
├── capabilityPermissions.js        # Identity-scoped capability permissions
├── capabilityTable.js              # Provider pricing table
├── capabilityReputation.js         # Global capability reputation tracking
├── capabilityAdvertiser.js         # Announce capabilities to peers
├── widgetManager.js                # Widget CRUD, rendering, data sources
├── widgetRegistry.js               # Widget type definitions
├── layoutManager.js                # Layout persistence & region management
├── layoutEngine.js                 # DOM-level layout application
├── themeManager.js                 # Theme loading and application
├── uiPlanExecutor.js               # Execute UI plans (spawn widgets, set layouts)
├── uiStateController.js            # UI mode management
├── uiFeedback.js                   # Visual feedback system
├── agentPlanner.js                 # Execution graph planner
├── agentEngine.js                  # Autonomous agent loop
├── agentRegistry.js                # Agent CRUD and lifecycle
├── agentRuntime.js                 # Agent execution environment
├── agentEvolution.js                # Agent self-improvement
├── agentPersistence.js             # Agent state persistence
├── agentReputation.js              # Agent trust scoring
├── contextEngine.js                # Situational context collection
├── contextMemory.js                # Rolling context memory
├── contextAnalyzer.js              # Context sufficiency analysis
├── contextArchiveClient.js         # Remote context archive query
├── frameAPI.js                     # Public FRAME API surface
├── activityFeed.js                 # Activity log & chat rendering
├── workspaceManager.js             # Panel overlay system
├── workflowMemory.js               # Workflow success rate tracking
├── federatedMemory.js              # Cross-peer memory sharing
├── capsule.js                      # Capsule data structure
├── capsuleStore.js                 # Capsule persistence
├── capsuleExecutor.js              # Capsule execution
├── capsuleNetwork.js               # Network capsule routing
├── capsuleScheduler.js            # Capsule scheduling
├── capsuleTemplates.js             # Predefined capsule templates
├── intentCapsule.js                # Intent-to-capsule conversion
├── intentMarket.js                 # Distributed intent marketplace
├── walletManager.js                # Multi-chain wallet management
├── creditLedger.js                 # frame credit balance tracking
├── computeWorker.js                # Compute task execution
├── aiBuilder.js                    # AI-powered dApp generator
├── aiCommandRouter.js              # Natural language command parsing
├── aiRuntime.js                    # AI model runtime
├── dappInstaller.js                # dApp installation & persistence
├── p2pTransport.js                 # WebRTC-based P2P transport
├── peerAuth.js                     # Peer authentication
├── peerDiscovery.js                # Peer discovery protocol
├── peerRouter.js                   # Peer message routing
├── lanDiscovery.js                 # LAN peer discovery
├── bootstrapNodes.js               # Bootstrap node configuration
├── relayConfig.js                  # Relay server configuration
├── networkAutoConfig.js            # Automatic network configuration
├── networkSimulator.js             # Network simulation for demos
├── networkViz.js                   # Network topology visualization
├── nodeDeployment.js               # Remote node deployment
├── distributedStorage.js           # Distributed storage protocol
├── agents/                         # Built-in agents
│   ├── aiAssistantAgent.js
│   ├── capabilityDiscoveryAgent.js
│   ├── contextAnalyzerAgent.js
│   └── systemHealthAgent.js
├── agentTemplates/
│   └── index.js
└── tokenAdapters/
    ├── ethereumAdapter.js
    ├── solanaAdapter.js
    └── bitcoinAdapter.js
```

## AI Layer (`ui/ai/`)

AI routing and orchestration.

```
ai/
├── intentRouter.js                 # AI-powered intent classification
└── orchestrator.js                 # Multi-step AI orchestration
```

## dApps (`ui/dapps/`)

Sandboxed applications. Each dApp has `manifest.json` and `index.js`.

```
dapps/
├── ai/                             # Local AI (intent parsing, chat, summarize)
├── bridge/                         # External asset bridge (mint/burn frame)
├── compute/                        # Distributed compute (render, transcode)
├── contacts/                       # Contact management
├── context/                        # Context query
├── contextArchive/                # Context archive
├── contracts/                      # Smart contracts & ride-sharing
├── layout/                         # UI layout application
├── maps/                           # Place resolution & routing
├── messages/                       # Messaging
├── notes/                          # Note-taking
├── systemOverview/                 # System dashboard
├── theme/                          # Theme application
├── timer/                          # Countdown timer
└── wallet/                         # frame wallet (balance, send, mint, burn, escrow)
```

### dApp Structure

Each dApp directory contains:

```
dapps/<id>/
├── manifest.json                   # dApp declaration (intents, capabilities)
└── index.js                        # dApp implementation (exports run function)
```

**Example manifest.json:**
```json
{
  "name": "Wallet",
  "id": "wallet",
  "intents": ["wallet.balance", "wallet.send"],
  "capabilities": ["wallet.read", "wallet.send", "storage.read"],
  "capabilitySchemas": [...]
}
```

## Layouts (`ui/layouts/`)

Predefined layout configurations (JSON).

```
layouts/
├── dashboard.json                  # Default dashboard layout
├── desktop.json                    # Full desktop layout
├── developer.json                  # Developer-focused layout
└── minimal.json                    # Minimal layout
```

## Themes (`ui/themes/`)

Theme definitions (JSON).

```
themes/
├── dark.json                       # Dark theme
├── light.json                      # Light theme
├── aurora.json                     # Aurora theme
└── matrix.json                     # Matrix theme
```

## Assets (`ui/assets/`)

Static assets.

```
assets/
└── fonts/
    └── TX-03-Regular 03-Regular.ttf
```

## Developer Tools (`ui/dev/`)

Developer tooling.

```
dev/
└── create-dapp-template.js         # dApp template generator
```

## Verifier (`ui/verifier/`)

Standalone chain verification tool.

```
verifier/
├── frame-verifier.html             # Standalone verification page
└── frame-verifier.js               # Verification logic
```

## Directory Purposes Summary

| Directory | Purpose |
|-----------|---------|
| `src-tauri/` | Rust backend — Tauri commands for identity, storage, wallet, contacts, messages, crypto |
| `ui/runtime/` | Core runtime — kernel, router, canonical hashing, state root, replay, federation |
| `ui/system/` | System services — widgets, layout, agents, capsules, networking, context, AI |
| `ui/dapps/` | Sandboxed applications — each has `manifest.json` + `index.js` |
| `ui/ai/` | AI routing layer — intent classification, orchestration |
| `ui/layouts/` | Predefined layout configurations (JSON) |
| `ui/themes/` | Theme definitions (JSON) |
| `ui/verifier/` | Standalone chain verification tool |
| `ui/dev/` | Developer tooling (dApp template generator) |
| `docs/` | Architecture documentation |

## Script Loading Order

The `index.html` file loads scripts in a specific order:

1. **Runtime core** - canonical, storage, identity, stateRoot, engine, router
2. **System services** - capability registry, widget manager, layout manager, etc.
3. **AI layer** - intent router, orchestrator
4. **dApps** - Loaded dynamically via router
5. **UI shell** - app.js

This order ensures dependencies are available when modules initialize.

## Related Documentation

- [Architecture Overview](architecture_overview.md) - System layers and boundaries
- [Runtime Pipeline](runtime_pipeline.md) - How modules interact
- [dApp Model](dapp_model.md) - dApp structure and manifest format
