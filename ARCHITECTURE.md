# Architecture

This document is the high-level architecture map for Remodex.
It is meant to help contributors answer:

- what the repository actually does
- where iOS app work lives
- where bridge and relay logic lives
- which boundaries are intentional and should not be collapsed
- how the Android migration fits into the current repository

## 1. Why this document exists

This repository is not a single mobile app.
It is a local-first system composed of:

- a mobile client
- a local bridge running on the user's machine
- a public or self-hosted relay
- the Codex runtime and local git or workspace actions that stay on the host machine

Understanding that separation is more important than understanding any single screen or file.

Use this file when you need to:

- decide where a change belongs
- keep local-first constraints intact
- avoid mixing UI work with bridge or relay responsibilities
- onboard into the repo before working on Android migration

## 2. Bird's-Eye View

### 2.1 Problem the project solves

Remodex lets a phone act as a secure remote control for a Codex session that still runs on the user's machine.

It is responsible for:

- pairing a phone with a trusted host bridge
- forwarding encrypted mobile requests and responses through a relay
- presenting thread, turn, composer, and repo workflows on mobile
- keeping Codex, git, and workspace actions on the host side

Non-goals:

- running Codex directly on the phone
- turning the relay into an application server
- making the repository depend on hosted-service defaults

### 2.2 High-level system model

At the highest level, the system works like this:

1. the host bridge starts and creates a live session
2. the mobile client pairs or resolves that trusted session through the relay
3. the mobile client and bridge establish a secure encrypted session
4. the mobile client sends user actions and renders streamed state
5. the bridge routes runtime, git, workspace, and desktop actions locally

### 2.3 Ground state vs derived state

Ground state / source of truth:

- bridge-side live session and trusted pairing state
- bridge-managed runtime state and local rollout data
- relay session availability and trusted resolve metadata
- mobile-side saved trust metadata and secure local caches

Derived state:

- mobile timeline render state
- grouped thread lists, badges, banners, and composer accessory state
- reconnect UI state and readiness indicators
- audit or parity documents for Android migration

Rule of thumb:

- protocol, trust, and host runtime state are the source of truth
- mobile UI state is derived and should be recomputable from protocol events and local caches

### 2.4 Typical request flow

Phone user action  
→ mobile UI and state container  
→ mobile transport and secure-session layer  
→ relay transport  
→ host bridge  
→ local Codex runtime, git handler, workspace handler, or desktop handler  
→ streamed response back through bridge and relay  
→ mobile timeline and composer state

## 3. Code Map

| Path / Module | Responsibility | Owns | Depends on |
|---|---|---|---|
| `CodexMobile/CodexMobile/Views` | SwiftUI screens and user interaction surfaces | onboarding, sidebar, turn, composer, settings UI | `CodexService`, view models, models |
| `CodexMobile/CodexMobile/Services` | iOS client state, transport, pairing, sync, notifications, compatibility logic | connection lifecycle, secure transport orchestration, thread and turn state | Foundation, Network, mobile models, bridge protocol |
| `CodexMobile/CodexMobile/Models` | shared iOS-side data contracts and UI-facing model types | threads, messages, runtime options, git models, RPC structures | Foundation only |
| `phodex-bridge/src` | host-side bridge runtime | session lifecycle, Codex transport, git and workspace handlers, desktop handoff, rollout mirroring | Node runtime, `ws`, local Codex process |
| `relay/` | transport hop and optional push endpoints | relay session routing, trusted resolve, optional push registration and completion | Node HTTP and WebSocket primitives |
| `Docs/` | stable project guidance | self-hosting, migration collaboration, Android planning docs | repository facts and conventions |

### Important modules and symbols

- `CodexService` — the iOS state and protocol hub; the main source of truth on the mobile side
- `ContentViewModel` — top-level connection and reconnect orchestration for the iOS root flow
- `TurnView` and `TurnComposerView` — the main user-facing interaction shell for conversation work
- `startBridge` in `phodex-bridge/src/bridge.js` — the bridge entrypoint that ties relay, runtime, and host handlers together
- `createRelayServer` in `relay/server.js` — the relay HTTP and WebSocket server entrypoint

## 4. API Boundaries and Layering

### Public or stable boundaries

| Boundary | Consumers | What crosses the boundary | What must not cross |
|---|---|---|---|
| Mobile UI ↔ mobile state | SwiftUI views and view models | view state, intents, user actions, display-safe models | raw relay socket details, encryption primitives |
| Mobile state ↔ protocol or transport | `CodexService` and transport helpers | typed RPC payloads, connection state, capability flags | SwiftUI view concerns |
| Mobile ↔ relay | mobile client, host bridge | WebSocket frames, trusted resolve requests, secure control messages, encrypted payloads | plaintext application payloads after secure session establishment |
| Bridge ↔ local runtime | bridge runtime and Codex process | JSON-RPC messages, runtime lifecycle commands | mobile UI details |
| Bridge ↔ local git or workspace | bridge handlers | typed host-side commands and results | relay routing concerns |

### Dependency rules

Allowed:

- mobile views → mobile services and view models
- mobile services → models and transport helpers
- bridge entrypoint → bridge handlers and transport
- relay server → relay transport and optional push service

Not allowed:

- mobile UI directly owning relay or secure transport protocol state
- relay executing runtime, git, or workspace logic
- bridge handlers depending on SwiftUI or mobile presentation logic
- hosted-service defaults leaking back into source-controlled project behavior

## 5. Architectural Invariants

- The host machine remains the execution boundary for Codex, git, and workspace actions.
- The relay is a transport hop, not an application brain.
- The mobile client is rich in state and interaction, but it does not become the runtime host.
- Pairing and trusted reconnect semantics are architectural, not cosmetic; they must survive UI rewrites.
- Project-scoped and worktree-scoped behavior is part of product semantics, not a sidebar presentation detail.
- Compatibility gating through runtime or bridge capabilities belongs in the mobile state layer, not scattered ad hoc through UI code.

### Negative-space invariants

- There is no legitimate architecture where the mobile app becomes the place that runs Codex jobs.
- There is no legitimate architecture where relay code gains plaintext application authority after secure session setup.
- There is no legitimate architecture where Android migration introduces public hosted defaults into committed source.
- There is no reason for UI-only changes to rewrite bridge or relay responsibilities.

## 6. Cross-Cutting Concerns

### State management

- On iOS, `CodexService` is the central state owner for connection, threads, turns, capabilities, and compatibility flags.
- View shells such as `SidebarView`, `TurnView`, and `TurnComposerView` render derived state and dispatch intents.
- Android migration should preserve the same separation, even if the concrete implementation uses `StateFlow` and Compose instead of SwiftUI and Observation.

### Error handling

- Connection and reconnect failures are translated into user-facing recovery states rather than raw socket errors.
- Saved pairing state is only cleared for failures that prove trust can no longer be reused.
- Protocol and capability mismatches should degrade via compatibility prompts or gated UI, not silent behavior drift.

### Async, reconnect, and recovery

- Reconnect and foreground recovery are first-class behavior, not optional polish.
- Active turn recovery, queued drafts, and stop fallback exist to preserve user control across transient disconnects.
- The bridge remains alive across transient relay disconnects and treats reconnect as a normal path.

### Security and privacy

- Session identifiers and pairing-like secrets should be redacted from logs.
- Trusted pairing state is persisted locally and treated as sensitive.
- Application payloads are encrypted end to end after secure session establishment.

### Testing strategy

- The highest-risk boundaries are connection lifecycle, trusted resolve, secure handshake, timeline merge behavior, and host-side command routing.
- Android migration should test protocol parity and recovery semantics before spending time on visual polish.

## 7. Android migration fit

The Android app is an additional mobile client for the same architecture, not a replacement architecture.

Its job is to:

- reproduce the mobile interaction surface
- preserve protocol, reconnect, and parity semantics
- talk to the same bridge and relay model

Its job is not to:

- change the meaning of the bridge or relay
- introduce a multi-backend abstraction layer first
- make `bridge_copilot` the center of the mobile app architecture

See:

- [Docs/android-migration-collaboration.md](/root/app/remodex_android/Docs/android-migration-collaboration.md)
- [Docs/android-migration-roadmap.md](/root/app/remodex_android/Docs/android-migration-roadmap.md)
- [Docs/android-migration-architecture.md](/root/app/remodex_android/Docs/android-migration-architecture.md)

## 8. Reading guide for newcomers

Recommended reading order:

1. [README.md](/root/app/remodex_android/README.md)
2. this file
3. [CodexService.swift](/root/app/remodex_android/CodexMobile/CodexMobile/Services/CodexService.swift)
4. [CodexService+Connection.swift](/root/app/remodex_android/CodexMobile/CodexMobile/Services/CodexService+Connection.swift)
5. [bridge.js](/root/app/remodex_android/phodex-bridge/src/bridge.js)
6. [relay/README.md](/root/app/remodex_android/relay/README.md)
7. Android migration docs in `Docs/`

## 9. Assumptions / unknowns

- The Android app does not yet exist in this repository, so Android-specific package boundaries are a design target rather than a current code fact.
- This document focuses on stable repository structure, not every feature-specific iOS implementation detail.

## 10. Evidence used

- [README.md](/root/app/remodex_android/README.md)
- [AGENTS.md](/root/app/remodex_android/AGENTS.md)
- [CLAUDE.md](/root/app/remodex_android/CLAUDE.md)
- [CodexService.swift](/root/app/remodex_android/CodexMobile/CodexMobile/Services/CodexService.swift)
- [CodexService+Connection.swift](/root/app/remodex_android/CodexMobile/CodexMobile/Services/CodexService+Connection.swift)
- [ContentViewModel.swift](/root/app/remodex_android/CodexMobile/CodexMobile/Views/Home/ContentViewModel.swift)
- [TurnView.swift](/root/app/remodex_android/CodexMobile/CodexMobile/Views/Turn/TurnView.swift)
- [TurnComposerView.swift](/root/app/remodex_android/CodexMobile/CodexMobile/Views/Turn/TurnComposerView.swift)
- [bridge.js](/root/app/remodex_android/phodex-bridge/src/bridge.js)
- [server.js](/root/app/remodex_android/relay/server.js)
- [relay/README.md](/root/app/remodex_android/relay/README.md)
