# ai-home-stack

**A self-hosted, voice-first AI advocate for one person.** It answers the phone-like
questions ("when is my next appointment?"), does the deep work no human has time for
(reconciling five specialists' records into one chart), and speaks up when something
needs attention ("your potassium came back high — ask your nephrologist about it").

Built as the second-generation follow-on to
[ai-stack](https://github.com/privateInferenceAI/ai-stack) (private business AI stack).
Same engineering discipline, new mission: one user, at home, voice-first, medical-advocate-grade.

## Hard constraint

**The user's real medical records never touch the internet.**

- All development and cloud work uses **synthetic patients only** (Synthea-generated).
- In production, every model that touches records runs locally on the home machine.
- Cloud model APIs (used during development only) never see real data.
- Flag anything that risks this constraint before adding it.

## Architecture (target state)

Single home server, Docker Compose, three LLM roles — all from the Qwen3 family
(one serving stack, one template dialect, one vendor's quirks):

| Role | Model (dev via API / prod local) | Job |
|---|---|---|
| **Analyst** | Qwen3-235B-A22B (FP8) | Deep medical reasoning: record synthesis, med reconciliation, visit prep |
| **Front desk** | Qwen3 small (~8B class) | Lives in the real-time voice loop; fast, warm, interruptible |
| **Document reader** | Qwen3-VL (~32B class) | Parses scanned PDFs, faxes, lab printouts into structured data |

Supporting cast: LiveKit (real-time voice), Parakeet or SenseVoice (ASR),
IndexTTS-2 or CosyVoice2 (TTS), LiteLLM (gateway — the single control point),
Qdrant + TEI (vector RAG over notes), HAPI FHIR + Postgres (structured medical
facts, in medicine's native format), n8n + MCP (tools, calendar, reminders),
Home Assistant (physical world, later phase).

Voice is a **cascade** (ASR → LLM → TTS), not end-to-end speech-to-speech: every
word exists as text at the gateway, where it can be logged and audited. See
`docs/architecture.md` for the reasoning behind every choice.

## Build order

The runbook (`docs/runbook.md`) is written **alongside** the build, phase by phase,
in the same longhand style as ai-stack's manual-build guide.

- **Phase 0 — Foundations.** Decisions log, dev API keys, Synthea synthetic patient, dev box.
- **Phase 1 — Gateway core.** Docker host, Postgres, LiteLLM, three Qwen API routes, first chat call.
- **Phase 2 — Medical memory.** Qdrant + TEI RAG, VLM document ingestion, HAPI FHIR facts layer.
- **Phase 3 — The advocate.** Medication reconciliation, care timeline, visit-prep briefs, post-visit conflict checks; n8n + MCP tools.
- **Phase 4 — Voice.** GPU box, LiveKit, ASR + TTS, local front-desk model, barge-in, latency tuning.
- **Phase 5 — The physical world.** Home Assistant, proactive spoken reminders.
- **Phase 6 — Cutover.** Home hardware, vLLM, Qwen3-235B at FP8, parity test suite, gateway config swap, cloud decommissioned.
- **Phase 7 — Hardening.** Encryption at rest, backups + restore drill, remote access, power, monitoring.

## Status

🚧 Phase 0 — repo bootstrap. See `docs/runbook.md`.
