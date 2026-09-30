# AGENTS.md — ai-home-stack repo briefing

## What this is

Second-generation follow-on to ai-stack: a **self-hosted, voice-first AI medical
advocate and general voice assistant for exactly one user**, running on a home server.

## Hard constraint (same spirit as ai-stack, stricter)

**The user's real medical records never touch the internet.**

- Dev and cloud phases use **synthetic patient data only** (Synthea). No exceptions.
- Production: every model that touches records runs locally. No cloud inference.
- Cloud model APIs are dev-phase scaffolding only, swapped out at cutover (Phase 6).
- Flag anything that risks this constraint before adding it.

## Key decisions (details in docs/architecture.md)

1. **Qwen3 family for all three LLM roles** (locked 2026-09-29): Analyst = Qwen3-235B-A22B (FP8),
   Front desk = Qwen3 ~8B, Document reader = Qwen3-VL ~32B.
2. **Voice cascade, not speech-to-speech**: ASR → LLM → TTS through LiveKit, so every
   word is auditable text at the gateway. S2S (Moshi/Qwen-Omni) demo only.
3. **FHIR-native facts layer** (HAPI FHIR + Postgres), not GraphRAG. Qdrant + TEI
   remains for unstructured notes. Reconsider Neo4j only with proven need.
4. **Dev phase uses direct model APIs** (DashScope/OpenRouter), not Bedrock.
   Synthetic data only. Parity test suite guards the Phase 6 cutover to local weights.
5. **Home hardware sized for the Analyst**: 4× 96 GB GPUs (384 GB) baseline.
6. **PII posture is relaxed vs ai-stack**: single user, home machine. Basic audit
   logging stays; heavy redaction machinery does not. Threat model is physical
   access, backups, and remote access.

## Conventions

- **The runbook is written alongside the build**, never after. Format follows
  ai-stack's manual-build guide: per step — Goal, Commands, Expected output,
  What just happened, Verify, If it breaks.
- Branches: feature branches; don't commit directly to main unless asked.
- Secrets: never commit. `.env` generated on-box, mode 600, gitignored.
- Images: prefer digest pinning; version tags with freeze-digest comments as fallback.
- This machine (Anthony's workstation) cannot run GPU workloads. The agent writes,
  Anthony runs on the dev box and pastes output back.

## Current state

Phase 0 — repo bootstrap. Build order in README + docs/runbook.md.
