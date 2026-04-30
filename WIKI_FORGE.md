# BeMore Wiki Forge Protocol

This repository is the canonical knowledge base for Hermes / BeMore Buddy wiki output.

The agent must not treat wiki writing as a one-off project. Any durable knowledge it learns, infers, creates, verifies, or repeatedly uses should be converted into wiki pages.

## Mission

Create and maintain wikis for every meaningful knowledge domain the agent touches:

- games and creative worlds
- Prismtek products and apps
- BeMore / Buddy architecture
- agent runtime behavior
- models and local runtime notes
- project runbooks
- workflows and delivery history
- lore, characters, items, locations, mechanics, and design systems
- repo maps and integration notes

Pokemon Champions is the first game wiki. It is not the only wiki.

## Operating rule

When the agent finishes a task, it should ask:

> Did this create durable knowledge that future Buddy, Hermes, Prismtek, or project agents should know?

If yes, create or update a wiki page before marking the task complete.

## Page lifecycle

Every wiki page should move through these states:

1. `stub` — topic exists, minimal verified facts.
2. `draft` — useful structure and initial content.
3. `verified` — checked against source files, screenshots, runtime behavior, or user-provided truth.
4. `canonical` — stable enough to be linked from indexes and reused by agents.
5. `archived` — superseded but preserved for history.

## Trust levels

Every page should clearly separate:

- **Verified** — confirmed by source, user statement, build output, logs, screenshots, or repo files.
- **Working theory** — likely, but not proven yet.
- **Open questions** — needs confirmation.
- **Next actions** — concrete work for the agent.

Never bury uncertainty. Mark it.

## Naming

Use kebab-case paths:

```text
wiki/domains/<domain>/<topic>.md
wiki/projects/<project>/<topic>.md
wiki/repos/<repo-name>/<topic>.md
wiki/runbooks/<system>/<task>.md
wiki/lore/<world>/<topic>.md
```

Examples:

```text
wiki/domains/pokemon-champions/index.md
wiki/domains/pokemon-champions/champions.md
wiki/projects/bemore-agent/local-model-runtime.md
wiki/repos/prismtek-apps/ios-testflight.md
wiki/runbooks/bemore/build-delivery.md
```

## Required page front matter

Every new page should start with:

```yaml
---
title: Page Title
domain: domain-or-project
status: stub | draft | verified | canonical | archived
owner: BeMore Wiki Forge
last_verified: YYYY-MM-DD
sources:
  - user
  - repo:path/or/source
privacy: public | private | internal
---
```

## Required page sections

Use this structure unless a page has a strong reason not to:

```md
# Title

## Summary

## Verified Knowledge

## Working Theory

## Open Questions

## Related Pages

## Next Actions

## Change Log
```

## Agent behavior

When creating pages, the agent should:

1. Prefer small, accurate pages over one giant dump.
2. Link related pages aggressively.
3. Avoid inventing facts to fill sections.
4. Move uncertain content into `Working Theory` or `Open Questions`.
5. Update indexes whenever a new page is created.
6. Keep repo, delivery, and runtime pages separate from lore/world pages.
7. Commit wiki updates with clear messages like `wiki: add pokemon champions index`.

## Domains to maintain now

Initial domains:

- Pokemon Champions
- BeMore Agent
- Prismtek Apps
- Prismtek Site
- Hermes Agent
- Hermes Web UI
- Hermes Desktop
- Local Model Runtime
- TestFlight Delivery
- Pixel Tools
- Prismtek Modding
- OpenClaw / NemoClaw

Add domains whenever the agent sees repeated project knowledge that needs memory.
