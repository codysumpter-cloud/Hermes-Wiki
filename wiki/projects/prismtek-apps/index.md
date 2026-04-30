---
title: Prismtek Apps
domain: prismtek-apps
status: stub
owner: BeMore Wiki Forge
last_verified: 2026-04-30
sources:
  - repo:codysumpter-cloud/prismtek-apps
  - user
privacy: internal
---

# Prismtek Apps

## Purpose

Living wiki for the `prismtek-apps` repository, including BeMoreAgent iOS/macOS work, TestFlight delivery, local model packaging, build numbers, runtime choices, and repo structure.

## Canonical Pages

- [TestFlight Delivery](../../runbooks/bemore/testflight-delivery.md)
- [Local Model Runtime](../../domains/local-model-runtime/index.md)

## Draft Pages

- iOS native app structure
- Build 47-52 delivery history
- Bundled Gemma 4 package
- MLC runtime link contract
- Desktop app upgrade path

## Verified Knowledge

- The repo contains the BeMoreAgent native iOS app under `apps/bemore-ios-native`.
- Build delivery and model packaging are active areas.
- The current intended bundled local model is Gemma 4 E2B IT MLC.

## Working Theory

- This repo should own app release runbooks while broader product behavior belongs in the BeMore Agent wiki.

## Open Questions

- Which macOS app paths should be documented alongside iOS?
- Which historical build failures should be summarized as lessons learned?

## Next Actions

- Create a page for the iOS app source map.
- Create a page for Build 47-52 delivery history.
- Create a page for bundled Gemma 4 model packaging.

## Change Log

- 2026-04-30: Created Prismtek Apps wiki index.
