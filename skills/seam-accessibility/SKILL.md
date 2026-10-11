---
name: seam-accessibility
description: Accessibility and design-system checks where old and new Flutter code meet. Use when new module screens run inside an old app shell, when a shared frame, navigation bar or keyboard handling changes, when screen readers miss module pages or results, when focus is lost after the keyboard opens, when a module copies design-system widgets, or when writing accessibility tests for migrated screens.
license: MIT. Copyright (c) 2026 Mateusz Rus. See LICENSE.
metadata:
  author: Mateusz Rus
  source: bridge-the-gap-skills
  version: "1.0.0"
---

# Seam accessibility

Most accessibility bugs are normal bugs. Mixing old and new code adds a few of its own, and they all sit at the seam: the places where a journey crosses from the old app into a module and back. Walk the journey across the seam and check four moments.

## Steps

1. **Walk the journey with a screen reader and a keyboard.** Old screen, module, old screen again. At each crossing check four moments: open a screen, type, go back, get the result. Done when each moment has a pass or a written finding.
2. **Open a screen: the page is named once.** An old app often announces pages from a navigator observer; that observer cannot see pages inside the module's own router. Name each module page through the template with `Semantics(namesRoute: true)`; on the web add one polite announcement. Keep the old observer quiet for the page that only hosts the module, so the user hears one name. Done when TalkBack and VoiceOver say each module page's name once.
3. **Type: focus survives frame and keyboard changes.** A shared frame or padding widget that changes its widget tree when the keyboard opens rebuilds the page and drops focus. Keep the tree shape stable and change values, not structure. Done when the focus test passes: the same `EditableTextState`, still focused, after the keyboard opens.
4. **Go back: Back is visible and reachable.** At the module root `canPop` is false, so iOS has no swipe there. The template shows a labelled Back that runs the back chain. Done when the root Back is announced, focusable with a keyboard, and meets the tap-target guidelines at 2x text.
5. **Get the result: the host announces it.** After the exit the module is gone and the host owns the next screen, so the host announces the outcome and the next screen starts with its title. Done when the outcome is heard once.
6. **One design system for both hosts.** Tokens as a `ThemeExtension` in one theme that the old and the new app both use. Templates own the safe area, keyboard, scrolling, the main button and Back; screens own only their content. A copy of a design-system widget inside a module gets an owner and a removal date, because fixes to the shared widget never reach the copy. Done when the module takes its widgets from the design system, or each copy is marked.
7. **Test at the seam.** Put the new screen inside the old shell with the old theme at 2x text and run the guideline checks; check page names in the semantics tree; run the focus test. Before a release, do a real pass with TalkBack and VoiceOver. Templates in [a11y-tests.md](references/a11y-tests.md). Done when the tests run in CI.

## Rules

- Prefer implicit announcements (named routes, live regions). Send an explicit `SemanticsService.sendAnnouncement` only where the platform needs it and `MediaQuery.supportsAnnounceOf` is true; Android has deprecated announcement events.
- The screen owns arrangement and meaning (headings, landmarks, reading order). A component owns its own role, state and focus. One semantics node per control.
- Fix a shared widget once and every screen that uses it gets the fix.

Code for each moment: [seam-checks.md](references/seam-checks.md). Verified with Flutter 3.44.3, go_router 16.3.0 and 18.0.2, get 4.7.3.
