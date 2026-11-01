# Bridge the Gap skills

AI agent skills for migrating a Flutter app step by step. They come from my Fluttercon 2026 talk *Bridging the Gap: Incremental Navigation & UI Migration in Flutter Apps*.

When a big app moves from an old architecture (GetX, Navigator 1.0) to a modular monorepo with go_router, old and new code live together for a long time. One user journey can start in old code, go through a new module and come back. These skills teach your coding agent to build and review that bridge.

## The skills

| Skill | Use it when |
|---|---|
| [`hybrid-navigation-bridge`](skills/hybrid-navigation-bridge/SKILL.md) | You show a go_router module inside a GetX or Navigator 1.0 app (embed) or add it to a root GoRouter (compose), or re-entering a module shows stale screens, dead buttons or duplicate GlobalKey errors |
| [`back-navigation-chain`](skills/back-navigation-chain/SKILL.md) | Back closes a whole flow or leaves the app, or you wire `PopScope` around a nested router |
| [`module-ports-and-adapters`](skills/module-ports-and-adapters/SKILL.md) | You extract a feature into a package with a handle, ports and host adapters, and keep GetX out of it |
| [`seam-accessibility`](skills/seam-accessibility/SKILL.md) | New screens run inside an old shell: page announcements, focus, Back, shared design tokens, accessibility tests |
| [`hybrid-migration-review`](skills/hybrid-migration-review/SKILL.md) | You review a pull request in a codebase where old and new run side by side |

## Install

For any agent that reads `SKILL.md` (Claude Code, Codex, GitHub Copilot, Cursor, Gemini CLI and others):

```bash
npx skills add mateuszrusdev/bridge-the-gap-skills
```

As a Claude Code plugin:

```bash
claude plugin marketplace add mateuszrusdev/bridge-the-gap-skills
claude plugin install bridge-the-gap-skills@bridge-the-gap
```

From a downloaded copy (a zip or a clone), point the same commands at the folder, from inside your Flutter project:

```bash
npx skills add /path/to/bridge-the-gap-skills
# or, for Claude Code:
claude plugin marketplace add /path/to/bridge-the-gap-skills
claude plugin install bridge-the-gap-skills@bridge-the-gap
```

By hand: copy the folders from `skills/` into your agent's skills folder, for example `.claude/skills/` (Claude Code), `.agents/skills/` (Codex, GitHub Copilot) or `.cursor/skills/` (Cursor).

## Try it

- "Embed the checkout module (go_router) in our GetX app. One router per visit, and Back must go step by step."
- "Back on the payment screen closes the whole checkout. Find out why and fix it."
- "Move the address flow from lib/app into packages/modules/address with ports and adapters. Keep GetX out of it."
- "Check accessibility where the new profile module runs inside the old shell, and add tests."
- "Review this pull request with the hybrid migration checklist."

## The ideas in one minute

- **Host, module, port, adapter.** Imports point in, calls go out: the host imports the module, the module never imports the host.
- **Embed today, compose later.** One router per navigation tree. Only the root router writes the URL.
- **The router lives and dies with its page.** The journey's state lives in the module.
- **Back is a chain.** The module decides first, the host goes last.
- **The module says what happened.** The host decides where to go.
- **Accessibility breaks at the seam.** Page names, focus, Back and results. One design system for both hosts.
- **Strangler fig.** The new code grows around the old until the old can go. Every adapter has a delete condition.

## Verified

The code in these skills compiles, and its tests pass (back chain, lifecycle, imports, accessibility) on Flutter 3.44.3 with go_router 16.3.0 and 18.0.2 and get 4.7.3. The platform facts were also checked in the Flutter 3.47.6 stable sources.

## Slides

[Bridging the Gap, Fluttercon 2026 (PDF)](slides/bridging-the-gap-fluttercon-2026.pdf)

## License

MIT, see [LICENSE](LICENSE). Each skill folder carries its own copy of the license and names the author in its frontmatter, so they travel with every installed skill.

## Author

Mateusz Rus. Questions or ideas: [message me on LinkedIn](https://www.linkedin.com/in/mateusz-rus-782b3b181).
