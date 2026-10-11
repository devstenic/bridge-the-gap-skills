---
name: back-navigation-chain
description: Back handling across a Flutter module boundary. Use when Back closes a whole flow or leaves the app instead of going one step back, when wiring PopScope around a nested router, when the back arrow at a module's first screen must behave like system Back, or when checking iOS swipe-back, Android predictive back and browser Back for an embedded module.
license: MIT. Copyright (c) 2026 Mateusz Rus. See LICENSE.
metadata:
  author: Mateusz Rus
  source: bridge-the-gap-skills
  version: "1.0.0"
---

# Back navigation chain

"The inner router can't pop" does not mean "leave the module". Back is a chain: the inner router tries first, then the module decides, and the host goes last.

## Steps

1. **List every Back.** System button, back gesture, the back arrow on screen, browser Back, a Cancel button. Done when each one is mapped to the same chain, or to a written exception.
2. **Make the steps a real stack.** Nest the step routes (or push them) so popping returns one step. A flow that moves between sibling routes with `go` has a single page, so Back has nothing to pop and the host closes the whole flow. Done when a real-router test shows Back from step N lands on step N-1.
3. **Give the module a back policy.** `requestBack()` returns `handled` or `notHandled`. It pops an inner step when it can; at the first step it decides: cancel and report it through the exit port, keep the draft, ask the user, or answer `notHandled`. Done when every first-step case has a named result and no path string decides it.
4. **Forward from the host page.** The page that shows the module wraps it in `PopScope(canPop: false)`. `onPopInvokedWithResult` calls `requestBack()`; the host goes back only on `notHandled`. Done when the host page contains no module paths.
5. **Use the same chain on screen.** At the module's first screen there is no inner page to pop, so the template shows its own Back. It calls `Navigator.of(context, rootNavigator: true).maybePop()`, which runs the host page's `PopScope`. Done when a test taps it and lands where system Back lands.
6. **Check each platform.** iOS, Android and web each add one detail; read [platforms.md](references/platforms.md). Done when the UI keeps a visible Back at the module root and the plan names compose as the fix for web history.
7. **Test the chain.** Four real-router tests: step back, first step with a draft, first step with nothing to lose, on-screen Back. Templates in [back-chain.md](references/back-chain.md). Done when they pass.

## Rules

- A Back gesture may request a cancellation. The flow decides what cancellation does and reports one outcome; the host only picks the next destination.
- One contract serves the close button, system Back and a host-initiated cancel.
- `canPop: false` on a page hands Back to your code: iOS turns off the edge swipe there and Android shows no predictive-back animation there. Keep a visible, labelled Back.
- In a composed shell the module's first screen is a normal page of the root router. Put the same decision in the `onExit` of the module's subtree (return false to stay).

Verified with Flutter 3.44.3 (and the 3.47.6 stable sources), go_router 16.3.0 and 18.0.2, get 4.7.3.
