---
title: Prismtek Site
domain: prismtek-site
status: stub
owner: BeMore Wiki Forge
last_verified: 2026-04-30
sources:
  - repo:codysumpter-cloud/prismtek-site
  - user
privacy: internal
---

# Prismtek Site

## Purpose

Living wiki for the Prismtek website, account flows, Buddy web UI, public/private knowledge boundaries, deployment, route compatibility, and site UX decisions.

## Canonical Pages

- [BeMore Agent](../bemore-agent/index.md)

## Draft Pages

- Account creation
- Buddy creation flow
- Buddy web UI surface
- User privacy boundaries
- Website deployment and routing

## Verified Knowledge

- The site is intended to let users create accounts and have a Buddy.
- The Buddy web UI is derived from Hermes web UI concepts but converted into BeMore Buddy product language.

## Working Theory

- The website should expose Buddy usage while keeping private chats and user data isolated to each account.

## Open Questions

- Which auth provider is canonical for production?
- Where should Buddy profile and memory data be stored?
- Which parts of Hermes Web UI should remain admin-only?

## Next Actions

- Create pages for account onboarding, Buddy privacy, and web UI routes.
- Link implementation notes to `prismtek-site` PRs and runbooks.

## Change Log

- 2026-04-30: Created Prismtek Site wiki index.
