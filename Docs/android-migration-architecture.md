# Android Migration Architecture

This document defines the target Android architecture for Remodex.
It is the implementation-facing companion to the repository-level [ARCHITECTURE.md](/root/app/remodex_android/ARCHITECTURE.md).

Use it when deciding:

- how the Android app should be structured
- where Android code should live
- which tech stack choices are preferred
- how UI, state, protocol, and host integration should be separated

Related documents:

- [ARCHITECTURE.md](/root/app/remodex_android/ARCHITECTURE.md)
- [Docs/android-migration-roadmap.md](/root/app/remodex_android/Docs/android-migration-roadmap.md)
- [Docs/android-migration-collaboration.md](/root/app/remodex_android/Docs/android-migration-collaboration.md)

## 1. Architecture goals

The Android architecture should:

- preserve local-first behavior and host-side execution boundaries
- match protocol and recovery semantics before chasing visual fidelity
- stay simple enough for one engineer and Codex to move quickly
- keep Android-specific framework details at the edges
- reserve only a thin future integration seam for `bridge_copilot`

The Android architecture should not:

- become a generic client platform
- introduce multiple runtime backends from day one
- overfit to ACP or MCP schema details
- start with heavy module splitting or framework-heavy state abstractions

## 2. Recommended technology stack

### Core language and UI

- Kotlin
- Jetpack Compose
- Navigation Compose

### State and async

- Coroutines
- `StateFlow`
- `ViewModel`

### Protocol and networking

- OkHttp WebSocket
- `kotlinx.serialization`

### Local persistence and security

- DataStore for non-secret local preferences and trusted metadata indexing
- Android Keystore for secret material
- encrypted file storage for history or sensitive local caches

### Device integrations

- CameraX and ML Kit for QR scanning
- system photo picker
- Android notification APIs
- voice capture only after core parity is stable

### Explicitly deferred by default

- Room
- heavy MVI frameworks
- large DI frameworks
- multi-module Gradle split
- Kotlin Multiplatform

## 3. Initial project shape

Start with a single Android app module and package-level boundaries.

Suggested shape:

```text
app/
  ui/
    onboarding/
    sidebar/
    thread/
    turn/
    settings/
    components/
    navigation/

  state/
    app/
    connection/
    threadlist/
    turn/
    settings/

  runtime/
    client/
    session/
    conversation/
    capabilities/

  protocol/
    rpc/
    pairing/
    secure/
    models/

  data/
    pairing/
    trust/
    history/
    preferences/

  support/
    logging/
    photos/
    voice/
    testing/

  integration/
    bridgecopilot/
```

The `integration/bridgecopilot` package should stay empty or minimal until the base Android app already has stable parity.

## 4. Layer responsibilities

### UI layer

Responsible for:

- rendering screens
- emitting user intents
- handling Android presentation details

Must not own:

- raw WebSocket logic
- secure handshake logic
- request correlation logic
- trusted reconnect decision rules

### State layer

Responsible for:

- turning protocol and storage signals into screen state
- managing connection status, thread lists, turn state, composer state, and banners
- managing queued drafts and recovery-facing product state

Must not own:

- low-level encrypted transport mechanics
- Android view rendering details

### Protocol layer

Responsible for:

- pairing QR payload parsing
- JSON-RPC request and response shapes
- secure handshake envelopes and crypto-facing models
- relay close code mapping
- request correlation and capability payload mapping

Must not know:

- Compose widgets
- navigation routes
- screen-specific UI state

### Runtime client layer

Responsible for:

- exposing typed operations to the rest of the app
- wrapping protocol details behind Android-facing APIs
- orchestrating connect, disconnect, request, subscribe, and recovery hooks

Must not become:

- a giant UI state owner
- a generic backend plug-in framework

### Data layer

Responsible for:

- saved pairing and trust metadata
- secure key material access
- local preferences
- optional encrypted history caching

Must not own:

- mainline protocol interpretation
- screen orchestration

## 5. Recommended key components

### `RemodexRuntimeClient`

Android-facing facade for:

- `connect()`
- `disconnect()`
- `observeEvents()`
- `listThreads()`
- `readThread()`
- `sendPrompt()`
- `stopTurn()`
- `sendQueuedDraft()`
- git and worktree request entrypoints

This should be the typed bridge-facing API, not the source of all UI state.

### `SecureSessionManager`

Owns:

- trusted pairing metadata loading
- secure handshake orchestration
- session key lifecycle
- reconnect-safe secure state

This is a security boundary and should stay separate from UI state.

### `ConnectionController`

Owns:

- launch auto-connect
- manual reconnect
- foreground reconnect
- re-pair escalation
- mapping transport failures to product recovery states

This should mirror product behavior from iOS, not just socket lifecycle.

### `ConversationStore`

Owns:

- thread lists
- turn timeline snapshots
- active turn tracking
- queued drafts
- merge rules for incoming events

This is the likely Android-side source of truth for conversation-derived state.

### `CapabilityStore`

Owns:

- bridge and runtime capability flags
- feature gating inputs for UI and request shaping

Capabilities should be centralized, not reinterpreted independently in each screen.

## 6. State model guidance

Recommended top-level Android state slices:

- `ConnectionUiState`
- `ThreadListUiState`
- `TurnUiState`
- `ComposerUiState`
- `SettingsUiState`
- `CapabilityState`

Recommended product-level state concerns:

- trusted pairing status
- reconnect and recovery status
- active turn metadata
- queued drafts and run pause or resume behavior
- project scope and repo context
- approval prompts and access mode

The Android app should preserve the semantic distinction between:

- source-of-truth protocol and secure session data
- derived UI state shown by screens

## 7. Protocol and recovery rules

The Android implementation must preserve these behaviors:

- trusted resolve after the initial QR bootstrap
- secure encrypted session after handshake success
- warm initialize compatibility for resumed runtime paths
- stop fallback when a direct active `turnId` is not available
- active turn visibility after reconnect or foreground recovery
- queued follow-up drafts during active runs
- timeline merge behavior that avoids fake duplicate rows or flattened reasoning artifacts

If these behaviors drift, the Android app is not yet at parity even if the screens look correct.

## 8. Storage and security strategy

### Use DataStore for

- relay URL metadata
- selected non-secret preferences
- trusted host identifiers and lookup metadata that are safe to persist outside the keystore

### Use Android Keystore for

- device identity or secret material
- encryption keys for secure local storage
- secrets that should not exist as plain values in app storage

### Use encrypted files for

- optional local timeline caches
- replay-safe local artifacts that need to survive process death

Avoid introducing Room until there is clear evidence that simple encrypted files and small indexed stores are no longer sufficient.

## 9. Logging and privacy rules

The Android app must:

- redact live `sessionId` values
- avoid logging bearer-like pairing material
- avoid logging decrypted sensitive payloads unless explicitly safe
- keep debug logging useful for lifecycle and recovery diagnosis without leaking secrets

Add a central redacted logger before broad protocol work begins.

## 10. `bridge_copilot` integration boundary

If Android later integrates with `bridge_copilot`, keep it thin.

Allowed responsibilities:

- session preference mapping
- permission prompt mapping
- capability mapping

Not allowed responsibilities:

- replacing the Android conversation model
- replacing Remodex-specific protocol and recovery semantics
- forcing raw ACP or MCP schema objects into the UI layer

## 11. Testing priorities

Highest priority tests:

- pairing payload parsing
- trusted resolve flow
- secure handshake and session resume
- reconnect state transitions
- active turn recovery
- stop fallback behavior
- queued draft resume behavior
- capability gating behavior

Lower priority until later:

- pixel-perfect visual parity
- broad optional integrations like voice or media polish

## 12. Architecture decision defaults

Use these defaults unless a future doc explicitly changes them:

- single app module first
- package boundaries before Gradle boundaries
- protocol parity before visual polish
- simple state model before framework-heavy abstractions
- Codex-first implementation, Claude-assisted planning or auditing only for large bundles
- thin integration seams instead of platform-first generalization
