# Android Migration Roadmap

This roadmap translates the Android migration plan into staged execution.
Use it to decide what should happen next, what each phase must produce, and what should not be pulled forward too early.

Related documents:

- [ARCHITECTURE.md](/root/app/remodex_android/ARCHITECTURE.md)
- [Docs/android-migration-collaboration.md](/root/app/remodex_android/Docs/android-migration-collaboration.md)
- [Docs/android-migration-architecture.md](/root/app/remodex_android/Docs/android-migration-architecture.md)

## 1. Roadmap principles

The roadmap follows these rules:

- migrate only the mobile client
- preserve the bridge, relay, and runtime contract
- lock protocol and recovery semantics before chasing visual fidelity
- keep implementation simple at first: single app module, package boundaries, minimal abstractions
- use Codex as the implementation default and Claude only for high-value bundled planning or audits

## 2. Phase overview

| Phase | Goal | Primary owner | Optional support | Exit condition |
|---|---|---|---|---|
| Phase 0 | freeze parity and implementation boundaries | Claude Opus | Codex | parity contract and Android package plan are stable |
| Phase 1 | establish Android app shell and safe local foundations | Codex | Claude Sonnet if needed | app boots and core navigation shells exist |
| Phase 2 | build thread, turn, and composer feature shells | Codex | Claude Sonnet | fake-data feature flow works end to end |
| Phase 3 | connect to the real bridge and preserve recovery semantics | Codex | Claude Opus audit, Sonnet review | real bridge flow works with reconnect and stop recovery |
| Phase 4 | add repo workflows and close high-value parity gaps | Codex | Sonnet or Opus selectively | major user journeys match accepted parity targets |
| Phase 5 | polish, test hardening, and optional integrations | Codex | Sonnet selectively | remaining gaps are explicit and intentionally accepted |

## 3. Phase 0: Boundary Freeze

### Goal

Freeze the Android migration contract before implementation spreads.

### Work in this phase

- confirm Android migration only rebuilds the mobile client
- freeze the list of behaviors that must remain 1:1
- freeze the list of behaviors that may become Android-native
- confirm the initial Android package structure
- define accepted out-of-scope items for the first runnable Android version
- confirm `bridge_copilot` only gets a thin adaptation boundary for preferences, permissions, and capabilities

### Deliverables

- frozen parity checklist
- Android package map
- first-pass test and acceptance matrix
- explicit non-goals list

### Do not do yet

- no protocol implementation
- no broad UI polish
- no generic multi-backend architecture
- no heavy module split

### Exit criteria

- contributors can answer what must stay 1:1
- contributors can answer what Android may implement natively
- Phase 1 and Phase 2 tasks are clear without reopening architecture debates

## 4. Phase 1: Android App Foundations

### Goal

Create a runnable Android shell with safe local primitives and the minimum navigation structure.

### Work in this phase

- create the Android app module
- establish package boundaries described in the Android architecture doc
- add app container or equivalent lightweight dependency wiring
- add redacted logging
- add DataStore and Android Keystore-based local persistence primitives
- implement onboarding shell
- implement QR scanning entrypoint
- implement empty sidebar, thread, turn, and settings shells
- establish navigation between the main surfaces

### Deliverables

- Android app project boots successfully
- onboarding to scan to thread-shell path exists
- local stores for pairing and trust metadata exist
- logging rules already avoid leaking live session identifiers

### Do not do yet

- no full secure transport implementation
- no broad repo action support
- no complicated offline history system
- no heavy DI, Room, or multi-module split

### Exit criteria

- app can launch and move through the main empty routes
- local foundations are in place for pairing and future protocol work
- no architecture debt has been introduced for speed

## 5. Phase 2: Core Feature Shells

### Goal

Build stable Android feature shells for threads, timeline, and composer before real protocol wiring.

### Work in this phase

- build thread list and project grouping shells
- build sidebar controls for open, create, archive, and selection flows
- build turn timeline shell with fake streamed content
- build composer shell with send, stop, attachment affordances, and queued draft affordances
- shape UI state, intents, and event flow in ViewModels
- add fake data, previews, and targeted unit tests

### Deliverables

- fake-data user flow that covers main navigation and conversation surfaces
- stable screen contracts for thread, turn, and composer flows
- first-pass UI state definitions aligned with the frozen parity contract

### Do not do yet

- no partial secure transport hacks
- no broad protocol assumptions embedded in UI widgets
- no pixel-perfect parity polishing

### Exit criteria

- thread, turn, and composer flows are understandable and testable
- state ownership is clear enough for real protocol wiring
- bundle boundaries are stable enough for Sonnet review if needed

## 6. Phase 3: Protocol And Recovery Integration

### Goal

Connect Android to the real Remodex bridge and preserve the behaviors that make the product trustworthy.

### Work in this phase

- implement pairing payload parsing
- implement trusted resolve requests
- implement WebSocket connection and reconnect flow
- implement secure handshake and encrypted payload flow
- implement request or response correlation and capability updates
- implement warm initialize compatibility handling
- implement active turn recovery
- implement stop fallback when `turnId` is missing
- implement queued follow-up drafts through active runs
- implement approval, access mode, and capability gating

### Deliverables

- Android can connect to a real bridge
- reconnect and foreground recovery preserve user control
- timeline and composer react to real runtime events
- stop remains usable under degraded or recovered state

### Do not do yet

- no low-value polish detours while recovery remains unstable
- no protocol shortcuts that drift from bridge semantics

### Exit criteria

- trusted reconnect works
- active turn state can be recovered
- queued drafts behave correctly
- recovery behavior passes the accepted parity checklist

## 7. Phase 4: Repo Workflow And Parity Closure

### Goal

Expand Android from conversation control into the broader repo and project workflow without breaking the core transport model.

### Work in this phase

- add git action surfaces and typed integrations
- add worktree and handoff flows
- add project scope and repo context polish
- add update prompts and compatibility UX
- resolve the highest-value parity gaps found in audits

### Deliverables

- repo workflows exist for the accepted Android MVP scope
- project-scoped work remains visible and understandable
- major high-value parity gaps are resolved or explicitly accepted

### Do not do yet

- no feature creep into generic backend orchestration
- no broad redesign of core mobile state ownership

### Exit criteria

- Android supports the major host-side workflows expected from Remodex mobile usage
- remaining differences are known, not accidental

## 8. Phase 5: Hardening And Optional Integrations

### Goal

Polish and harden the Android app once core parity is credible.

### Work in this phase

- add targeted notification flows
- add photo and voice support based on priority
- harden tests around reconnect, timeline merge, and action routing
- improve performance or rendering rough edges discovered during real use
- add thin `bridge_copilot` adapters only if they still fit the approved Android boundaries

### Deliverables

- audited parity gap list
- hardened tests for the highest-risk flows
- optional integrations that do not distort the core architecture

### Exit criteria

- remaining work is clearly polish, not latent core behavior debt
- Android is usable without hidden architectural drift

## 9. Recommended implementation order inside phases

Default execution order:

1. parity freeze
2. Android app shell
3. local stores and logger
4. onboarding and QR scan entry
5. thread and sidebar shell
6. turn and composer shell
7. secure transport and trusted resolve
8. reconnect and active turn recovery
9. stop fallback and queued drafts
10. capability and approval flows
11. git and worktree actions
12. polish and optional integrations

## 10. Phase gate checklist

Before moving to the next phase, confirm:

- current phase exit criteria are satisfied
- no hosted-service assumptions were added
- no broad architecture debate is still unresolved
- remaining deviations are documented and intentionally accepted
- the next phase can proceed without reopening already frozen decisions
