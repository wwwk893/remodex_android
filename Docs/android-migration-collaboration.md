# Android Migration Collaboration Principles

This document defines the execution rules for the Android migration effort and the default division of labor between Codex, Claude Sonnet, and Claude Opus.

Use this document whenever the task is about:

- Android parity planning or implementation for Remodex
- deciding whether a task should go to Codex or Claude
- preparing a large `bridge_copilot` request for Sonnet or Opus
- reviewing parity gaps between iOS behavior and Android behavior

Related documents:

- [ARCHITECTURE.md](/root/app/remodex_android/ARCHITECTURE.md)
- [Docs/android-migration-roadmap.md](/root/app/remodex_android/Docs/android-migration-roadmap.md)
- [Docs/android-migration-architecture.md](/root/app/remodex_android/Docs/android-migration-architecture.md)

Keep the project local-first. Do not reintroduce hosted-service assumptions, hardcoded public relay defaults, or a remote-first product model.

## 1. Goal And Scope

The Android migration should:

- rebuild only the mobile client
- preserve the existing bridge, relay, and Codex runtime model
- keep local-first pairing, trusted reconnect, and project-scoped workflow semantics
- prioritize protocol parity and recovery parity before visual parity
- use Claude for high-value batch work and Codex for continuous implementation and bug fixing

The Android migration should not:

- move runtime execution onto Android
- redesign the product around hosted backends
- introduce a generic multi-backend client platform first
- let `bridge_copilot` or ACP/MCP schemas dictate the Android app architecture
- start with pixel-perfect UI parity before protocol and recovery behavior are stable

## 2. Non-Negotiable Parity Rules

These behaviors must remain 1:1 unless an explicit repo decision says otherwise:

- QR pairing and trusted reconnect semantics
- secure transport handshake and resume semantics
- warm runtime initialize compatibility
- active turn recovery after reconnect or foreground resume
- stop fallback when a usable `turnId` is missing
- item-scoped timeline merge and late reasoning merge behavior
- queued follow-up draft behavior during active runs
- capability-driven UI gating
- project and worktree scoped chat behavior
- local-first defaults with no hardcoded public relay assumptions
- redaction of live `sessionId` or bearer-like pairing identifiers in logs

These areas may use Android-native implementation details as long as behavior stays aligned:

- navigation layout and presentation style
- QR scanner implementation
- local persistence primitives
- notification integration
- photo and voice capture integration
- Android lifecycle-specific reconnect scheduling

## 3. Default Model Roles

### Codex

Codex is the default implementation engine. Use Codex for:

- Kotlin and Compose implementation
- protocol models, mappers, and typed bridge APIs
- WebSocket and secure transport implementation
- ViewModel and StateFlow wiring
- bug fixing, regression fixing, and parity polishing
- tests, fixtures, and local refactors
- any task that is already well-scoped and does not need a new architecture decision

### Claude Sonnet

Sonnet is the default batch planner and batch skeleton writer. Use Sonnet for:

- feature-bundle implementation cards
- grouped UI skeleton planning for a whole bundle
- bundle-level test and edge-case checklists
- parity review for a bounded Android feature slice

Sonnet should receive a narrow but substantial bundle, not random small tasks.

### Claude Opus

Opus is the expensive architecture and audit tool. Use Opus only for:

- initial parity contract freeze
- protocol and recovery deep review
- final parity audit when the Android build is mostly assembled
- rare cross-cutting decisions whose outcome affects multiple bundles and would otherwise cause broad rework

If a task does not clearly save multiple Sonnet rounds or multiple days of Codex rework, it should not go to Opus.

## 4. Claude Call Gatekeeping

Do not send a task to Claude when any of these is true:

- the task touches fewer than 4 meaningful files
- the task is a pure bug fix
- the task is mostly compilation, lint, import, resource, or small UI cleanup work
- the task is a trivial DTO, enum, mapper, or boilerplate generation task
- the task is a second or third local refinement after Claude already produced the core plan

Do send a task to Claude when all of these are true:

- the task is bundled around one coherent feature slice
- the boundary is stable enough to describe clearly
- the expected output can drive 1 to 3 days of Codex work
- the prompt can name concrete files, constraints, and acceptance criteria

Stop using Claude for a bundle and hand the work back to Codex when:

- the same bundle has already required 2 rounds of Claude corrections
- the remaining work is mostly implementation detail or bug fixing
- the output is repeating architecture advice instead of moving the codebase forward

## 5. Recommended Budget

Use the balanced plan by default.

### Conservative

- Opus: 1 call
- Sonnet: 4 calls
- Best for: strong self-direction and maximum call savings

### Balanced

- Opus: 2 calls
- Sonnet: 8 calls
- Best for: normal Android migration flow with low rework risk

### Sprint

- Opus: 2 to 3 calls
- Sonnet: 10 to 12 calls
- Best for: pushing a first runnable version quickly

For this repo, default to the balanced plan unless the user explicitly asks to optimize harder for speed or for minimum Claude usage.

## 6. Bundle Strategy

Prefer sending Claude one of these bundles instead of file-by-file tasks:

1. Connection bundle
   pairing, secure transport, trusted reconnect, re-pair escalation, relay close-code handling
2. Thread bundle
   sidebar, thread list, project grouping, open/create/archive flows
3. Turn bundle
   timeline, stream merge, active turn, stop fallback, queued drafts, capability gating
4. Repo bundle
   git actions, worktree actions, handoff, project scope switch behavior
5. Polish bundle
   settings, notifications, voice, photo, compatibility prompts, final parity audit

Do not ask Claude for broad alternative architectures. Ask for one recommended path plus explicit non-goals and acceptance criteria.

## 7. Standard Prompt Requirements

Every Claude request for this migration must include:

1. the exact goal of the bundle
2. the relevant file list only
3. frozen constraints
4. current Android status
5. expected output format
6. acceptance checklist
7. explicit out-of-scope items

Every Claude response is only considered usable if it includes:

1. assumptions
2. affected files or file groups
3. state flow or behavior flow
4. edge cases
5. acceptance checklist
6. explicit non-goals

## 8. Reusable Task Card Templates

### 8.1 Opus Boundary Freeze Card

```text
Task:
Freeze the Android parity contract for this bundle. Do not propose a platform rewrite.

Inputs:
- frozen repo constraints
- the specific source files listed below

Hard constraints:
- local-first
- do not rewrite bridge/relay/runtime
- no hosted-service assumptions
- no multi-backend platformization
- keep code simple and maintainable

Output:
1. behaviors that must stay 1:1
2. behaviors that may be Android-native
3. Android state machine or control flow
4. acceptance checklist
5. non-goals
6. open questions, maximum 3
```

### 8.2 Sonnet Bundle Planning Card

```text
Task:
Produce the Android implementation card for this bundle based on the frozen parity contract.

Inputs:
- frozen parity contract excerpt
- current Android file tree or diff
- relevant source files listed below

Output:
1. file list
2. responsibility of each file
3. state and event flow
4. edge cases
5. tests that must exist
6. changes that should be avoided
```

### 8.3 Sonnet Bundle Audit Card

```text
Task:
Audit the current Android implementation of this bundle against the frozen parity contract.

Inputs:
- frozen parity contract excerpt
- current Kotlin files or diff
- known issues if any

Output:
1. parity gaps
2. risky behavior mismatches
3. minimal fix order
4. tests still missing
5. acceptable deviations
```

### 8.4 Codex Implementation Card

```text
Task:
Implement the approved Android bundle directly in the repo.

Inputs:
- frozen parity contract excerpt
- approved bundle implementation card
- current Android source files

Execution rules:
- prefer minimal diffs
- avoid broad refactors
- implement behavior before visual polish
- stop using Claude for this bundle unless a new cross-cutting architecture issue appears
```

## 9. Execution Order

Use this order by default:

1. Opus freezes the initial parity contract
2. Codex creates the Android package skeleton and local stores
3. Codex builds onboarding, pairing entry, and thread or turn empty shells
4. Sonnet reviews the first major bundle only if boundaries are clear
5. Codex implements the bundle end to end
6. Opus reviews protocol and recovery behavior before late-stage polish
7. Sonnet or Codex performs final parity audit depending on remaining gap size

## 10. Android Migration Definition Of Done

A bundle is done when:

- its acceptance checklist is satisfied
- behavior matches the frozen parity contract for that slice
- no hosted-service assumptions were introduced
- the code remains understandable without framework-heavy indirection
- remaining differences are explicit and intentionally accepted
