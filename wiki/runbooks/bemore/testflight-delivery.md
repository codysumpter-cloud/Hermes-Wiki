---
title: BeMore TestFlight Delivery
domain: bemore-testflight-delivery
status: stub
owner: BeMore Wiki Forge
last_verified: 2026-04-30
sources:
  - repo:codysumpter-cloud/prismtek-apps
  - user
privacy: internal
---

# BeMore TestFlight Delivery

## Summary

Living wiki/runbook for delivering BeMoreAgent iOS builds to TestFlight.

## Verified Knowledge

- The BeMore iOS CI & TestFlight workflow is the main release lane for the native iOS app.
- Build 52 is intended to bundle the Gemma 4 E2B IT MLC package.
- The app package must verify bundled model files inside the archived `.app` before upload success is claimed.

## Working Theory

- Delivery failures should be summarized here after they are fixed, so future agents do not repeat the same mistakes.

## Open Questions

- Has the latest Build 52 upload completed successfully in App Store Connect?
- Is the build assigned to internal or external tester groups?

## Related Pages

- [Prismtek Apps](../../projects/prismtek-apps/index.md)
- [Local Model Runtime](../../domains/local-model-runtime/index.md)

## Next Actions

- Add Build 47-52 timeline.
- Add known failure modes: signing, runner queue, bundle resources, model identity, archive verification.
- Link the repo runbook from `apps/bemore-ios-native/ADMIN_TESTFLIGHT_RUNBOOK.md`.

## Change Log

- 2026-04-30: Created TestFlight delivery wiki page.
