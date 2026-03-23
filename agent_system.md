# FRAME Agent System

Agents observe the system and automatically create capsules when work should be done. They do not bypass FRAME: all work flows through `FRAME_CAPSULE.createCapsule`, `FRAME_CAPSULE.signCapsule`, and `FRAME_CAPSULE_NETWORK.announceCapsule`, then the normal scheduler and execution pipeline.

---

## 1. Agent architecture

- **Layer:** Above intent execution. Agents produce capsules; the capsule scheduler and executor handle placement and execution.
- **Dependencies:** Context memory, capability table, capsule store, identity, capsule API, capsule network.
- **No bypass:** Agents never call `FRAME.handleIntent` directly for their own tasks; they create signed capsules and announce them.

---

## 2. Agent lifecycle

Each agent is a simple object with three methods:

1. **observe()** — Returns a snapshot of relevant system state (e.g. rolling context, capability table, capsule queue).
2. **decide(observation)** — Returns an action object, or `null` if no action should be taken.
3. **act(action)** — Creates a capsule from the action, signs it, and announces it. Returns a Promise (resolves to `{ announced, capsuleId }` or similar).

Flow: **observe → decide → act**. The engine runs this loop for every registered agent on a fixed interval (every 5 seconds). If `decide` returns non-null, the engine calls `act(decision)` and respects a per-agent cooldown (30 seconds) before allowing another `act` from the same agent.

---

## 3. Agent engine

**File:** `ui/system/agentEngine.js`

- **runAgents()** — Loads agents from `FRAME_AGENT_REGISTRY.getAgents()`. For each agent: if cooldown elapsed, call `observe()`, then `decide(observation)`. If decision is non-null, set `agent.lastRun = now`, call `act(decision)` (and optionally log resolution for capsule id).
- **start()** — Starts the loop: run once immediately, then every 5 seconds (`LOOP_INTERVAL_MS`).
- **stop()** — Clears the interval.
- **registerAgent(agent)** — Delegates to `FRAME_AGENT_REGISTRY.register(agent)`.
- **getActivityLog()** — Returns recent entries: `{ type: 'decision'|'capsule'|'error', agentId, decision?, capsuleId?, message?, timestamp }` (used by the Agent Activity widget).

**Cooldown:** Minimum 30 seconds (`COOLDOWN_MS`) between `act()` calls per agent; enforced by checking `agent.lastRun` before calling `act`.

---

## 4. Agent registry

**File:** `ui/system/agentRegistry.js`

- **register(agent)** — Requires `agent.id` (string) and `observe`, `decide`, `act` (functions). Replaces existing agent with same id, or appends. Returns boolean.
- **getAgents()** — Returns a copy of the registered agents array.

Built-in agents register themselves when their script runs (e.g. `FRAME_AGENT_REGISTRY.register({ id: '...', observe, decide, act })`).

---

## 5. Capsule generation

Agents build capsules with:

- **FRAME_CAPSULE.createCapsule({ ... })** — Fields: `originIdentity` (from `FRAME_IDENTITY.getCurrent()`), `requestedCapability`, `payload` (e.g. `{ action, payload }`), `paymentRule: { type: 'credit', amount: 1 }`, `privacyRule: 'public'`, `executionPolicy: { maxRetries: 2, timeout: 10000 }`.
- **FRAME_CAPSULE.signCapsule(capsule)** — Sets `capsule.id` and `capsule.signature`; returns Promise.
- **FRAME_CAPSULE_NETWORK.announceCapsule(signedCapsule)** — Sends the capsule through the scheduler (local or single peer) or broadcast. Returns Promise.

So **act(decision)** is async: get current identity → createCapsule → signCapsule → announceCapsule, and return the resulting Promise. The engine does not await `act` in a way that blocks the loop; it only attaches a `.then` to log capsule id when the announcement resolves.

---

## 6. Safety model

- **Cooldown:** 30 seconds between `act()` per agent to avoid duplicate capsules in quick succession.
- **lastRun:** Each agent may have `agent.lastRun` set by the engine after each `act()`; the engine skips `act()` if `(now - agent.lastRun) < COOLDOWN_MS`.
- **No direct execution:** Agents do not invoke the intent pipeline directly; all execution goes through capsules and the normal scheduler/executor.
- **Activity log:** Decisions and capsule creations (and errors) are logged for the Agent Activity widget and debugging.

---

## 7. Built-in agents

All live under `ui/system/agents/`.

| Agent | File | Purpose |
|-------|------|---------|
| Context Analyzer | `contextAnalyzerAgent.js` | observe: `FRAME_CONTEXT_MEMORY.getRollingContext()`. decide: activity spike (≥8 events in window) or many similar actions (≥3 same action) → `{ type: 'analyze_context', payload: { events } }`. act: create capsule `requestedCapability: 'ai.analyze'`, payload `{ action: 'ai.analyze', events }`, announce. |
| Capability Discovery | `capabilityDiscoveryAgent.js` | observe: `FRAME_CAPABILITY_TABLE.getAllProviders()` keys. decide: if capability set changed (new or disappeared) → `{ type: 'update_capability_map' }`. act: create capsule `requestedCapability: 'network.capability_map'`, announce. |
| System Health | `systemHealthAgent.js` | observe: capsule queue (pending/announced/executing count), connections, peers. decide: if queue ≥ 5 → `{ type: 'load_distribution' }`. act: create capsule `requestedCapability: 'compute.redistribute'`, announce. |
| AI Assistant | `aiAssistantAgent.js` | observe: recent rolling context. decide: repeated commands (same action ≥2) or intent-like queries (≥2) → `{ type: 'assist_user' }`. act: create capsule `requestedCapability: 'ai.assist'`, announce. |

---

## 8. Extension mechanism

- **Register a new agent:** Implement an object with `id`, `observe`, `decide`, `act`, and call `FRAME_AGENT_REGISTRY.register(agent)` (e.g. from a script loaded after the registry and engine). The engine will pick it up on the next run.
- **Script order:** In `index.html`, load `agentRegistry.js`, then `agentEngine.js`, then each agent script so that agents register before `FRAME_AGENT_ENGINE.start()` is called from `app.js`.

---

## 9. Startup

**app.js:** After the capability advertiser is started, the boot sequence calls `FRAME_AGENT_ENGINE.start()` so the agent loop runs (every 5 seconds) for the rest of the session.

---

## 10. Widget: Agent Activity

- **Data source:** `agent.activity`. Uses `FRAME_AGENT_ENGINE.getActivityLog()`; each entry shown as agent id (label) and type/capsule id/decision/error (value). Last 20 entries.
- **Default widget:** `default_agent_activity`, title "Agent Activity", data source `agent.activity`, refresh 5s, priority 52, region sidebar.

---

## 11. File summary

| File | Purpose |
|------|---------|
| `ui/system/agentRegistry.js` | register(agent), getAgents(). |
| `ui/system/agentEngine.js` | start/stop, runAgents (observe → decide → act), cooldown, activity log. |
| `ui/system/agents/contextAnalyzerAgent.js` | Context memory patterns → ai.analyze capsule. |
| `ui/system/agents/capabilityDiscoveryAgent.js` | Capability table changes → network.capability_map capsule. |
| `ui/system/agents/systemHealthAgent.js` | Capsule backlog → compute.redistribute capsule. |
| `ui/system/agents/aiAssistantAgent.js` | Repeated/incomplete user activity → ai.assist capsule. |

Integration: **app.js** (start engine after boot), **widgetManager** (agent.activity data source, default_agent_activity widget).
