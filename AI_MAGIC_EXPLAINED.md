# FRAME AI Magic Explained 🪄

## Yes, the AI Works!

FRAME has **real AI** running locally in your browser. Here's how it works:

### 1. **Local LLM Model** 🤖

**Model:** `Xenova/LaMini-Flan-T5-248M` (248M parameters, ~100MB)

**How it loads:**
- Uses Transformers.js (loads from CDN: `@xenova/transformers`)
- Downloads model weights on first use
- Caches in browser IndexedDB
- Runs entirely in your browser (no API calls!)

**Location:** `ui/dapps/ai/modelLoader.js`

```javascript
// Model loads automatically when you use AI features
await loader.loadModel(); // Downloads & caches model
var text = await loader.generate('Parse this intent: send 5 frame to alice');
```

### 2. **Intent Parsing** 🎯

**Two-stage approach:**

1. **Rule-based parsing** (fast, always works)
   - Regex patterns for common intents
   - High confidence (0.80-0.95)
   - Examples: "send 5 frame to alice" → `wallet.send`

2. **AI model parsing** (smart, fallback)
   - Uses local LLM to parse complex intents
   - Lower confidence (0.65)
   - Handles variations and edge cases

**Location:** `ui/dapps/ai/inference.js`

```javascript
// Try rules first, then AI model
var result = await parseIntent('send 5 frame to alice');
// Returns: { intent: { action: 'wallet.send', payload: {...} }, confidence: 0.95, source: 'rules' }
```

### 3. **UI Planner Magic** ✨

**What makes widgets appear automatically:**

When you say something like "show me what's happening", here's what happens:

1. **Intent Resolution** → Parses your text into an intent
2. **Context Gathering** → Collects recent activity, active widgets, system metrics
3. **Widget Scoring** → Scores available widgets by relevance (0-1)
4. **UI Plan Generation** → Creates a plan with top 5 widgets
5. **Widget Creation** → Dynamically creates and renders widgets
6. **Reactive Updates** → Widgets subscribe to events and auto-update

**Location:** `ui/dapps/ai/uiPlanner.js` + `ui/system/conversationEngine.js`

```javascript
// Magic happens here:
var intent = await resolveIntent('show me what's happening');
var context = getContext(); // Recent activity, widgets, metrics
var plan = planUI(intent, context, capabilities, widgets);
// Returns: { type: 'ui.plan', layout: 'dashboard', widgets: [...] }
await executeUIPlan(plan); // Widgets appear!
```

### 4. **Reactive Event System** ⚡

**What makes widgets feel alive:**

- **Event Bus** → Central event system (`FRAME_EVENT_BUS`)
- **Widget Subscriptions** → Widgets subscribe to relevant events
- **Auto-refresh** → Widgets update when events fire
- **Activity Stream** → Shows all events in real-time

**Example:**
```javascript
// Widget subscribes to wallet events
eventBus.subscribe('wallet.balance.updated', () => {
  refreshWidget('wallet_balance'); // Auto-updates!
});

// When you send frame, event fires:
eventBus.emit('wallet.balance.updated', { balance: 100 });
// Widget automatically refreshes!
```

### 5. **Conversation Memory** 🧠

**Tracks context for natural follow-ups:**

- **Last Intent** → Remembers what you just did
- **Last Entities** → Remembers targets, amounts, etc.
- **Active Workflow** → Tracks multi-step processes
- **Referenced Widgets** → Knows which widgets you're looking at

**Example:**
```
You: "send 5 frame to alice"
FRAME: [sends, tracks: lastIntent='wallet.send', lastEntities=[{type:'target',value:'alice'}]]

You: "send 10 more"
FRAME: [knows you mean alice, sends 10 more frame]
```

## The Complete Magic Flow 🎭

### Example: "Show me what's happening"

```
1. User types: "show me what's happening"
   ↓
2. AI parses intent: { action: 'system.overview', payload: {} }
   ↓
3. UI Planner gathers context:
   - Recent activity: [wallet.send, receipt.appended, ...]
   - Active widgets: [wallet_balance, network_status]
   - System metrics: { network: 'active', receipts: 42 }
   ↓
4. Widget scoring:
   - system_status: 0.95 (high activity)
   - wallet_balance: 0.88 (recent sends)
   - network_activity: 0.82 (network active)
   - recent_tasks: 0.75 (recent activity)
   ↓
5. UI Plan generated:
   {
     type: 'ui.plan',
     layout: 'dashboard',
     widgets: [
       { widget: 'system_status', position: 'workspace' },
       { widget: 'wallet_balance', position: 'workspace' },
       { widget: 'network_activity', position: 'workspace' }
     ]
   }
   ↓
6. Widgets created & rendered:
   - system_status widget appears
   - wallet_balance widget appears
   - network_activity widget appears
   ↓
7. Widgets subscribe to events:
   - system_status → subscribes to ['receipt.appended', 'stateRoot.updated']
   - wallet_balance → subscribes to ['wallet.balance.updated']
   - network_activity → subscribes to ['network.peer.connected']
   ↓
8. When events fire → widgets auto-refresh!
```

## What Makes It Feel "Magic"? ✨

1. **Natural Language** → You just type what you want
2. **Intent Understanding** → AI parses your meaning
3. **Context Awareness** → Knows what's relevant right now
4. **Dynamic UI** → Widgets appear automatically
5. **Reactive Updates** → Everything updates in real-time
6. **Conversation Memory** → Remembers what you said
7. **Intent Preview** → Shows dangerous actions before executing
8. **Activity Stream** → See everything happening live

## Testing the AI 🧪

### Test 1: Intent Parsing
```javascript
// In browser console:
var result = await window.FRAME.handleIntent('ai.parse_intent', { 
  text: 'send 5 frame to alice' 
});
console.log(result);
// Should return: { intent: { action: 'wallet.send', payload: { amount: 5, target: 'alice' } } }
```

### Test 2: AI Chat
```javascript
var result = await window.FRAME.handleIntent('ai.chat', { 
  message: 'hello', text: 'hello' 
});
console.log(result.response);
// Should return AI-generated response
```

### Test 3: UI Planning
```javascript
// Type in FRAME: "show me what's happening"
// Watch widgets appear automatically!
```

### Test 4: Reactive Updates
```javascript
// Create a widget, then:
window.FRAME_EVENT_BUS.emit('wallet.balance.updated', { balance: 100 });
// Widget should auto-refresh!
```

## Current Status ✅

- ✅ **Local LLM** - Working (Transformers.js model)
- ✅ **Intent Parsing** - Working (rules + AI fallback)
- ✅ **UI Planning** - Working (context-aware widget generation)
- ✅ **Reactive System** - Working (event bus + subscriptions)
- ✅ **Conversation Memory** - Working (tracks context)
- ✅ **Intent Preview** - Working (shows dangerous actions)
- ✅ **Activity Stream** - Working (real-time events)

## The Magic is Real! 🎉

FRAME combines:
- **Local AI** (runs in browser, no API calls)
- **Intent-driven UI** (widgets appear based on what you say)
- **Reactive updates** (everything updates automatically)
- **Context awareness** (knows what's relevant)

**It's not just rule-based** - it uses a real AI model for parsing complex intents and generating responses. The "magic" comes from the seamless integration of AI + reactive UI + context awareness.

Try it: Type "show me what's happening" and watch widgets appear! 🪄
