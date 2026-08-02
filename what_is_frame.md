  1. What is FRAME?

  FRAME is a universal runtime for structured systems. Everything it hosts is the same kind of thing: a FRAME — a node
  with identity, attributes, relationships, and declared capabilities. There is no second ontology underneath. People,
  devices, packages, policies, AI participants, and history records are all FRAMEs living in one Runtime Graph.

  It is not a normal OS (it does not own processes/filesystems as its world model). It is not a normal app platform
  (apps do not own state; they contribute into a shared graph). It is not “just a runtime” in the JVM/Node sense — the
  durable world is the graph, not the process. It is not a graph database — the graph is under governance: clients
  cannot write it directly; change goes through Intent → Governance → Mutation.

  Meaning is emergent. FRAME Core does not decide that something “is a car” or “is a document.” It stores FRAMEs and
  relationships; packages and Perspectives give those nodes domain meaning.

  ────────────────────────────────────────

  2. What happens when I run FRAME?

  frame boots ProductRuntime:

  frame
  → data directory (authority / packages / logs / tmp)
  → recover or first-boot kernel (graph + history)
  → owner identity + host-root FRAMEs (first boot)
  → package index rebuilt from the graph
  → optional AI config
  → HTTP server (session-authenticated)
  → client protocol (frame.client.v1)
  → optional Connected Host peers
  → ready

  Fresh first boot: empty storage → seed owner identity FRAME, host-root FRAME, default allow policy / intent handler →
  create durable kernel → issue owner SessionGrant → listen (default 127.0.0.1:8001).

  After shutdown / restart: same data dir → load snapshot + replay change log → same authoritative graph head → package
  cache rebuilt from graph → sessions must re-authenticate (session files are not durable authority of who you are
  forever without grants).

  ────────────────────────────────────────

  3. What is actually stored?

  Authoritative (must survive):
  • Runtime Graph (FRAMEs + relationships)
  • History / Changes (revisioned mutations)
  • Kernel snapshot + change log under authority/
  • Identity / owner binding on the graph
  • Package lifecycle state on the graph
  • Policies, evidence, receipts that were committed via governance
  • Peer sync grants (under authority)
  • Package artifacts that were admitted (under packages/)

  Derived / cache / runtime (rebuildable or ephemeral):
  • .frame/packages.json disk package index (cache of graph)
  • Projection / Attention / Watch / recommendations
  • Live client sessions, event queues
  • AI provider availability
  • HTTP in-flight state, tmp/

  Acknowledged durable Changes are meant to survive restart via snapshot + change-log recovery.

  ────────────────────────────────────────

  4. What can a FRAME represent?

  Anything you model as a node with attrs + relationships. A Person, Device, Project, Organization, AI, document,
  service, package install record, or authorization envelope is constructed as FRAMEs (usually by packages or Acts),
  not hardcoded into FRAME Core.

  FRAME provides the substrate. Domains are content of the graph, contributed by packages and users.

  ────────────────────────────────────────

  5. Relationships

  Edges between frame_uuids with typed relationship kinds (contains, references, owns, implements, …). Hierarchy is
  only one projection; the graph itself is not a single tree.

  Examples you can build:
  • Alice --owns→ Device
  • Org --contains→ Alice
  • Project --references→ Document

  Relationships are first-class graph truth once committed. Clients see them through projections, not by editing
  adjacency lists.

  ────────────────────────────────────────

  6. Capabilities and providers

  Capability: a named function the world may need (construct, publish, TemperatureSensor, …) — declared, not hardcoded
  into Core behavior.

  Provider: a FRAME (often from a package) that implements a capability.

  FRAME gains functionality by mounting packages that declare offers/requests; Core brokers resolution without knowing
  domain meaning.

  Example: Device needs notify. A package activates a provider FRAME that offers notify. Resolution binds
  Device→provider. Core never learns what “notify” means beyond the contract.

  ────────────────────────────────────────

  7. Packages / dApps

  A .frame package is an immutable creator→runtime artifact: manifest, contract (capabilities, profiles, permissions,
  …), optional assets/runtime bytes, digest + signature.

  Lifecycle (governed): discover/verify → install → activate (Intent) → contributes capabilities/behavior/projections
  while Active → suspend / deactivate → remove.

  Packages are how you build with FRAME: you ship definitions into the host; you do not fork Core for each product. In
  this repo, the shipped built-in package tree is essentially packages/interaction/authorization-envelope/.

  ────────────────────────────────────────

  8. Governance and authority

  Actual write path:

  Recommendation (optional)
  → Act (cross Commitment Boundary)
  → Intent
  → Governance (policies / strategies)
  → Mutation (GovernedMutation / HostAlgebra)
  → Change on graph + history
  → Receipt / Event
  → clients re-project

  Clients cannot patch the Runtime Graph. They hold a SessionGrant (authenticated session). Identity is on the graph /
  identity chain — not “trust the HTTP header.” Authorization FRAMEs / envelopes scope standing permission. Delegation
  is explicit and revocable. Policies are FRAMEs evaluated by the single governance engine.

  ────────────────────────────────────────

  9. Attention / interaction

  Attention (per client/session): inhabit the world without writing it.
  • Attend — center a locus
  • Enter / Ascend — navigate depth
  • Watch — notice changes at locus
  • Why — explanation over current locus
  • Suppose — private hypothetical branch
  • Real — leave Suppose
  • Recommendation — world proposes a possible Act
  • Act — attempt to commit

  Attention = understanding. Act = attempt to cross the Commitment Boundary into Intent→Governance→Mutation.

  Authorization envelopes package standing scope so later Acts stay bounded.

  ────────────────────────────────────────

  10. Projection / Perspective

  The graph has no single UI. A Perspective selects how to interpret the graph; a Projection is the view for that
  Perspective + Attention.

  Same world → different projections on desktop, phone, shell, browser (or any future surface). Surfaces are clients of
  one authority, not separate databases.

  ────────────────────────────────────────

  11. The client system

  frame.client.v1 is the universal client contract: open session → ops (Attend/Act/…) → snapshot → events → close.

  A client needs: protocol + projection schema + SessionGrant.
  A client does not need HostAlgebra, raw mutation APIs, or filesystem authority.

  Browser (ui via clientSession), desktop, phone, shell, and remote HTTP clients share that contract. Attention is per 
  client; committed graph revision is shared.

  ────────────────────────────────────────

  12. AI

  AI is a participant, not a second authority. It observes projections, can Recommend / explain, and Act only with
  delegation like anyone else. Malformed AI output does not become graph truth without Act→Governance.

  Without AI config / provider, FRAME still runs; recommendations from AI simply are not there.

  ────────────────────────────────────────

  13. Connected Host (30 seconds)

  Two FRAME hosts authenticate with peer grants, then sync admitted scope. They remain separate authorities, not one
  mega-server. Sync is authorized; revoke stops sync. Disconnect → local work continues under local authority;
  reconnect converges admitted state again.

  ────────────────────────────────────────

  14. Offline behavior

  A host works fully offline on its own graph: Attend, Act, packages, persistence. When another host becomes reachable
  and peer authority allows it, admitted state can sync again. No network ≠ no FRAME.

  ────────────────────────────────────────

  15. Persistence and recovery

  • Snapshot — durable checkpoint of graph + history
  • Change log — append Changes before/around snapshot for crash windows
  • Restart — load snapshot, replay log past checkpoint
  • Backup / restore — capture/restore authoritative data dir layout

  Acknowledged durable Changes are intended to survive restart; caches rebuild.

  ────────────────────────────────────────

  16. Package security

  Artifact → content digest → signature against trust store → contract validation → governed install/activate. Bytes on
  disk are not authority. Unsigned/placeholder/wrong-signer/tampered packages fail closed on production paths.
  Activation only after governance admits lifecycle Intents onto the graph.

  ────────────────────────────────────────

  17. Execution

  Production executable packages use node as the sole supported exec mode. native / native_placeholder are rejected.
  There is one authoritative execution/admission path: package must be Active on the graph and have a real executor. No
  silent “Active but no executor.”

  ────────────────────────────────────────

  18. Exchange / assets

  Exchange Space remains a generic substrate convention: listings/reviews/audits as FRAMEs + capability discovery
  (exchange_space.rs). Not a product marketplace.

  Wallet / assets still exist as a thin governed vertical (wallet_asset.rs): wallet FRAMEs + governed transfer via
  HostAlgebra — generic asset/ledger-shaped construction, not a bank product.

  ────────────────────────────────────────

  19. What the UI actually is

  Implemented:
  • Browser UI (ui/) — React host client over frame.client.v1: artifacts, packages, import/export, activity, govern,
    history, AI sections
  • Desktop binary (frame-desktop, optional feature) projecting scenes over HTTP client session
  • Shell / phone interaction modules as projection vocabularies over the same client API
  • Wire TS runtime (runtimes/frame_wire_ts) for protocol conformance

  Possible but not a polished multi-domain product UI: rich domain experiences (cars, dashboards, etc.) — those would
  be packages + Perspectives built with FRAME, not shipped as FRAME Core.

  ────────────────────────────────────────

  20. What is NOT inside FRAME anymore?

  After reduction / boundary cleanup:
  • Gauge / OBD / vehicle diagnostics — not FRAME
  • Weather-station products — not FRAME
  • RISC-V / ISA demos — not FRAME
  • Ecosystem / distributed product packages — not FRAME
  • Marketplace product helpers — not FRAME (Exchange Space remains generic)
  • Object-era archive on HEAD — gone

  Those are things you build with FRAME in other repos.

  ────────────────────────────────────────

  21. One complete example

  1. Alice constructs Device (Act → Intent → Governance → Change).
  2. Package activates; offers notify on Device.
  3. AI participant observes; emits Recommendation.
  4. Alice Why → explanation from projection/history.
  5. Alice Act on recommendation → Commitment Boundary → Intent → Governance accepts → Mutation → Receipt.
  6. Graph revision advances; second client’s Watch/events → new Projection.

  Systems involved: interaction/Attention, client.v1, packages, capabilities, AI participant, governance, kernel
  persistence, client event fan-out.

  ────────────────────────────────────────

  22. What files/modules make FRAME?

  Simplified map at 737ef67:

  ┌───────────────────────────────────────────────┬───────────────────────────────────────────────────────────────────┐
  │ Area                                          │ Role                                                              │
  ├───────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
  │ frame_runtime_host/                           │ Production Rust host: graph, governance, packages, HTTP, product  │
  │                                               │ boot                                                              │
  ├───────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
  │ …/frame_package/                              │ FRAME ontology ops: relationships, packages, perspectives,        │
  │                                               │ exchange, physics                                                 │
  ├───────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
  │ …/interaction/                                │ Attention, Commitment Boundary, shell/phone/desktop client APIs   │
  ├───────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
  │ …/product_runtime.rs + product_boot.rs        │ Installable product host lifecycle                                │
  ├───────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
  │ …/client_protocol.rs + client_http.rs         │ frame.client.v1                                                   │
  ├───────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
  │ …/kernel/                                     │ Snapshot, change log, recovery                                    │
  ├───────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
  │ …/connected_host.rs + peer_sync_auth.rs       │ Multi-host sync with grants                                       │
  ├───────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
  │ …/ai_provider.rs + ai_participant.rs          │ Optional AI as participant                                        │
  ├───────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
  │ …/identity_chain.rs + session_grant.rs +      │ Identity and session authority                                    │
  │ http_auth.rs                                  │                                                                   │
  ├───────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
  │ packages/interaction/                         │ Built-in authorization-envelope package                           │
  ├───────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
  │ runtimes/frame_wire_ts/                       │ Wire/session TS implementation                                    │
  ├───────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
  │ ui/                                           │ Browser universal client                                          │
  ├───────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
  │ docs/                                         │ Canonical architecture spine                                      │
  ├───────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
  │ tools/                                        │ architecture-check, verify-canonical, package-release             │
  ├───────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
  │ dist/frame/                                   │ Release layout (frame binary + ui-dist)                           │
  ├───────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
  │ fixtures/                                     │ Wire/canonical vectors                                            │
  └───────────────────────────────────────────────┴───────────────────────────────────────────────────────────────────┘

  ────────────────────────────────────────

  23. What do I physically have now?

  • Binaries: frame (product entry), frame-runtime-host, optional frame-desktop
  • Runtime: ProductRuntime over LocalHost / kernel — sole graph authority
  • Clients: browser UI, desktop/shell/phone paths, remote HTTP — all frame.client.v1
  • Packages: contract + install/activate lifecycle; one built-in interaction package tree
  • Networking: HTTP host API, Connected Host peer sync with grants, wire session protocol
  • Persistence: authority snapshot + change log, backup/restore, package artifacts
  • Security: SessionGrants, identity chain, package signatures/trust, governance gate, peer auth
  • AI: optional configured providers as participants (stub/real), never raw authority
  • Builders get: a substrate to create worlds, packages, Perspectives, and clients without modifying Core

  ────────────────────────────────────────

  FRAME IN 30 SECONDS

  We built a governed relationship-graph runtime: everything is a FRAME; only Intent→Governance→Mutation writes the
  world; clients inhabit via Attention and commit via Act over frame.client.v1; packages add capabilities without
  forking Core; hosts can sync as separate authorities; state survives as snapshot + history. Domain products (cars,
  weather, marketplaces) are not in the repo anymore — they are what you build on this.
