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
          │              │ ui/runtime/engine.js          │            │
          │              │ handleIntent                  │            │
          │              │ • kernelNormalize             │            │
          │              │ • normalizeViaAiDApp (→ AI)   │            │
          │              │ • validateIntent              │            │
          │              │ • handleBuiltIn (system.*)    │            │
          │              │ • router.resolve(intent)      │            │
          │              │ • permissions, integrity      │            │
          │              │ • router.execute(match,..)    │            │
          │              │ • buildReceipt, appendReceipt │            │
          │              └──────────────┬───────────────┘             │
          │                             │                             │
          ▼                             ▼                             ▼
┌──────────────────┐         ┌──────────────────────┐       ┌─────────────────────┐
│ FRAME_LAYOUT     │         │ ui/runtime/router.js │       │ FRAME_STORAGE        │
│ FRAME_WIDGETS    │         │ resolve: loadManifests│      │ (chain, permissions) │
│ FRAME_UI_STATE   │         │   list_dapps, match  │       │ FRAME_STATE_ROOT     │
└──────────────────┘         │ execute: import run()│       └─────────────────────┘
          │                  │   buildScopedApi    │
          │                  └──────────┬──────────┘
          │                             │
          │                             ▼
          │                  ┌──────────────────────┐
          │                  │ dApp run(intent, api)│
          │                  │ • capabilityGuard    │
          │                  │ • async blockers on  │
          │                  └──────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  CONTEXT & MEMORY                                                               │
│  FRAME_CONTEXT_ENGINE.getContext() → activeWidgets, installedDapps, recentActivity│
│  FRAME_CONTEXT_MEMORY → shortTerm (RAM), rolling (localStorage), archive pointer│
│  FRAME_CONTEXT_ARCHIVE_CLIENT.queryArchive → getBestCapability('context.archive'│
│  FRAME_ACTIVITY.pushEntry → addEvent, addRollingEvent                           │
└─────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│  NETWORK (coordination layer)                                                   │
│  FRAME_PEER_ROUTER (peer table, rings, routeIntent)                             │
│  FRAME_P2P (WebRTC, send/recv intent_route, intent_result)                      │
│  FRAME_PEER_DISCOVERY (bootstrap, announce)                                     │
│  FRAME_INTENT_MARKET (publish, bid, localStorage)                               │
└─────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│  TAURI (native)                                                                 │
│  list_dapps, storage_read, storage_write, list_identities, get_current_identity │
│  sign_data, get_identity_public_key, send_payment, get_balance, etc.            │
└─────────────────────────────────────────────────────────────────────────────────┘
