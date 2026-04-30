# Agent Wiki Creation Directive

Use this directive for Hermes, BeMore Buddy, and MacBook agents.

## Prime directive

Whenever the agent learns or creates durable project knowledge, it must create or update wiki pages in this repository.

The agent should not wait for a user to say "make a wiki". Wiki creation is part of finishing the task.

## When to write a wiki page

Create or update wiki pages when a task includes any of these:

- a new game, app, repo, domain, feature, or tool
- repeated troubleshooting knowledge
- delivery or build history
- architecture decisions
- runtime behavior
- product specs or tech specs
- lore, canon, characters, creatures, mechanics, regions, items, UI systems, quests, maps, or progression
- workflows the agent will need again
- facts the user has had to repeat more than once

## Completion checklist

Before marking a task done, answer:

1. Did this produce durable knowledge?
2. Did I create or update the relevant wiki page?
3. Did I link it from the domain index?
4. Did I mark uncertainty clearly?
5. Did I include next actions?

If the answer to #1 is yes and #2 is no, the task is not complete.

## Scope rule

Create separate pages for separate concepts. Do not dump everything into one file.

Good:

```text
wiki/domains/pokemon-champions/creatures.md
wiki/domains/pokemon-champions/champions.md
wiki/domains/pokemon-champions/battle-system.md
wiki/projects/prismtek-apps/ios-testflight.md
wiki/projects/prismtek-apps/local-model-runtime.md
```

Bad:

```text
wiki/everything.md
```

## Privacy rule

If knowledge includes secrets, credentials, personal info, private customer/user data, or unreleased sensitive business data, do not publish it into a public wiki page. Use `privacy: private` or `privacy: internal`, redact secrets, and keep operational details minimal.

## Source rule

Pages must distinguish verified facts from theories.

Use:

- `Verified Knowledge` for confirmed facts.
- `Working Theory` for plausible but unconfirmed notes.
- `Open Questions` for unknowns.
- `Next Actions` for concrete follow-up.

## Default folders

Use these folders first:

```text
wiki/domains/
wiki/projects/
wiki/repos/
wiki/runbooks/
wiki/lore/
templates/
```

## First active priority

Pokemon Champions is the first game wiki. Expand it immediately, but also create/update pages for every other Prismtek, Hermes, BeMore, Buddy, local runtime, delivery, and creative-tools domain the agent touches.
