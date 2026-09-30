# Architecture decisions — ai-home-stack

Living decision record. Each entry: what we chose, what we rejected, why.
Newest at the bottom; never edit history, append corrections.

## 2026-09-29 — Initial shape (from the gen-2 planning session)

### D1. Three LLM roles, all Qwen3

| Role | Model | Why |
|---|---|---|
| Analyst | Qwen3-235B-A22B @ FP8 | Frontier-class open weight; Apache 2.0; first-class vLLM support |
| Front desk | Qwen3 ~8B | Latency matters more than IQ inside the voice loop |
| Document reader | Qwen3-VL ~32B | Scanned records, faxes, lab printouts → structured data |

One vendor family = one chat-template dialect, one serving stack, one set of
operational quirks. Chosen over: GLM 5.3 Flash (thinner quant/serving ecosystem,
tighter KV headroom at 384 GB) and mixed-vendor picks. GLM 5.3 full (744B) and
Kimi K3 (~1T) were priced out of the 4-GPU tier; the ~1T tier also forces 4-bit
quantization back in, which we rejected for medical reasoning.

**Contingency:** the dev phase A/B-tests Qwen vs GLM Flash on the parity suite
with the user's real (synthetic) workloads before hardware purchase. If GLM wins
decisively, the box grows to 6 GPUs.

**Update 2026-09-29 — LOCKED.** Qwen confirmed for all three roles; the GLM A/B
contingency is dropped. Swap-ability is preserved by design rather than by test:
models are config behind LiteLLM, not architecture. A future model change is a
gateway config edit plus a parity-suite run, not a stack rebuild. To keep that
promise true: model names live in env/config only, prompts stay model-agnostic,
and no role's logic may hardcode Qwen-specific behavior. Hardware baseline (D8)
confirmed at 4× 96 GB.

### D2. Voice cascade, not speech-to-speech

ASR (Parakeet/SenseVoice) → LLM → TTS (IndexTTS-2/CosyVoice2) through LiveKit.
Rejected: full-duplex S2S (Moshi, Qwen-Omni) for production — a black box you
cannot audit, redact, or guardrail. In a cascade every word is text at the
gateway. S2S may appear as a demo, never as the medical path.

### D3. FHIR-native facts layer, not GraphRAG

Medical advocacy needs a normalized record: one medication list, one problem
list, one allergy list, who prescribed what and when. That is a database, not a
vector index and (at one-patient scale) not a graph. HAPI FHIR (open source,
Postgres-backed) stores facts in medicine's native interoperability format.
Qdrant + TEI (bge-m3 + reranker) remains for unstructured note retrieval.
Neo4j/LightRAG deferred until a proven need exists.

### D4. Dev phase on direct model APIs

Pay-per-token Qwen via DashScope/OpenRouter instead of renting GPU instances
($10–99/hr) or standing up Bedrock. Rationale: bursty dev workloads cost dollars
per week; LiteLLM abstracts the provider so cutover is a config change.
Rejected: Bedrock (adds IAM plumbing for no dev-phase benefit here; the
Bedrock/IAM learning goals are covered by the reserve-bank-demo project).

### D5. Synthetic data only until cutover

Synthea generates the dev patient population. Real records are created only on
the home machine, after Phase 6. This keeps "records never touch the internet"
compatible with cloud-hosted dev models.

### D6. Parity suite guards the cutover

20–30 fixed prompts (med reconciliation, lab trend, visit-prep brief, doc
extraction, tool calls) run against both the API models and the local weights
before cutover. Managed/hosted variants of open models drift from self-hosted
weights (serving stack, chat template defaults, context limits). The parity
suite catches it before the user does.

### D7. Relaxed PII posture (vs ai-stack)

Single user, home machine, no multi-tenant ACLs. Kept: audit logging of record
access, encrypted disk, off-box encrypted backups. Dropped: regex PII redaction
outlets, role-based chunk ACLs. Threat model: physical access, backup handling,
remote access — not multi-user leakage.

### D8. Home hardware baseline

4× 96 GB GPU (384 GB VRAM) server: Analyst at FP8 (~235 GB) + ~100 GB KV cache
headroom + room for front-desk/VL/ASR/TTS models. Baseline cost $40–50K.
Sizing confirmed 2026-09-29 with the D1 lock.
