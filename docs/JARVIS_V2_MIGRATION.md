# JARVIS V2 — Migration Plan

Status: **DRAFT v0.1. No refactor has started. Each phase from Phase 1 on needs explicit approval.**
Related: [ARCHITECTURE](JARVIS_V2_ARCHITECTURE.md) · [FEATURE MATRIX](JARVIS_V2_FEATURE_MATRIX.md) · [MODEL STRATEGY](JARVIS_V2_MODEL_STRATEGY.md) · [CONTEXT ENGINE](JARVIS_V2_CONTEXT_ENGINE.md)

---

## Current status and blockers

| # | Blocker | Impact | Unblock |
|---|---|---|---|
| B1 | **The source isn't in the repo.** `github.com/simogalli26-cmd/Jarvis` was empty (no branches, no commits) when these docs were written. `C:\JARVIS` is only on the local machine. | The code audit, test runs, log review, benchmarks and preflight are all impossible from a cloud session. Every "current JARVIS" claim is from the brief. | Push `C:\JARVIS` (stable branch + tags), following the secrets checklist below |
| B2 | **The repo is public.** | Pushing `C:\JARVIS` as-is could publish OAuth tokens, API keys, personal memory and recordings | Run the secrets checklist **before** the first push, and consider making the repo private |
| B3 | Benchmarks need the real machine: RTX 5060, local Qwen, API keys, real mic, real latency from Italy | STT and model choices stay provisional | Run Phase 1 and Phase 2 benchmarks locally, or from a session with access |
| B4 | OpenAI prices and model ids are unverified (official page unreachable from the build environment) | OpenAI rows in the model strategy may be wrong | List models available to the key; check the official pricing page |
| B5 | Branch name: the harness requires `claude/jarvis-v2-architecture-akxnhh` (the brief asked for `jarvis-v2-architecture`) | Cosmetic | Rename or recreate locally as `jarvis-v2-architecture` if preferred |
| B6 | This branch is a root commit (the repo was empty), unrelated to the local history | A plain merge needs `--allow-unrelated-histories` | Use option A below |

### Bringing these docs together with the real code

**Option A (recommended).** Push the real project, then let the next session rebase the docs onto it:

```bash
cd C:\JARVIS
git status                                   # clean, or commit first
git tag v1-stable-pre-v2                     # recovery checkpoint (skip if a stable tag exists)
# >>> run the secrets checklist below first <<<
git remote add origin https://github.com/simogalli26-cmd/Jarvis.git   # if not set
git push -u origin main --tags               # use your stable branch name
```

Then, on GitHub: Settings → Branches → set the default branch to `main`. The docs branch was
pushed first to an empty repo, so GitHub may have made it the default.

**Option B.** Copy the docs into the local repo yourself:

```bash
git fetch origin claude/jarvis-v2-architecture-akxnhh
git switch -c jarvis-v2-architecture
git checkout origin/claude/jarvis-v2-architecture-akxnhh -- docs/JARVIS_V2_*.md
git commit -m "docs: JARVIS V2 architecture drafts"
```

### Secrets checklist (before any push to a public repo)

- `.gitignore` covers at least: `.env`, `*.env`, `token*.json`, `credentials*.json`, `client_secret*.json`, `*.pem`, `*.key`, `config/*secret*`, `logs/`, `*.log`, personal memory data (`memory/data`, `*.db`, `*.sqlite`), recordings (`*.wav`), model weights, `venv/`, `__pycache__/`.
- Check history, not just the working tree: `git log --all --oneline -- "*token*" "*credential*" "*.env"`, and search for key prefixes (`sk-`, `AIza`, `ya29.`). Optional: run `gitleaks detect`.
- If a secret was **ever** committed: rotate it (new key / revoke the OAuth token). Removing it from history isn't enough once it's public.
- Consider making the repo private (Settings → General → Danger Zone → Change visibility).

---

## Ground rules for every phase

1. **Branch.** All V2 work stays on the V2 branch. `v1-stable-pre-v2` stays untouched.
2. **Flags.** Every new subsystem sits behind a flag that defaults off until its phase's exit criteria pass: `V2_CONTEXT_ENGINE`, `V2_ROUTER`, `V2_AGENT_LOOP`, `V2_ARTIFACTS`, `V2_MCP`, `V2_MEMORY`, `V2_SCREEN`, `V2_FOCUS`. Rollback = flip the flag.
3. **Shadow first.** Router and Context Engine run in shadow (decide and log, don't act) before they take control.
4. **Gate.** A phase is done when (a) preflight passes, (b) the intelligence eval shows no regression, (c) its own exit criteria pass, and (d) the user approves the next phase.
5. **No deletions.** Replaced code is retired behind a flag first, then removed in a later phase ("REMOVE LATER").

---

## Phase 0 — Checkpoint + audit + diagnostics + preflight

**Goal:** know the truth before changing anything. *Allowed without further approval:* tiny diagnostics/preflight scaffolding and obviously safe fixes.

| Step | Deliverable |
|---|---|
| 0.1 | Source in the repo (B1/B2), tag `v1-stable-pre-v2`, V2 branch created |
| 0.2 | **Code audit**: read `voice/ brain/ tools/ permissions/ integrations/ orb3d/ launcher/ memory/ tests/ docs/`; confirm or correct every **(brief)** claim in the [ARCHITECTURE §6–7](JARVIS_V2_ARCHITECTURE.md#6-non-negotiables-preserved-brief-5) tables; final KEEP/IMPROVE/REPLACE/MERGE/DEFER/REMOVE LATER per subsystem |
| 0.3 | Run the existing tests; record the baseline pass/fail |
| 0.4 | Log review: top failure modes, polling spam, unrotated logs, and transcripts in logs (privacy) |
| 0.5 | `/diag` endpoint: whitelist only, read from the running process, plus a whitelist test and a canary test |
| 0.6 | `preflight` skeleton with checks 1–12 from [ARCHITECTURE §5.2](JARVIS_V2_ARCHITECTURE.md#52-preflight-phase-0-grows-forever) and an exit code |
| 0.7 | Per-turn metrics JSONL (latencies, provider, model; no transcripts) |
| 0.8 | **Intelligence eval harness**: scripted multi-turn conversations A–E + the [benchmark set](JARVIS_V2_MODEL_STRATEGY.md#8-benchmark-plan), deterministic assertions, run against **current** JARVIS to get a baseline score |

**Exit:** audit table final; preflight runs (FAILs allowed but known); a baseline eval score exists.
That score is the number V2 has to move. "Feels stupid" becomes measurable.

---

## Phase 1 — STT + voice reliability

**Goal:** JARVIS hears correctly.

1. **Audio capture audit** (before touching models). Dump ~10 captured utterances locally under DEBUG and delete them afterwards. Check:
   - sample rate and resampling path (16 kHz mono into Whisper);
   - channels;
   - peak dBFS (clipping) and RMS (gain);
   - first-word truncation (pre-roll buffer);
   - trailing truncation (endpointing hang time);
   - VAD thresholds;
   - silence ratio.
2. **Centralise config:** `STT_MODEL`, `STT_DEVICE`, `STT_COMPUTE_TYPE`, `STT_LANGUAGE`, `STT_BEAM_SIZE`, `STT_VAD`, `STT_HINTS_FILE`.
3. **GPU bring-up on RTX 5060 (Blackwell, sm_120)**. Verify, don't assume:
   - a smoke test loads `large-v3-turbo` with `device="cuda"`, transcribes a 5 s clip, and reports the device actually used plus VRAM;
   - if the installed CTranslate2 wheel doesn't support sm_120: (a) try a CTranslate2 build against CUDA 12.8+; (b) fall back to a PyTorch Whisper backend with CUDA 12.8 wheels; (c) try whisper.cpp with CUDA; (d) last resort `large-v3-turbo` on CPU int8 (slow; measure);
   - check VRAM headroom with local Qwen loaded at the same time (8 GB card).
4. **Benchmark** 30–50 real Italian utterances recorded through the real capture path. Include commands and domain words. Transcribe references by hand.

   | Config | Model | Device | Compute | Beam | Hints |
   |---|---|---|---|---|---|
   | baseline | small | cpu | int8 | current | current |
   | T1 | large-v3-turbo | cuda | float16 | 5 | off |
   | T2 | large-v3-turbo | cuda | float16 | 5 | on |
   | T3 | large-v3-turbo | cuda | int8_float16 | 5 | on |
   | T4 | large-v3-turbo | cuda | float16 | 1 | on |

   Metrics: WER, domain-term accuracy, latency for a 3 s utterance (p50/p95), VRAM.
5. **Domain hints file** (editable): Jarvis, Spotify, Rocket League, VS Code, Google Drive, Google Calendar, Claude, ChatGPT, … Never personal memory.
6. Port the **mute flag** (one speak function) and a single named **endpoint hang-time** constant (pack P02).

**Exit:**
- WER ≥ 40 % lower (relative) than baseline;
- domain terms ≥ 95 % correct;
- p50 STT latency ≤ 400 ms for a 3 s utterance;
- preflight check 5 asserts `device == cuda`.

---

## Phase 2 — Intelligence engine: context engine + model router

**Goal:** JARVIS understands and remembers within the conversation. This is the core phase.

| Step | Deliverable | Flag / mode |
|---|---|---|
| 2.1 | Provider adapter interface (Anthropic / OpenAI / local), with streaming, tool calls, usage/cost capture and the model-specific quirks in [MODEL STRATEGY §10](JARVIS_V2_MODEL_STRATEGY.md#10-provider-adapter-notes-claude-models) | — |
| 2.2 | **Model benchmark** ([MODEL STRATEGY §8](JARVIS_V2_MODEL_STRATEGY.md#8-benchmark-plan)), then pick SMART/FAST/PREMIUM from evidence | — |
| 2.3 | Router: signals, tiers, modes incl. AGENT; **shadow mode** logs decisions next to the current routing | `V2_ROUTER=shadow` |
| 2.4 | Personality centralised (`personality/` variants) | — |
| 2.5 | Context Engine: layers, discourse state, entity registry, referent resolver, summaries, session digests | `V2_CONTEXT_ENGINE` |
| 2.6 | Switch the BALANCED default brain to SMART; T0 scope narrowed; stickiness + escalation | `V2_ROUTER=on` |
| 2.7 | Brain swap with alias map and refusal; HUD model chip | — |
| 2.8 | Domain-scoped tool selection + `request_tools` | — |

**Exit:**
- acceptance tests A–E pass in the eval;
- eval score clearly above the Phase 0 baseline;
- no T0 regressions (direct commands still instant);
- p50 time-to-first-audio for conversational turns within budget;
- cost per day within the [envelope](JARVIS_V2_MODEL_STRATEGY.md#9-cost-envelope-anthropic-list-prices-estimates).

---

## Phase 3 — Multi-step agent

| Step | Deliverable |
|---|---|
| 3.1 | `TaskState` persistence (SQLite) + pause/resume |
| 3.2 | Agent loop with `AGENT_MAX_STEPS` / `AGENT_MAX_SECONDS` / correction limits |
| 3.3 | Tool arg validation (JSON Schema + semantic validators) → structured correctable failures |
| 3.4 | Action ledger + duplicate-write protection (idempotency keys) |
| 3.5 | Tool verifiers; completion check; final report rendered **from `TaskState`** |
| 3.6 | Taint tracking + permission precedence (FORBID > CONFIRM > ALLOW > DEFAULT) |
| 3.7 | Stop/interrupt → task `paused`, resumable |

**Exit:**
- mocked-tool chains pass, including failure injection (Drive error → partial report, no "fatto");
- duplicate Calendar create is rejected;
- the injection test proposes no unrequested write.

---

## Phase 4 — Artifact / PDF engine + Drive upload

| Step | Deliverable |
|---|---|
| 4.1 | `create_text`, `create_markdown`, `create_pdf` (+ `create_docx` if practical) from structured `Document` JSON |
| 4.2 | Italian-capable embedded font; predictable `ARTIFACTS_DIR`; verifiers |
| 4.3 | `drive_upload_file(local_path, folder_id?, name?)` with path policy + md5 read-back |
| 4.4 | Web research tools hardened: untrusted wrapping, source tracking |
| 4.5 | **Real multi-step acceptance test** (below) |

### Real multi-step acceptance test (brief §29)

Prompt: *"Cerca informazioni sui sommari, crea un PDF ordinato con quello che trovi e caricalo sul mio Drive."*

1. Real web research, with ≥ 3 sources recorded in `TaskState.source_urls`.
2. Synthesis into a `Document` JSON with a sources section.
3. Real PDF at `ARTIFACTS_DIR/TEST_JARVIS_sommari_<timestamp>.pdf`; the verifier passes (exists, size > 0, readable, title present).
4. Real Drive upload; read-back metadata; md5 match.
5. Final report lists the steps and evidence (path, Drive link).
6. **Record:** steps, tools, tier/model per step, wall time, tokens/cost, final result.
7. **Cleanup:** trash the Drive TEST file and delete the local TEST file, then record that cleanup happened.

**Exit:** the acceptance test passes twice in a row; it is also added to `preflight --full`.

---

## Phase 5 — MCP + integrations

- Official MCP client; server config with `command`, `args`, `env_allowlist`; per-server trust level.
- MCP tools registered as `mcp:<server>`; unknown writes `DANGEROUS`; outputs untrusted (taint).
- First servers, chosen by value: e.g. Gmail (read-only first), Notion, GitHub.
- **Exit:** an MCP server process sees only allowlisted env vars (test); a write tool requires confirmation; preflight checks each configured server's handshake.

## Phase 6 — Memory V2 + skills + school mode

- `memory/notes/<category>/*.md` + SQLite FTS5 index; migration from the current store; the `remember/recall/update/forget` API unchanged.
- Secret filter on write. Writes are indexed immediately (retrievable next turn). Failed captures are spoken.
- `skills/<name>/SKILL.md` loader, with the initial skills `school-notes`, `research-pdf`, `organize-day`, `study-session`, `website-build`, `coding-task`.
- School flow: transcript → cleanup → structured summary → study notes → key concepts → questions → flashcards → study sheet → PDF (Goodnotes-ready export).
- **Exit:** memory round-trip in preflight; the old API tests still pass; one end-to-end school-notes PDF.

## Phase 7 — Screen awareness

- Ask mode: fresh native capture at question time; honest failure.
- Watch mode: local thumbnail diff every 5 s; stillness threshold (default 4 min); one frame per trigger; cooldown; HUD indicator always visible.
- **Exit:** a test proves no cloud call happens on ordinary screen changes; stale-frame answers are impossible (capture timestamp ≥ question timestamp); indicator state appears in `/diag`.

## Phase 8 — Focus sessions

Adapted pack P09–P12 for Windows:

- server tick;
- fresh foreground-window read;
- host-hash tab identity;
- defer → settle → announce lock;
- drift callouts, snooze, excuse, pause, resume, retarget, abort;
- report card;
- aggregates-only ledger.

**Exit:** the ledger whitelist test and the canary-label privacy test pass; `/diag` shows deferred/settle state.

## Phase 9 — Performance + polish

- Sentence-streamed TTS from the LLM stream; prompt-cache hit rate ≥ 80 % on conversational turns.
- Latency budget per stage, set from Phase 0 metrics.
- Personality swap lines (P15), source/provenance display for research answers (P03), log hygiene final pass.
- Build mode: external coding worker in a workspace boundary.

---

## Approval gate

These docs complete the design step. **Nothing beyond Phase 0 starts without explicit approval.**

**First implementation step, once B1/B2 are resolved:** Phase 0, steps 0.2 → 0.8.
That means the audit, `/diag`, the preflight skeleton, metrics, and an intelligence-eval baseline of the current JARVIS.
