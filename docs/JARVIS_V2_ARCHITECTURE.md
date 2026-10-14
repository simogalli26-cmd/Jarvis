# JARVIS V2 — Architecture

Status: **DRAFT v0.1 — design only, no code changed.**
Branch: `claude/jarvis-v2-architecture-akxnhh`
Companion docs: [MIGRATION](JARVIS_V2_MIGRATION.md) · [FEATURE MATRIX](JARVIS_V2_FEATURE_MATRIX.md) · [MODEL STRATEGY](JARVIS_V2_MODEL_STRATEGY.md) · [CONTEXT ENGINE](JARVIS_V2_CONTEXT_ENGINE.md)

> **Evidence caveat.** The JARVIS source (`C:\JARVIS`) is not in this repository yet.
> The GitHub repo was empty when this doc was written, so `voice/`, `brain/`, `tools/`,
> `docs/ARCHITECTURE.md`, `docs/PROGRESS.md`, tests and logs could not be read.
> Everything said here about the *current* system comes from the V2 brief and is marked
> **(brief)**. Each of those claims must be confirmed in the Phase 0 audit before
> any refactor ([MIGRATION §Phase 0](JARVIS_V2_MIGRATION.md#phase-0--checkpoint--audit--diagnostics--preflight)).
> The reference pack *was* read in full and is grounded.

---

## 1. Product goal

The main complaint is **"JARVIS feels stupid."** V2 is about intelligence first and features second.

> JARVIS V2 = a capable modern AI assistant that happens to have local tools,
> not a small local model with commands attached.

Order of importance (from the brief, binding for every trade-off in these docs):

1. understand the user
2. remember context
3. plan correctly
4. complete goals
5. use tools reliably
6. be fast enough
7. be extensible
8. extra features later

### Why it feels stupid: the diagnosis

These are structural causes, inferred from the brief's symptom list. Confirm them against logs in Phase 0.

| Symptom (brief) | Likely structural cause | V2 fix |
|---|---|---|
| Misunderstands context | Weak default brain (local Qwen) does conversational reasoning | SMART cloud tier is the default brain ([MODEL STRATEGY](JARVIS_V2_MODEL_STRATEGY.md)) |
| Loses track of previous turns | Context is prose history only, truncated or not curated | Context Engine with structured discourse state ([CONTEXT ENGINE](JARVIS_V2_CONTEXT_ENGINE.md)) |
| Poor ambiguous follow-ups ("aprilo", "l'altro") | No entity/referent tracking; the model must re-derive referents from raw text | Entity registry + referent candidates injected into context |
| Can't plan complex tasks | Single-shot tool call per turn; no task state | Multi-step agent loop + structured `TaskState` |
| Uses tools without understanding the goal | Router picks a tool before the goal is understood; all tool schemas in every prompt | Goal-first planning; domain-scoped tool selection |
| Behaves like a command parser | Deterministic router answers things that need reasoning | Deterministic router stays as **tier 0** for unambiguous commands only; everything else goes to the brain |

---

## 2. Principles

1. **Evolution, not reset.** Every working subsystem stays until a measured replacement beats it.
2. **Honesty is architectural.** "Fatto" is generated from verified `ToolResult`s, never from model prose.
3. **Structured state beats prose.** Task state, entities and artifacts are data, not chat history.
4. **The right model, the right context, the right tools, every request.** Routing is deterministic and costs no extra model call.
5. **Fail closed.** Unknown tools, unknown models and tainted contexts default to the stricter path.
6. **Instrument before you fix** (pack P16). Diagnostics and preflight come before features.
7. **"Done" means preflight passed** (pack P07).

---

## 3. Target request flow

```
 VOICE ─┐   TEXT ─┐   SCREEN ─┐   FILE ─┐
        └────────┴───────────┴─────────┘
                     │  InputEvent{modality, payload, ts, session_id}
                     ▼
            ┌─────────────────┐   owns: session lifecycle, turn ids, mic owner,
            │ SESSION MANAGER │         stop/interrupt, typed+voice shared context
            └────────┬────────┘
                     ▼
            ┌─────────────────┐   tier-0 deterministic router (KEEP):
            │ FAST PATH GATE  │   unambiguous commands ("stop", "apri Spotify")
            └───┬─────────┬───┘   skip the LLM entirely, but still go
       direct   │         │ needs reasoning
       command  │         ▼
                │  ┌─────────────────┐  builds ContextPackage: discourse state,
                │  │ CONTEXT ENGINE  │  entities, TaskState, env state, recent
                │  └────────┬────────┘  turns, summary
                │           ▼
                │  ┌─────────────────┐  relevant durable memory only (k ≤ 5)
                │  │ MEMORY RETRIEVAL│
                │  └────────┬────────┘
                │           ▼
                │  ┌─────────────────┐  deterministic signals → tier + model;
                │  │   AI ROUTER     │  stickiness; escalation; brain-swap pin
                │  └────────┬────────┘
                │           ▼
                │  ┌─────────────────┐  personality + rules; domain-scoped tools
                │  │  PRIMARY BRAIN  │
                │  └────────┬────────┘
                │           ▼
                │  ┌─────────────────┐  plan → act → observe → verify → check
                │  │   AGENT LOOP    │  budgets, ledger, duplicate guard
                │  └────────┬────────┘
                │           ▼
                │   TOOL SELECTION → ARG VALIDATION → PERMISSION LAYER
                │           ▼
                └──►  LOCAL / CLOUD / MCP TOOLS  → EXECUTION → ToolResult
                            ▼
                     VERIFICATION (tool-specific verifier)
                            ▼
                     COMPLETION CHECK (TaskState: all essential steps verified?)
                            ▼
                     HUD / TTS / MEMORY UPDATE / LEDGER / METRICS
```

Direct commands also pass through **permission → execution → verification**. The fast path
skips the LLM, never the safety and honesty layers.

### Cross-cutting systems

| System | Responsibility | Phase |
|---|---|---|
| Diagnostics (`/diag`) | Whitelisted live state of the *running* process | 0 |
| Preflight | End-to-end live checks; N PASS/FAIL/WARN; non-zero exit on fail | 0 |
| Structured metrics | Per-turn latencies, provider, model, steps, failures (no transcripts) | 0 |
| Action ledger | Per-turn/per-task record of tool calls + verification + state change | 3 |
| Provider state | Current tier/model per session, stickiness, escalation reasons | 2 |
| Working memory | `TaskState` per goal | 2–3 |
| Skills | `skills/<name>/SKILL.md` workflow guidance | 6 |
| MCP | External tool servers, env-allowlisted | 5 |
| Screen awareness | Ask-about-screen + local-diff screen watch | 7 |
| Focus sessions | Accountability timer, drift detection, aggregates only | 8 |
| Artifact engine | Structured document creation (txt/md/pdf/docx) | 4 |
| Recovery | Resume paused tasks; supervisor restarts; partial-success reports | 3 |

---

## 4. Components and contracts

Contracts are written in Python-ish pseudotypes. They are **interfaces to converge on**,
mapped onto existing modules during the Phase 0 audit, not new code to drop in.

### 4.1 Session Manager (KEEP + IMPROVE)

Preserves **(brief)**: continuous conversation sessions, wake word, wake gate, single
microphone ownership, stop controls, typed + voice shared context.

Adds:
- A `turn_id` on every event, carried through logs, metrics, ledger and `/diag`.
- `session.discourse` and `session.tasks`, owned by the Context Engine and persisted in SQLite.
- Session boundary policy: idle > `SESSION_IDLE_MINUTES` (default 30) closes the session. Open tasks are **paused**, not dropped, and become resumable ("continuiamo quello di ieri").

### 4.2 Fast Path Gate (KEEP the deterministic router, narrow its scope)

The current deterministic routing logic **(brief)** becomes **tier 0**:

- Handles only **unambiguous, self-contained** commands: stop, volume up/down with an explicit target, open/close a named app, media controls.
- **Must defer to the brain** when the utterance contains a referent ("aprilo", "quello", "l'altro"), a correction ("no,", "non quello"), a conjunction of actions ("cerca … e caricalo"), or anything the lexicon doesn't match with high confidence.
- Context-aware cases ("abbassa un po'" after "apri Spotify") may stay on the fast path if the Context Engine supplies an unambiguous target (see [CONTEXT ENGINE §7 test B](JARVIS_V2_CONTEXT_ENGINE.md#7-acceptance-tests-traced)).

> Rule: the fast path may only answer when it is *sure*. Being wrong costs more than an LLM call.

### 4.3 Context Engine (NEW, core of V2)

Full spec: [JARVIS_V2_CONTEXT_ENGINE.md](JARVIS_V2_CONTEXT_ENGINE.md). Summary:

```
ContextPackage = {
  system:      personality + rules            # stable, prompt-cached
  tools:       schemas for selected domains   # stable per domain set
  profile:     ≤5 durable memory items         # relevant only
  summary:     rolling session summary         # older turns, structured
  task:        TaskState JSON (active goal)    # never summarised away
  discourse:   topic, entities+salience, last_outputs, artifacts, corrections
  environment: foreground app, media state, date/time, mode, screen-watch flag
  recent:      last N turns verbatim
  tool_results: this turn, untrusted-wrapped
  message:     current user message (+ referent candidates)
}
```

### 4.4 AI Router (IMPROVE existing LOCAL/BALANCED/QUALITY; add AGENT)

Full spec: [JARVIS_V2_MODEL_STRATEGY.md](JARVIS_V2_MODEL_STRATEGY.md). Summary:

- Tiers: `T0 deterministic` · `FAST` · `SMART` (default brain) · `PREMIUM`.
- Modes: `LOCAL` · `BALANCED` · `QUALITY` · `AGENT`; each mode sets a floor and a ceiling.
- Selection is a pure function of deterministic signals: task type, complexity, ambiguity, conversation state, tool need, privacy, prior failure, workflow depth. **No extra model call.**
- Stickiness: one provider/model per session/task unless escalation, mode change or provider failure.
- Escalation: counters in `TaskState`/session (misunderstanding ×2, invalid args ×2, loop, same task failed ×2) move the session up one tier.
- Brain swap (pack P08): explicit alias map; unknown version means refuse; runtime-only by default.

### 4.5 Agent loop (NEW)

```
loop (≤ AGENT_MAX_STEPS, ≤ AGENT_MAX_SECONDS):
  1. PLAN     brain emits/updates TaskState.pending_steps (structured)
  2. SELECT   brain picks next tool from domain-scoped set
  3. VALIDATE schema + semantic validators (paths, dates, ids)
              invalid → ToolResult(ok=False, code=INVALID_ARGS, fix_hint) → retry ≤ 2
  4. GUARD    ledger duplicate check (write tools); taint check
  5. PERMIT   permission layer (FORBID > CONFIRM > ALLOW > DEFAULT TIER)
  6. EXECUTE  tool → ToolResult
  7. VERIFY   tool-specific verifier → verified: bool, evidence
  8. RECORD   TaskState.completed_steps / errors / created_files; ledger; metrics
  9. CHECK    all essential steps verified? → done | continue | partial | blocked
final report ← rendered from TaskState (not from model memory)
```

Budgets (defaults; `AGENT` mode raises them):

| Constant | Default | AGENT mode |
|---|---|---|
| `AGENT_MAX_STEPS` | 12 | 25 |
| `AGENT_MAX_SECONDS` | 180 | 600 |
| `AGENT_MAX_ARG_CORRECTIONS` | 2 per step | 2 per step |
| `AGENT_MAX_SAME_TOOL_REPEATS` | 2 | 3 |

Completion semantics:

| Outcome | Condition | What JARVIS says |
|---|---|---|
| `done` | every `essential` step `verified=True` | "Fatto." + evidence (path, Drive link, event time) |
| `partial` | ≥1 essential step failed/unverified, others verified | What succeeded, what failed and why, offer to retry the failed step |
| `blocked` | waiting on permission/clarification | The one question needed |
| `aborted` | budget exceeded / user stop | What was completed, what was not |

**Never say "fatto" if an essential step failed.** This is enforced by the renderer,
which takes `TaskState`, not free text.

### 4.6 Tools (KEEP the ToolResult contract; IMPROVE selection and validation)

Preserves **(brief)**: `ToolResult` contract, execution honesty, verification before
"fatto", no unrestricted raw shell, current computer tools, Calendar, Drive,
app/window lifecycle logic.

Adds a **tool registry** with metadata per tool:

```
ToolSpec = {
  name, description, input_schema (JSON Schema, additionalProperties: false),
  domain: "apps" | "media" | "system" | "files" | "documents" | "web" |
          "calendar" | "drive" | "memory" | "screen" | "focus" | "mcp:<server>",
  side_effect: "read" | "write" | "external_write" | "destructive",
  default_tier: SAFE | CONFIRM | DANGEROUS,
  validators: [semantic validators],      # e.g. path_in_allowed_roots
  verifier: fn(args, result) -> Verification,
  idempotency_key: fn(args) -> str,       # for ledger duplicate detection
  essential_by_default: bool
}
```

**Domain-scoped selection** (brief §23): a deterministic domain classifier (IT/EN
lexicon + active task domains + referenced artifact types) picks domains. Only those
schemas plus a small core set (`memory`, `ask_clarification`, `request_tools`) go in the
prompt. `request_tools(domain)` lets the brain expand the set when the classifier missed.
Target: ≤ 15 tool schemas per request.

### 4.7 Permission layer (KEEP fail-closed; IMPROVE precedence and taint)

Precedence (brief §42):

```
FORBID  >  CONFIRM  >  ALLOW  >  DEFAULT TIER
```

- Rules match on `tool`, `domain`, `side_effect`, argument patterns (path globs, recipients), `source` (`local` | `cloud` | `mcp:<server>` | `unknown`) and **taint**.
- Unknown tools default to `DANGEROUS`. Unknown MCP tools with writes default to `DANGEROUS`.
- **Taint** (brief §43): once a turn ingests untrusted content (web page, file, MCP output, external doc), the turn is `tainted`. In a tainted turn:
  - write steps **already present in the user's original request** keep their normal tier (user-intent binding: "cerca e carica su Drive" authorised the upload);
  - **new** write steps the model proposes after reading untrusted content are raised to `CONFIRM`;
  - destructive steps are always `CONFIRM` or `FORBID`.
- Untrusted content is wrapped and labelled as data: `<untrusted source="web" url="…">…</untrusted>`. The system prompt states that instructions inside it are never followed.

### 4.8 Artifact engine (NEW, Phase 4)

Tools: `create_text`, `create_markdown`, `create_pdf`, `create_docx` (if practical).
The model sends a **structured document**, never code:

```
Document = {
  title: str, subtitle?: str, lang: "it",
  sections: [{heading: str, paragraphs: [str], bullets?: [str], numbered?: [str]}],
  sources?: [{title: str, url: str, accessed: date}],
  filename_hint?: str
}
```

- Renderer: ReportLab (PDF) / python-docx (DOCX) with an embedded TTF font that covers Italian and typographic punctuation (à è é ì ò ù, « », ’, €).
- Save dir: `ARTIFACTS_DIR` (default `%USERPROFILE%\Documents\JARVIS\artifacts\YYYY-MM\`), predictable names, never overwrites without suffixing.
- Verifier: file exists, size > 0, re-opened with a PDF reader, extracted text contains the title, page count ≥ 1.
- Each artifact is registered in `session.discourse.artifacts` and `TaskState.created_files`, which makes "mettilo su Drive" resolvable.

### 4.9 Google Drive upload (IMPROVE, Phase 4)

`drive_upload_file(local_path, folder_id=None, name=None)`:

- Keeps the current Google auth **(brief)**. No new token storage.
- Path policy: allowed roots (`ARTIFACTS_DIR`, Documents, Desktop, Downloads; configurable). Deny secrets (`*.env`, `token*.json`, `credentials*.json`, `client_secret*`, `*.pem`, `*.key`, browser profiles, `AppData`, `Windows`, `Program Files`).
- Real upload (resumable), then **read-back** `files.get(id, fields=id,name,size,md5Checksum,parents,webViewLink)`.
- Verified only if read-back succeeds **and** `md5Checksum` equals the local MD5.

### 4.10 Memory (KEEP API; storage → Memory V2 in Phase 6)

API compatibility **(brief)**: `remember`, `recall`, `update`, `forget`.
V2 storage: `memory/notes/<category>/*.md` (human-readable) + SQLite FTS5 index.
Categories: `preferences`, `projects`, `people`, `workflows`, `captures`, `school`.
Write filter rejects secrets (API keys, tokens, passwords, OAuth material) using pattern and entropy checks.
Lessons from pack P05: a write is not an index (the new note must be retrievable on the next turn), and a capture never fails silently (a failed write is spoken).

### 4.11 Voice (KEEP architecture; IMPROVE STT)

Preserves **(brief)**: wake word, wake gate, single mic ownership, stop controls, supervisor.

STT upgrade (Phase 1), centralised config:

```
STT_MODEL         = "large-v3-turbo"      # candidate; current: "small" (brief)
STT_DEVICE        = "cuda"                 # current: "cpu" (brief)
STT_COMPUTE_TYPE  = "float16"              # alt: "int8_float16"; current: "int8" (brief)
STT_LANGUAGE      = "it"                   # forced when the session language is it
STT_BEAM_SIZE     = 5                      # benchmark 1/2/5 for latency
STT_VAD           = true                   # silero VAD
STT_HINTS_FILE    = "config/stt_hints.txt" # editable domain vocabulary
```

- Domain hints ("Jarvis, Spotify, Rocket League, VS Code, Google Drive, Google Calendar, Claude, ChatGPT") go in a small editable file passed as `initial_prompt`/`hotwords` (verify against the installed faster-whisper version). **Never** inject personal memory into the STT prompt.
- GPU use must be *verified*, not assumed: preflight asserts `device == cuda` on the live model and an RTF below the threshold. The RTX 5060 is Blackwell (sm_120) and needs a CUDA 12.8+ stack. Confirm that the installed CTranslate2 wheel runs on it with a smoke test before relying on it; fallbacks are listed in [MIGRATION §Phase 1](JARVIS_V2_MIGRATION.md#phase-1--stt--voice-reliability).
- Audio capture audit before swapping models: sample rate, channels, clipping, gain, truncation at start/end, silence ratio, endpointing hang time, VAD thresholds. A better model cannot fix broken audio.
- Port from pack P02: a single named **endpoint hang-time** constant (default ~900 ms; tune from field data). Interrupt words bypass it. A **mute flag** sits in the one function that speaks, so test runs never talk.

### 4.12 HUD / Orb (KEEP)

The Three.js orb/HUD **(brief)** stays. V2 adds HUD signals only:
- active tier/model chip (pack P08 label rules: "SONNET 5.5", never a raw slug),
- active workflow step "2/5 — upload Drive",
- permission-wait state,
- **always-visible** screen-watch indicator (pack P06/P14),
- focus session card (Phase 8).

### 4.13 Personality (CENTRALISE)

`personality/` holds one file per variant (`neutral`, `jarvis`, `tutor`, `build`) plus a
shared `rules.md` (honesty, brevity, language). Loaded once as the stable system prefix,
so it is cache-friendly and survives provider swaps.

Desired voice **(brief)**: calm, intelligent, concise, natural Italian, observant, subtle dry humour, not cheesy, not robotic, not support-desk, not constantly sarcastic.
Rules adapted from pack P04: at most one dry line per reply, brevity beats a forced joke, never read on-screen content aloud, and when JARVIS doesn't know something it says so plainly.

### 4.14 Screen awareness (Phase 7)

- **Ask mode:** capture a *fresh* frame natively on Windows at question time. Never answer from a cached frame; if capture fails, say so (pack P06).
- **Watch mode:** a downscaled thumbnail every 5 s, diffed locally. Zero cloud calls on normal changes. A frame goes out only when the user asks or the stillness threshold fires (default 4 min, configurable; the pack itself calls 60 s a demo setting). Visible indicator whenever it is on.
- Frames are never persisted unless `DEBUG_KEEP_FRAMES` is set.

### 4.15 Focus sessions (Phase 8)

Adapted from pack P09–P12 for Windows:

- server-side session with a 1 s tick;
- fresh foreground-window query every tick (Win32, not a cached event);
- browser tab identified by **host hash only**;
- deferred lock: defer, settle, announce;
- drift grace, escalating callouts, snooze, excuse, pause/resume/extend/abort;
- report card;
- **aggregates-only ledger** with a whitelist test and a canary-label privacy test.

### 4.16 MCP (Phase 5)

- Official MCP client SDK; servers declared in config with an explicit `env_allowlist`. Server processes receive **only** those variables.
- MCP tools are registered as `domain = "mcp:<server>"`, `source = mcp`. Unknown write tools default to `DANGEROUS`, and annotations from the server are hints, not grants.
- MCP output is untrusted, so it taints the turn.
- MCP is the preferred mechanism for **new** external apps (Gmail, Notion, GitHub, Spotify advanced, browser automation).

### 4.17 Skills (Phase 6)

`skills/<name>/SKILL.md` with frontmatter `name`, `description`, `triggers`, `domains`,
and a body with workflow guidance. Skills are **not executable**. When triggered, the body
is injected into context and its domains are pre-selected.
Initial set: `school-notes`, `research-pdf`, `organize-day`, `study-session`, `website-build`, `coding-task`.

### 4.18 Build mode (later)

JARVIS orchestrates an **external** coding worker (e.g. Claude Code CLI) inside a declared
workspace directory. It passes a task brief, permission is `CONFIRM`, and it reads back
the results. The conversational model never gets a raw shell.

---

## 5. Diagnostics, preflight, observability

### 5.1 `/diag` (Phase 0)

Localhost-only and read from the **running** process (pack P16). Whitelisted keys only:

```
state, session_id, turn_id, mic_owner, wake_listener{running, last_heartbeat_age_s},
mode, tier, provider, model, sticky_since, last_escalation_reason,
latency_ms{wake, stt, brain_first_token, brain_total, tts_first_audio, turn} (last + p50),
active_tool, workflow{task_id, step, of, status}, permission_wait,
last_recoverable_error{code, at}, screen_watch{on, last_diff_age_s},
focus{on, deferred, settle_ticks}, stt{model, device, compute_type}
```

Never exposed: keys, tokens, raw transcripts, memory contents, window titles, URLs.
A test asserts that the response key set equals the whitelist and that a canary string
injected into a transcript never appears in it.

### 5.2 Preflight (Phase 0, grows forever)

`preflight` runs **live** chains, not mocks (pack P07). `--quick` performs no external writes;
`--full` runs external writes with `TEST_` names and cleans up afterwards.

| # | Check | Mode |
|---|---|---|
| 1 | backend up (supervisor health) | quick |
| 2 | HUD served files == files on disk (hash) | quick |
| 3 | each configured provider: one minimal real call; model id reachable | quick |
| 4 | session: open → text turn → coherent answer → close | quick |
| 5 | STT: bundled Italian WAV → WER ≤ threshold; `device` reported == configured | quick |
| 6 | TTS: synthesise one line → non-empty audio (mute flag on) | quick |
| 7 | memory: remember TEST → recall → forget → recall empty | quick |
| 8 | files: write/read/delete in `ARTIFACTS_DIR` | quick |
| 9 | artifact: create TEST PDF → verifier passes | quick |
| 10 | computer: safe read-only action (foreground window query) | quick |
| 11 | screen: capture one frame → non-blank (if configured) | quick |
| 12 | secrets: config/token files not HTTP-reachable; logs + `/diag` contain no key patterns | quick |
| 13 | Calendar: create TEST event (far future) → read back → delete | full |
| 14 | Drive: upload TEST file → metadata + md5 match → trash | full |
| 15 | intelligence smoke: acceptance conversations A–E (replayed) | full |

Output: one ✓/✗/! line per check, then `N PASS · N FAIL · N WARN`. Exit code ≠ 0 on any FAIL.
From Phase 0 on, **no change is "done" until preflight passes**, and the summary line goes in the commit or PR.

### 5.3 Metrics and logs

- Per-turn JSONL record: `turn_id, ts, wake_ms, rec_ms, stt_ms, brain_ms, ttft_ms, tool_ms[], tts_ms, turn_ms, mode, tier, provider, model, escalated, agent_steps, tool_failures, permission_wait_ms, tokens_in, tokens_out, cache_read`. No transcripts.
- `LOG_LEVEL = NORMAL | DEBUG`. Polling and heartbeat lines are DEBUG only. Rotation is by size (5 MB × 5).
- Repeated stack traces are deduplicated by exception fingerprint: the first is logged in full, then a counter.
- Raw transcripts only under DEBUG, in a separate file with short retention.

---

## 6. Non-negotiables preserved (brief §5)

| System | V2 stance | Verified in source? |
|---|---|---|
| Three.js orb / HUD | KEEP | ⏳ Phase 0 |
| Continuous conversation sessions | KEEP, extend with discourse/task state | ⏳ |
| Wake word architecture + wake gate | KEEP | ⏳ |
| Single microphone ownership | KEEP, invariant: no subsystem may open the mic (pack P13 "ear law") | ⏳ |
| Stop controls | KEEP; stop also aborts the agent loop cleanly | ⏳ |
| Typed + voice shared context | KEEP; Context Engine is modality-agnostic | ⏳ |
| ToolResult contract | KEEP; add `verified`, `evidence`, `code`, `fix_hint` if missing | ⏳ |
| Verification before "fatto" | KEEP; enforce via TaskState renderer | ⏳ |
| Permission layer, fail-closed | KEEP; add precedence + taint | ⏳ |
| No unrestricted raw shell | KEEP | ⏳ |
| Google Calendar / Drive | KEEP; add structured upload | ⏳ |
| Computer tools, app/window lifecycle | KEEP; extend with UIA adapters | ⏳ |
| Deterministic routing | KEEP as tier 0 with narrowed scope | ⏳ |
| Supervisor | KEEP; add preflight hook | ⏳ |
| Memory API (`remember/recall/update/forget`) | KEEP signature; swap storage in Phase 6 | ⏳ |

⏳ means it can't be verified until the source is in the repo.

---

## 7. Provisional subsystem classification

Provisional because it is based on the brief, not the code. Re-grade in Phase 0.

| Subsystem | Class | Rationale |
|---|---|---|
| `voice/` wake, gate, mic ownership, stop | **KEEP** | Listed as working and non-negotiable |
| `voice/` STT | **IMPROVE** | small/CPU/int8 judged unacceptable; GPU turbo candidate |
| `voice/` TTS | **KEEP** (audit) | No complaint in brief; measure latency, consider sentence streaming |
| `brain/` deterministic router | **KEEP → MERGE** | Becomes tier 0 inside the AI Router |
| `brain/` conversational LLM default (local Qwen) | **REPLACE** as default | Weakest model must not be the default brain; stays for LOCAL/FAST |
| `brain/` prompt/history handling | **REPLACE** | Superseded by the Context Engine |
| `tools/` + ToolResult | **KEEP + IMPROVE** | Add registry metadata, validation, verifiers, ledger |
| `permissions/` | **KEEP + IMPROVE** | Precedence + taint + MCP sources |
| `integrations/` Calendar, Drive | **KEEP + IMPROVE** | Structured upload with md5 read-back |
| `orb3d/` | **KEEP** | Add chips/indicators only |
| `launcher/`, supervisor | **KEEP** | Add preflight hook + `/diag` |
| `memory/` | **KEEP API → IMPROVE storage** | Memory V2 in Phase 6 |
| `tests/` | **KEEP + extend** | Add intelligence eval + acceptance suites |
| `docs/` | **KEEP** | V2 docs added alongside |
