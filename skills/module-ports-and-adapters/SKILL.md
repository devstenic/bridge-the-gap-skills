---
name: module-ports-and-adapters
description: Ports and adapters for a Flutter feature module in an app that is being migrated. Use when extracting a feature from a legacy (for example GetX) app into a package, designing a module's handle and public facade, defining data, exit and frame ports, writing host adapters over legacy code, replacing navigation calls with outcome types, or adding a test that keeps GetX and app code out of a module.
license: MIT. Copyright (c) 2026 Mateusz Rus. See LICENSE.
metadata:
  author: Mateusz Rus
  source: bridge-the-gap-skills
  version: "1.0.0"
---

# Module ports and adapters

Hexagonal architecture applied to a UI module. One rule makes it work: imports point in, calls go out. The host imports the module; the module never imports the host. At runtime the module calls its ports, and the host's adapters run the old code.

## Words

- **Host**: the app that shows the module, today often the old GetX app.
- **Module**: the new feature package. Everything inside is private behind one facade.
- **Handle**: what the host may do with the module: create it, mount its routes, forward Back, dispose it.
- **Port**: an interface the module owns, in its own words: data it needs, how it exits, the frame it is shown in.
- **Adapter**: host code that implements a port with old code. The only place where old code and module types meet.

## Steps

1. **Pick one slice.** One journey with a clear entry and clear exits, for example checkout: Address, Payment, Review. Done when the slice, its entry and its outcomes are named.
2. **List what the module needs from outside.** Data it does not own yet, its exits, its frame (app bar, navigation chrome), infrastructure (flags, device info). Done when each need is classified as a data port, an exit port, a frame port, or a capability the module implements itself.
3. **Define the handle.** `create(...)` returns a fresh instance per visit; `routes` is its route subtree with its guard; `requestBack()`; `dispose()`. Done when the host needs nothing else to show, drive and remove the module.
4. **Define the ports in the module's words.** Small interfaces with module types only: no GetX, no direct calls to the old API, no old models. The exit port says what happened (order placed, cancelled, sign-in required), never where to go. The frame port takes builders. Done when the old host and a future new host could each implement every port without changing it. Code in [ports-and-adapters.md](references/ports-and-adapters.md).
5. **Write the adapters in the host.** One adapter per port, in the host's adapters folder. Each one maps old models at the edge and states its delete condition. Done when the module's port names appear in the host only in adapters and in the place that creates the module.
6. **Publish one facade.** The package's main library exports the handle, the ports and the types that cross the boundary. The DI container, repositories, flow state and inner routes stay private. Done when every export line has a named consumer.
7. **Guard the rule with a test.** An architecture test inside the module reads its files and fails on imports of GetX, the host app or sibling modules; a pubspec check catches the same dependencies. Done when the test runs in CI. Template in [architecture-test.md](references/architecture-test.md).
8. **Mount it.** Embed or compose the module with the hybrid-navigation-bridge skill; wire Back with the back-navigation-chain skill.

## Rules

- A port belongs to the module whose language it speaks, even when the host implements it.
- A port over legacy code is a strangler seam: it carries its target state and its delete condition. The bridge must shrink, or it becomes the next legacy.
- A capability that is the module's own domain and has no legacy implementation is a repository inside the module, not a host port.
- A missing required dependency fails when the module is created. An optional capability is an explicit mode in the API, never "whatever is registered in the container".
- Ports over legacy code shrink over time. Host callbacks for decisions only the host can make (where to go next, sign-out) stay.

Verified with Flutter 3.44.3, go_router 16.3.0 and 18.0.2, get 4.7.3.
