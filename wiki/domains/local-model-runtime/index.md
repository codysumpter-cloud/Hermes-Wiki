---
title: Local Model Runtime
domain: local-model-runtime
status: stub
owner: BeMore Wiki Forge
last_verified: 2026-04-30
sources:
  - repo:codysumpter-cloud/prismtek-apps
  - user
privacy: internal
---

# Local Model Runtime

## Purpose

Living wiki for on-device/local model support across BeMore Agent, including MLC packages, bundled model delivery, runtime linking, native libraries, model IDs, and real-device validation.

## Canonical Pages

- [Prismtek Apps](../../projects/prismtek-apps/index.md)
- [BeMore Agent](../../projects/bemore-agent/index.md)

## Draft Pages

- Gemma 4 E2B IT bundled package
- MLC/TVM native runtime linking
- Real-device first-token validation
- Memory pressure recovery

## Verified Knowledge

- The current intended bundled iOS model is `welcoma/gemma-4-E2B-it-q4f16_1-MLC`.
- Bundling the model package solves the fragile in-app multi-shard download path.
- True local token generation still requires native runtime linking and real-device proof.

## Working Theory

- Bundled model delivery and native runtime linking should remain separate wiki pages so delivery success is not confused with first-token generation.

## Open Questions

- Has Build 52 with Gemma 4 been uploaded and installed from TestFlight?
- Has real-device first-token generation been proven with native MLC/TVM runtime linked?

## Next Actions

- Document the Gemma 4 package delivery path.
- Document the native runtime link contract.
- Add real-device validation checklist.

## Change Log

- 2026-04-30: Created local model runtime wiki index.
