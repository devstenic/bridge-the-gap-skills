---
name: hybrid-migration-review
description: Review a pull request, branch or diff in a Flutter app where old and new architecture run side by side (GetX or Navigator 1.0 next to go_router modules). Use when asked to review migration work, a new module, a host adapter, routing or Back changes, or accessibility where old and new code meet. Read-only; reports blockers and findings with file and line evidence.
license: MIT. Copyright (c) 2026 Mateusz Rus. See LICENSE.
metadata:
  author: Mateusz Rus
  source: bridge-the-gap-skills
  version: "1.0.0"
---

# Hybrid migration review

A read-only review for code that crosses the seam between an old app and new modules. Judge the contract first, the code second.

## Steps

1. **Name the change.** Which seams does it touch (entry, exit, Back, frame, data)? Which mount mode (embed or compose)? Who owns each destination and each back stack? Done when you can state the transition contract in one sentence per touched seam. When you cannot (no owner, two owners, an unknown outcome), stop the detailed review and report a blocker to start.
2. **Read the diff against the checklist.** Apply every item of [checklist.md](references/checklist.md), in its order: ownership and imports, public surface, lifecycle and re-entry, Back, exits and results, URL, accessibility at the seam, tests. Done when every item is marked pass, finding, or not applicable, with a reason.
3. **Classify each finding.** Blocker to start (the contract is missing), blocking (the contract is broken), should fix, note. Each finding carries `file:line`, what breaks for the user or the next developer, and the smallest fix. Done when every blocking finding has evidence from the diff.
4. **Report.** Verdict first (merge, merge after fixes, or do not merge), then findings by severity, then what you did not check and why. Suggest code in the report; leave the files unchanged.

## Rules

- Start every review from the question: which transition does this change implement, from which state, on which trigger, and which router owner may decide the destination and the resulting stack? `go('/x')` is a mechanism, not a contract.
- Many observers are fine; one executor per transition.
- A failing happy path is rare. Look at the second entry, the double tap, the late async result and the rebuilt tree.
- Pre-existing debt that the change does not widen is a note with an owner. A new or widened break of navigation, accessibility or the boundary is blocking.

For fixes, point to the matching skill: hybrid-navigation-bridge, back-navigation-chain, module-ports-and-adapters or seam-accessibility.
