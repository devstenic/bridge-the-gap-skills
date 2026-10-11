# Hybrid migration review checklist

Mark each item pass, finding, or not applicable. Each item: what to look for, and why it matters.

## 1. Ownership and imports

- **Imports point in.** No module file imports GetX, the host app or a sibling module; the module's pubspec has no such dependency. *Why:* one import turns the module back into part of the monolith.
- **Adapters live in the host.** Old code, GetX and module types meet only in the host's adapters folder. *Why:* one known place to delete when the old code goes.
- **Ports speak the module's language.** Port methods use module types, not legacy models or API clients. *Why:* a port in legacy words cannot be implemented by the new host.
- **Each adapter has a delete condition.** *Why:* without one, the bridge becomes the next legacy.
- **Required dependencies are required.** No `isRegistered` checks deciding behaviour; optional behaviour is an explicit mode. *Why:* container state silently changes the product.
- **An architecture test exists for a new module.** *Why:* the import rule holds only if a machine checks it.

## 2. Public surface

- **The facade is an allowlist.** The module exports its handle, ports and boundary types. No DI container, repositories, flow state or inner routes. *Why:* every extra export is a way around the boundary.
- **The host never writes module state.** Inputs go through `create` or a handle method. *Why:* hidden writes depend on build timing and break silently.

## 3. Lifecycle and re-entry

- **One router per navigation tree; one router per mounted module.** *Why:* two routers in one tree fight over Back and the URL.
- **The router is created with the page and disposed with it.** No cache of routers keyed by location. *Why:* cached routers cause dead screens on re-entry, duplicate GlobalKey errors and stale pages.
- **The module is created per visit and disposed on leave.** *Why:* state from a previous visit leaks into the next.
- **Re-entry is start, resume or focus, never a cache hit.** *Why:* the user gets an old, broken stack.
- **One journey is mounted once.** The same GoRouter is never shown in two places. *Why:* duplicate keys and two executors of one transition.
- **Late async results check that their visit is still current.** *Why:* a dead visit must not navigate through a new router.

## 4. Back

- **The steps form a real stack** (nested routes or push), so Back returns one step. *Why:* sibling routes with `go` leave one page, and Back closes the whole flow.
- **The host page uses `PopScope(canPop: false)` and calls the module's `requestBack()` before its own fallback.** *Why:* the module decides first, the host goes last.
- **The host never inspects module paths to guess what Back means.** *Why:* path matching breaks with the next route rename.
- **The on-screen Back at the module root runs the same chain.** *Why:* a second Back implementation drifts.
- **A visible, labelled Back exists at the module root.** *Why:* with `canPop: false`, iOS has no swipe there.
- **In compose, the module root uses `onExit` with the same decision, and lets a finished journey leave.** *Why:* `onExit` also runs on `go` after completion.

## 5. Exits and results

- **The module reports outcomes, not routes** (order placed, cancelled, sign-in required). *Why:* the host owns where the user goes next.
- **The host maps every outcome.** Exhaustive `switch`, no wildcard leading to a terminal action. *Why:* an unknown value must not trigger "finish".
- **One result, one channel.** `pop(result)` to a live parent on the same stack, or a port across the boundary, never both. *Why:* two channels deliver twice or disagree.

## 6. URL and entry

- **Only the root router owns the URL.** An embedded router writing the URL is recorded as transitional. *Why:* two writers break browser Back and Forward.
- **Cold entry to a later step restarts or recovers.** Only listed public entry points accept direct entry. *Why:* a typed URL must not show a success screen without the journey.

## 7. Accessibility at the seam

- **Each module page is named once** for screen readers (`namesRoute` through the template); the old navigator observer is quiet for the host page. *Why:* the old observer cannot see pages inside the module's router.
- **Frame and keyboard changes keep the widget tree shape.** *Why:* a reshaped tree rebuilds the page and drops focus.
- **The host announces the outcome on the next screen.** *Why:* the module is gone after the exit.
- **No copied design-system widgets without an owner and a removal condition.** *Why:* fixes to the shared widget never reach the copy.
- **Duplicated semantics are absent.** One semantics node per control. *Why:* screen readers announce two buttons.

## 8. Tests

- **Real-router tests cover the wiring:** step back, Back at the root, leave and re-enter, cold entry. *Why:* unit tests of a guard do not prove where it is attached or when it runs.
- **Guard decisions have table tests.** *Why:* fast, exhaustive cover of the decision itself.
- **Accessibility tests run the new screen inside the old shell at 2x text** with `meetsGuideline`, plus a focus test for keyboard changes. *Why:* the seam is where old and new break each other.
- **A manual TalkBack and VoiceOver pass is planned before release.** *Why:* guideline checks do not cover reading order or speech.
