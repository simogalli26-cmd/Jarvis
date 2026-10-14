# JARVIS V2 — Feature Matrix

Status: **DRAFT v0.1.**
Reference: *"Build Your Own Jarvis — The GPT-6 Astra Prompt Pack"* (16 prompts, Sept 2026), read in full.
The pack is a **design reference, not the new architecture.** It isn't committed here (third-party
content); this doc summarises ideas in our own words.

> "Current JARVIS" cells come from the V2 brief **(brief)** because the source isn't in the repo yet.
> Re-check each one in the Phase 0 audit.

Classification key for reference features:
**A** already better in our JARVIS · **B** adapt idea only · **C** port feature · **D** rebuild our feature using this idea · **E** do not use

---

## 1. What the pack is

A browser-first, single-user build for macOS + Chrome:
- standard-library Python server on :4700;
- 3d-force-graph "galaxy" of markdown notes;
- Web Speech API for STT and TTS;
- one OpenAI model as the brain (keyword-overlap retrieval over notes).

Part Two adds focus sessions, a webcam, screen watch, brain swap and field instrumentation.

**Its real value is its laws, not its stack:**
- "done" means preflight passed;
- build truth endpoints first;
- an honest error beats a helpful guess;
- name things out loud but store nothing;
- a wrong lock is worse than no lock;
- never answer from a stale frame;
- secrets are never browser-reachable.

Our JARVIS **(brief)** is already ahead on everything structural: wake word, native mic
ownership, permissioned tool execution, verified results, Calendar/Drive, Windows computer control, supervisor.

---

## 2. Prompt-by-prompt audit

| # | Prompt | Core idea | Architectural principle | UX benefit | Security / privacy | Impl. quality | Overlap with ours | Class |
|---|---|---|---|---|---|---|---|---|
| 01 | Galaxy & Brain | 3D note graph + `/chat` answering only from notes | Secrets in project root, never in served dir; "verify before saying done" | Visible knowledge | Good: key never browser-reachable | Demo-grade: stdlib server, keyword-overlap retrieval, notes-only brain | Orb/HUD + sessions **(brief)** | **E** for graph/brain/server; **B** for the secrets-placement law (becomes a preflight check) |
| 02 | Voice | Web Speech in/out; buffered finalisation | **One named endpointing constant**; interrupt words bypass it; **mute flag inside the one speak function** | No mid-sentence cut-offs; silent test tabs | Browser STT sends audio to the vendor | Browser-only, Chrome-only | Native wake/STT/TTS **(brief)** | **E** for Web Speech; **C** port the mute flag; **B** adapt hang-time constant + interrupt bypass to our VAD |
| 03 | Demo moment | Fly camera to the source note | **Don't lie about provenance** (no single node for a multi-source answer); small talk doesn't move the UI | Trust in answers | — | Good rule, galaxy-specific | Orb has no note graph | **B**: show sources/artifacts for research answers; chit-chat never triggers visual actions. DEFER to Phase 9 |
| 04 | Character | Butler persona in one block; boot greeting with real counts | **Persona centralised in one place**; brevity beats forced jokes; greetings use real data, never hardcoded | Personality that's easy to change | — | Good | Personality not centralised **(brief §32)** | **D**: rebuild our personality as `personality/` variants |
| 05 | Total Recall | "remember that" → markdown file + live index | **A write is not an index** (retrievable next turn); **capture never fails silently** | Grows memory by voice | Notes in plain files | Simple, sound | Memory API exists **(brief)** | **D**: Memory V2 = markdown notes + FTS index, keeping our API |
| 06 | Sight | Screen share → frame at question time → vision model | **Fresh frame at ask time, never cached**; say so when the share ended; media type matches encoding; **visible indicator** | Can answer about the screen | Frames go to the cloud only on ask | Browser `getDisplayMedia` | Not present **(brief §34)** | **B**: same laws, native Windows capture (Phase 7) |
| 07 | Preflight | Live end-to-end checks, N pass/fail/warn, exit ≠ 0 | **"Done" = preflight passed**; live chains not mocks; served files == disk; config not reachable | Reliability | Explicit secrets check | Excellent idea | Not present **(brief §30)** | **C**: port, adapted to our stack. Highest value in the pack |
| 08 | Brain Swap | Voice model swap via alias map | **Explicit alias dict; unknown version → refuse; runtime-only swap** | Try models live | — | Good | Routing modes exist **(brief)** | **C**: port into the AI Router (our providers, not necessarily OpenRouter) |
| 09 | Focus Sessions | Server-side timer; drift detection; callouts; report | **Fresh foreground read every tick** (not cached events); **host-hash only**; client state = whitelisted booleans/counters; constants named at top | Accountability | Strong privacy by structure | macOS-specific (bundle id, AppleScript) | Not present | **B**: re-implement for Windows in Phase 8, after core intelligence |
| 10 | Deferred Lock | Don't lock the tab you clicked from; settle first | **Defer → settle → announce**; never guess from background windows | No wrong locks | — | Good | — | **B** (Phase 8) |
| 11 | Re-target | "lock on this tab" + card button | **Overlays must not trust "frontmost"**: read the thing you care about | Changing the target mid-session | — | Good | Relevant to our HUD: when the HUD has focus, "foreground app" is JARVIS itself | **B**: apply the overlay rule to computer tools now; focus parts in Phase 8 |
| 12 | Name the Distraction | Speak the distraction's name, store it nowhere | **Label lives one tick**; **canary-label test** proves it's absent from state, report and notes | Memorable callouts | Excellent privacy pattern | Good | — | **B**: the canary test technique is adopted now for `/diag` and log privacy tests; callouts in Phase 8 (optional) |
| 13 | Camera | Local pose landmarks; phone/slouch nudges | **"Ear law": no organ may turn on the mic**; local-only frames | Posture nudges | Webcam = high privacy cost | Browser MediaPipe | Single mic ownership **(brief)** already enforces the ear law | **A** for the ear law; **E** (defer indefinitely) for the camera feature |
| 14 | Screen Watch | 5 s thumbnail diff locally; stillness → one frame → nudge | **Zero cloud cost until it needs to think**; visible "watching" indicator; stillness threshold = personality (60 s is a demo setting) | Proactive help when stuck | Frames only on trigger | Good | Not present | **C**: port with native capture (Phase 7) |
| 15 | Personality swap lines | Curated lines on brain swap | **One function for every swap path**; pretty names, never raw slugs; pinned aliases | Polish | — | Fine | — | **B** (Phase 9 polish) |
| 16 | Field Proof | `/diag`, debug overlay, whitelisted ledger, probe page | **Read diag from the running server**; whitelist tests; "when fixes change nothing, the input is lying" | Debuggability | Whitelists prevent leaks | Excellent | Not present **(brief §31)** | **C**: port `/diag` + whitelist tests (Phase 0) |

### What must not be copied

- Browser Web Speech API as primary STT/TTS (P02): worse accuracy, vendor-bound, Chrome-only.
- Browser-only mic lifecycle (P02, P06): breaks our wake word + single mic owner.
- Standard-library-only server (P01): fine for a demo, weaker than our supervisor/backend.
- API keys in any browser-accessible config (P01 avoids this; we must keep avoiding it).
- A notes-only brain with keyword-overlap retrieval (P01): a regression from tool-using assistant to Q&A box.
- Fresh-folder architecture ("paste Prompt 01 in an empty folder"): V2 is an evolution.
- Simplistic browser-only assistant patterns, or anything weaker than our tool/permission/verification system.
- Webcam posture monitoring (P13), for now: high privacy cost, low priority.
- macOS mechanics (bundle ids, AppleScript, menu-bar card) as-is: the ideas port, the code doesn't.

---

## 3. Decision matrix

| Feature | Current JARVIS (brief) | Reference pack | V2 decision | Why |
|---|---|---|---|---|
| **Wake** | Wake word + wake gate + supervisor | None (click mic) | **KEEP** | Ours is strictly better; hands-free is core |
| **STT** | faster-whisper `small`, CPU, int8; quality unacceptable | Web Speech API | **IMPROVE**: benchmark `large-v3-turbo` on CUDA (fp16 / int8_float16), forced `it`, domain hints, audio-capture audit | Accuracy is the first link of "understanding"; the pack's option is worse |
| **TTS** | Exists (details ⏳) | Web Speech, British voice | **KEEP + audit**; add sentence-streaming from the LLM stream; port the mute flag | Latency to first audio matters most |
| **Session** | Continuous sessions; typed+voice shared | Server-side short history | **KEEP + IMPROVE**: discourse state, `TaskState`, session digests, resume | Continuity is priority #2 |
| **UI** | Three.js orb / HUD | 3D force-graph galaxy | **KEEP orb**; add model chip, workflow step, permission wait, screen-watch indicator | No reason to replace a working UI |
| **Personality** | Not centralised | One persona block + variants | **REBUILD (D)** as `personality/` variants: neutral, jarvis, tutor, build | Survives provider swaps; one place to tune |
| **Memory** | API remember/recall/update/forget | Markdown captures + live index | **KEEP API, REBUILD storage (D)**: `memory/notes/*.md` + SQLite FTS5 (Phase 6) | Human-readable, durable, searchable |
| **Context** | Prose history **(likely)** | Short history only | **NEW** Context Engine (layers, entities, referents, summaries) | Biggest lever on "feels stupid" |
| **Brain** | Local Qwen as default **(brief §8)** | One cloud model | **REPLACE default** with the SMART cloud tier; Qwen stays for LOCAL/offline | Weakest model must not be the conversational default |
| **Router** | Deterministic + LOCAL/BALANCED/QUALITY | None (brain swap only) | **IMPROVE**: tiers T0/FAST/SMART/PREMIUM, + AGENT mode, deterministic signals, stickiness, escalation, brain swap with refusal | Right model per request, zero routing cost |
| **Agent loop** | Single-step tool use **(likely)** | None | **NEW**: plan/act/verify/check with budgets, ledger, partial-success reports | Complete goals, not commands |
| **Tools** | Structured tools + ToolResult + verification | Endpoints only | **KEEP + IMPROVE**: registry metadata, domain-scoped selection, arg validation, verifiers, idempotency keys | Fewer tool confusions, honest results |
| **Permissions** | Fail-closed layer | None | **KEEP + IMPROVE**: FORBID > CONFIRM > ALLOW > DEFAULT, taint from untrusted content, MCP sources | Safety for agent + MCP era |
| **PC control** | Structured Windows tools, app/window lifecycle | macOS frontmost via CLI | **KEEP + EXTEND**: UIA/pywinauto adapters, verified controls, vision fallback last; overlay rule (P11) | Structured beats pixel clicking |
| **Calendar** | Google Calendar integration | None | **KEEP**; add plan→events, idempotent create via ledger | Works; duplicates are the risk |
| **Drive** | Google Drive integration | None | **IMPROVE**: `drive_upload_file` with path policy + md5 read-back verification | Needed for the multi-step acceptance test |
| **Web** | ⏳ (research capability unknown) | None (author's own runs elsewhere) | **AUDIT → IMPROVE**: search + fetch tools, untrusted-wrapped, sources tracked in `TaskState` | Research step of §29 |
| **Documents** | ⏳ | None | **NEW** artifact engine: `create_text/markdown/pdf/docx` from structured JSON | No arbitrary code execution |
| **PDF** | ⏳ | None | **NEW**: ReportLab + Italian-capable TTF; verifier (exists, size, readable, title) | Acceptance test §29 |
| **Screen** | Not present | P06 ask + P14 watch | **PORT (C)** with native capture, fresh frames, visible indicator, local diff | Big UX win, low cloud cost |
| **Focus** | Not present | P09–P12 | **ADAPT (B)** for Windows, Phase 8 | Valuable but after intelligence |
| **MCP** | Not present | None | **NEW** (Phase 5): official client, env allowlist, unknown writes DANGEROUS, output untrusted | Preferred path for new integrations |
| **Skills** | Not present | None (paid community mentions skills) | **NEW** (Phase 6): `skills/<name>/SKILL.md`, non-executable | Portable workflow guidance |
| **Preflight** | Not present | P07 | **PORT (C)** in Phase 0; grows by one check per incident | "Done" = preflight passed |
| **Diagnostics** | Not present | P16 | **PORT (C)** in Phase 0: `/diag` whitelist from the running process | Instrument before fixing |
| **Observability** | ⏳ | Tick timings advice | **NEW**: per-turn metrics JSONL, no transcripts | Tune from field numbers |
| **Logs** | Polling spam / unrotated **(brief §45)** | — | **IMPROVE**: NORMAL/DEBUG, rotation, trace dedupe | Signal over noise |
| **Brain swap** | ⏳ | P08 + P15 | **PORT (C)** with explicit aliases + refusal | Honest model control |
| **Camera** | Not present | P13 | **DO NOT BUILD** (for now); keep the ear-law invariant | Privacy cost > value today |
| **School mode** | ⏳ | None | **NEW** skill + artifact flow (transcript → notes → PDF) | High-value daily use |
| **Build mode** | ⏳ | Pack *is* built by a coding agent | **LATER**: external coding worker in a workspace; no raw shell for the brain | Keeps the security boundary |
