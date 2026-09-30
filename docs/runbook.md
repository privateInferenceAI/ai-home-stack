# ai-home-stack runbook

**Living document — written alongside the build, phase by phase.**
Style follows ai-stack's manual-build guide: every step gives the goal, the
commands, the expected output, a plain-English explanation, a verification
probe, and the top failure causes. Detail is added as each phase is built;
trim later if it gets verbose.

Conventions:
- `$ ` prefix = run as your user. `# ` = run as root / with sudo.
- Secrets live in `/opt/ai-home-stack/.env`, mode 600, generated on-box. Never committed.
- Everything on the home server lives under `/opt/ai-home-stack/`.
- Dev-phase cloud models see **synthetic data only**. Real records never leave the home machine.

---

## Phase 0 — Foundations

**Outcome:** decisions logged, dev API access working, a synthetic patient to
build against, a dev box to build on. Nothing here touches real medical data.

### 0.1 Accounts and API keys

**Goal:** pay-per-token access to the Qwen3 family for the dev phase.

1. Create an Alibaba Cloud Model Studio (DashScope) account and generate an API
   key. DashScope serves Qwen3 models over an OpenAI-compatible endpoint
   (`https://dashscope.aliyuncs.com/compatible-mode/v1`).
2. Alternative or supplement: an OpenRouter key (useful for A/B-testing GLM
   5.3 Flash against Qwen on identical prompts — see architecture D1/D6).
3. Record which models the dev phase will use:
   - Analyst: `qwen3-235b-a22b` (hosted)
   - Front desk: `qwen3-8b` (hosted, for logic dev only — runs locally from Phase 4)
   - Document reader: `qwen3-vl-32b` (hosted)

**Verify:** a raw curl chat-completion against the DashScope endpoint returns a
completion from `qwen3-235b-a22b`. Paste the model's response header/body excerpt.

**If it breaks:** 401 = key wrong or not activated for the model; 404 = model
name mismatch (DashScope names differ from HuggingFace names — check the
console's model list).

### 0.2 The synthetic patient

**Goal:** a realistic fake person whose records we can safely send to cloud APIs.

1. Install Synthea on the dev box (`git clone` + gradle build, or the prebuilt jar).
2. Generate one patient matching the real user's profile in the abstract:
   male, age ~54, type 2 diabetes, stage 5 CKD on dialysis, hypertension —
   Synthea modules: `diabetes`, `chronic_kidney_disease`, `hypertension`,
   plus a decade of primary-care history.
3. Export as FHIR R4 bundles (Synthea's native output) **and** generate the
   "messy real world" corpus: print a subset of encounters/labs to PDF, scan-
   effect some of them (the document reader's job in Phase 2).

**Verify:** the FHIR bundle contains Conditions (E11*, I10, N18.6), at least one
MedicationRequest, and Observations with HbA1c + eGFR series over time.

**If it breaks:** Synthea disease modules are config-driven — if the patient
comes out without dialysis, check module ordering (CKD progression needs years
of simulated history; raise the age range or run more seeds).

### 0.3 The dev box

**Goal:** a cheap always-on host for the text stack (GPU comes in Phase 4).

- AWS `t3.large` (2 vCPU / 8 GB), Ubuntu 24.04, 60 GB gp3, in the user's dev
  account. No GPU needed yet.
- Baseline hardening (same as ai-stack phase1a/1b teaches): updates, UFW deny
  incoming except SSH, fail2ban, no password auth.
- Repo cloned to `/opt/ai-home-stack`; `.env` generated on-box, mode 600.

**Verify:** `ufw status` shows only 22/tcp allowed; `fail2ban-client status sshd`
shows the jail active.

### 0.4 Decisions log

`docs/architecture.md` already records D1–D8. Any decision made during a phase
gets appended there in the same format the week it is made. No oral tradition.

---

## Phase 1 — Gateway core (text)

**Outcome:** LiteLLM as the single control point, three Qwen routes behind it,
a chat UI, per-key spend logging in Postgres. All against synthetic data.

### 1.1 Docker + network

Install Docker CE; create external bridge `ai-home-net`. Same pattern as
ai-stack phase1b, minus the NVIDIA toolkit (no GPU on this box).

### 1.2 Postgres

Container, unpublished port, named volume. Why the gateway needs a database:
virtual keys, budgets, and spend logs live here — API management, not a cache.

### 1.3 LiteLLM

Container, port 4000. `config.yaml` with three model entries pointing at
DashScope's OpenAI-compatible endpoint:
`analyst` → qwen3-235b-a22b · `front-desk` → qwen3-8b · `doc-reader` → qwen3-vl-32b.
Master key + DB URL from `.env`.

**Verify:** `curl localhost:4000/health/liveliness`; then a chat call to
`analyst` with a virtual key and a second call to a key whose `models` ACL
excludes `analyst` — the second must fail. That failure *is* the API-management
story working.

### 1.4 Open WebUI

Container, port 3000, model-filtered to the gateway models, signups disabled
after the admin account exists, telemetry off. SSH-tunnel access only.

### 1.5 First end-to-end probe

Ask the analyst (via the UI) a question answerable only from the synthetic
patient's FHIR bundle pasted into context. Confirms: routing, keys, spend
logging (`SpendLogs` row appears in Postgres), and that the model grounds on
supplied records.

---

## Phase 2 — Medical memory  *(outline — detailed when built)*

- Qdrant + TEI (bge-m3 embeddings, bge-reranker) for unstructured notes.
- Ingestion v2: ai-stack's pipeline + OCR/parse stage routed through the
  doc-reader model; corrupt-file dead-lettering carried over.
- HAPI FHIR server (Docker) + Postgres; Synthea bundle import.
- Facts extraction: analyst model proposes FHIR resources from each ingested
  document → human confirms → stored. The confirmed store is the chart.
- RAG-in-context via an Open WebUI filter function (gen-1 pattern), upgraded:
  FHIR facts injected as structured context alongside vector chunks.

## Phase 3 — The advocate  *(outline)*

- Medication reconciliation across prescribers (deterministic merge + analyst review).
- Care timeline view; lab trend watches (HbA1c, eGFR, potassium).
- Visit-prep brief generator (per specialist, from FHIR + recent notes).
- Post-visit reconciliation: new document → diff against chart → flag conflicts.
- n8n workflows + MCP tool layer (calendar, reminders, email drafts).

## Phase 4 — Voice  *(outline)*

- GPU dev box (g6e.2xlarge, hourly, powered on only during voice work).
- LiveKit server container; Parakeet/SenseVoice ASR; IndexTTS-2/CosyVoice2 TTS.
- Front-desk model goes **local** here (first self-hosted LLM of the build).
- Barge-in, VAD tuning, latency budget (<1.5 s end-to-end target).

## Phase 5 — The physical world  *(outline)*

- Home Assistant bridge; proactive spoken reminders (meds, appointments).

## Phase 6 — Cutover  *(outline)*

- Home hardware lands (4× 96 GB baseline). vLLM serving Qwen3-235B FP8.
- Parity suite (D6) green against local weights → LiteLLM config swap.
- Cloud API keys revoked; dev box decommissioned. Real records come home.

## Phase 7 — Hardening  *(outline)*

- LUKS at rest, encrypted off-box backups + restore drill (ai-stack backup.sh pattern),
  Tailscale-only remote access, UPS/NUT, monitoring + alerting.
