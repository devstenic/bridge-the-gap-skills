---
name: hybrid-navigation-bridge
description: Navigation bridge for a Flutter app that is half old, half new. Use when mounting a go_router module inside a GetX or Navigator 1.0 app (embed) or adding it to a root GoRouter (compose), when deciding who creates and disposes a module's router, when re-entering a module shows stale screens, dead buttons or a duplicate GlobalKey error, or when two routers write the browser URL.
license: MIT. Copyright (c) 2026 Mateusz Rus. See LICENSE.
metadata:
  author: Mateusz Rus
  source: bridge-the-gap-skills
  version: "1.0.0"
---

# Hybrid navigation bridge

One user journey can start in old code, run through a new module and end in old code again. This skill builds the bridge: the old app shows the module, the module brings its own routes, and every visit gets a fresh module and a fresh router.

## Words

- **Host**: the app that shows the module. Today often a GetX or Navigator 1.0 app.
- **Module**: a feature package with private routes, state and data behind one public facade.
- **Seam**: a place where the journey crosses between host and module: an entry, an exit, a Back.
- **Embed**: the old host shows the module on one page; the module's own GoRouter runs inside it.
- **Compose**: the new host has one root GoRouter; the module adds its routes to it.

## Steps

1. **Map the journey.** List every seam: where the user enters the module, every way out (outcomes such as order placed, cancelled, sign-in required) and every Back. Done when each seam has one owner: the module owns transitions inside its routes, the host owns every destination outside them.
2. **Choose the mount mode per host.** Embed while the shell is the old navigator; compose once the shell is a root GoRouter. Done when the choice is written down and the embed has a delete condition, for example "remove when the shell runs on go_router". Read [embed-and-compose.md](references/embed-and-compose.md).
3. **Give the module one route subtree.** The module exposes its routes and their guard as one object; inner paths stay private. Embed and compose mount the same subtree. Nest the steps so `go` builds a real back stack. Done when the host mounts the module without knowing any inner path.
4. **Embed: own the lifetimes.** Write `ModuleHost` and `EmbeddedRouter` once, in a shared routing package. The host page has no State of its own: `ModuleHost` creates the module on entry and disposes it on leave; `EmbeddedRouter` creates the router with the page and disposes it with the page. One router per mounted module, shared by all its steps. Done when a test shows that leaving disposes the module and re-entry starts at the first step.
5. **Make re-entry a decision.** Every entry is start, resume or focus, never a cache hit. Delete router caches keyed by location. Done when no code keeps a router alive between visits. Read [lifecycle-and-reentry.md](references/lifecycle-and-reentry.md).
6. **Decide who owns the URL.** Only the root router should write the browser URL. When embedded on the web, both routers write it; treat that as transitional. Real URLs, deep links and browser history come with compose. Done when the plan says which router owns the URL in each host and what cold entry to a later step does (default: restart the flow).
7. **Wire Back and exits.** Back goes through the host page to the module (back-navigation-chain skill). Exits go through the module's exit port (module-ports-and-adapters skill). Done when no module code calls the host's navigator.
8. **Prove it with real routers.** Tests for enter, step forward, system Back, leave, re-enter fresh, and cold entry to a later step. Done when they pass with the real GoRouter and the real old navigator; templates in [lifecycle-and-reentry.md](references/lifecycle-and-reentry.md).

## Rules

- One router per navigation tree. Only the root router writes the URL.
- The router is a disposable view of the journey. The journey's state lives in the module (a flow object, Cubits per screen).
- One GoRouter instance is mounted in one place at a time.
- A GetX app built with `GetMaterialApp(getPages: ...)` runs on the classic Navigator (1.0). GetX has its own Router mode, but classic `Get.toNamed` calls stop working there, so switching means changing every named-route call at once. Embedding avoids that big bang.

Verified with Flutter 3.44.3, go_router 16.3.0 and 18.0.2, get 4.7.3.
