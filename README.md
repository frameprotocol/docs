# FRAME Documentation

Welcome to the FRAME documentation. This directory contains comprehensive documentation for the FRAME runtime system.

## What is FRAME?

FRAME is a **local-first, deterministic runtime** for applications and agents. It executes user intents through a capability-constrained execution engine, producing cryptographically signed receipts that enable verifiable state reconstruction and integrity verification.

Instead of traditional apps directly mutating state, FRAME processes intents through a deterministic kernel that:
- Resolves intents to dApps
- Executes dApps inside a capability-scoped API
- Records executions as cryptographically linked receipts
- Recomputes a deterministic state root representing the entire system state

All application behavior is verifiable, replayable, and deterministic.

## Documentation Structure

### Core Concepts
- **[Architecture Overview](architecture_overview.md)** - High-level system architecture and layers
- **[Core Principles](core_principles.md)** - Foundational invariants and design philosophy
- **[Directory Structure](directory_structure.md)** - Repository layout and file organization

### Runtime System
- **[Runtime Pipeline](runtime_pipeline.md)** - Intent execution flow from user input to receipt
- **[Kernel Runtime](kernel_runtime.md)** - Deterministic runtime core, protocol versions, integrity locks
- **[Deterministic Execution](deterministic_execution.md)** - Sandboxing rules, replay guarantees
- **[Router and Intents](router_and_intents.md)** - Intent resolution, manifest scanning, dApp dispatch

### Capability System
- **[Capability System](capability_system.md)** - Capability declaration, grants, and enforcement
- **[dApp Model](dapp_model.md)** - dApp structure, manifests, module execution

### State and Integrity
- **[Storage Model](storage_model.md)** - Canonical JSON storage and validation rules
- **[Receipt Chain](receipt_chain.md)** - Execution receipts, signing, chain linking
- **[State Root](state_root.md)** - Deterministic state root computation
- **[Replay and Verification](replay_and_verification.md)** - Deterministic replay and state validation
- **[Snapshots and Backups](snapshots_and_backups.md)** - State export/import and recovery
- **[Execution Proofs](execution_proofs.md)** - Verifiable execution proofs and trace recording

### Identity and Security
- **[Identity and Vaults](identity_and_vaults.md)** - Multi-identity system, Ed25519 keys, vault isolation
- **[Threat Model](threat_model.md)** - Security guarantees and trust assumptions

### Advanced Systems
- **[AI System](ai_system.md)** - Intent parsing, classification, UI planning
- **[Composite Execution](composite_execution.md)** - Multi-step intent execution, rollback behavior
- **[Agent System](agent_system.md)** - Agent registry, lifecycle, automation
- **[Capsule System](capsule_system.md)** - Portable execution units, workflow chaining
- **[UI System](ui_system.md)** - Widget-based UI, layout management, adaptive planning

### Network and Federation
- **[Networking Model](networking_model.md)** - Deterministic network requests, response sealing
- **[Federation and Sync](federation_and_sync.md)** - Receipt chain export, state synchronization
- **[Attestations](attestations.md)** - Cryptographic state proofs and cross-instance verification
- **[Distributed Execution](distributed_execution.md)** - Deterministic distributed execution (DDE) and compute tasks

### Financial Systems
- **[Wallet and Bridge](wallet_and_bridge.md)** - frame tokens, escrow, bridge mint/burn mechanics
- **[Contracts and Escrow](contracts_and_escrow.md)** - Contract lifecycle, proposals, signing

### Protocol Specification
- **[FRAME Protocol Spec](FRAME_PROTOCOL_SPEC.md)** - Single source of truth for protocol constants and schemas
- **[Protocol Versions](protocol_versions.md)** - Version constants and compatibility

### Verification and Testing
- **[Runtime Invariants](runtime_invariants.md)** - All invariants that must always hold
- **[Test Coverage](test_coverage.md)** - Test coverage mapping and missing tests
- **[Integration Flows](integration_flows.md)** - How modules connect and interact
- **[Failure Modes](failure_modes.md)** - All possible failures and recovery
- **[Protocol Compliance Checklist](protocol_compliance_checklist.md)** - Developer audit checklist
- **[Debugging Guide](debugging_guide.md)** - How to debug runtime issues
- **[Formal State Machine](formal_state_machine.md)** - Pure state machine correctness model
- **[Boot Sequence](boot_sequence.md)** - Exact startup order and verification

### Reference
- **[Glossary](glossary.md)** - Key terminology and definitions
- **[Documentation Plan](DOCUMENTATION_PLAN.md)** - Complete documentation roadmap

## Quick Start

1. Read [Architecture Overview](architecture_overview.md) for the big picture
2. Review [Core Principles](core_principles.md) to understand invariants
3. Follow [Runtime Pipeline](runtime_pipeline.md) to see how intents execute
4. Explore [Capability System](capability_system.md) to understand dApp permissions

## Protocol Versions

FRAME uses frozen protocol versions for compatibility:

- **PROTOCOL_VERSION**: `2.2.0` - Overall protocol version
- **RECEIPT_VERSION**: `2` - Receipt format version
- **CAPABILITY_VERSION**: `2` - Capability taxonomy version
- **STATE_ROOT_VERSION**: `3` - State root computation version

These versions define compatibility across FRAME nodes. See [Protocol Versions](protocol_versions.md) for details.

## Key Invariants

These principles must always hold:

1. **Deterministic Execution** - Same receipt chain always produces same state root
2. **Canonical Data Model** - All persisted state is canonical JSON
3. **Cryptographic Receipts** - Every state transition produces a signed receipt
4. **Capability-Based Isolation** - dApps only access declared capabilities
5. **Immutable Runtime Surfaces** - Kernel interfaces frozen after boot
6. **Verifiable State Root** - Hash over canonicalized system state
7. **Replay Verifiability** - State reconstructible from receipt chain
8. **Local Sovereignty** - All execution occurs locally

See [Core Principles](core_principles.md) for detailed explanations.
