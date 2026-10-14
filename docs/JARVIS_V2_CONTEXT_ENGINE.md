# JARVIS V2 — Context Engine

Status: **DRAFT v0.1 — design only.**
Related: [ARCHITECTURE](JARVIS_V2_ARCHITECTURE.md) · [MODEL STRATEGY](JARVIS_V2_MODEL_STRATEGY.md)

The Context Engine is the single biggest lever on "JARVIS feels stupid". A strong model
with a badly built context still feels stupid; a good model with the right context
feels sharp. Its job is to give the brain **exactly what it needs to resolve this turn**:
no less, and not the whole history.

> Evidence caveat: the current prompt/history handling in `brain/` has not been read
> (source not in repo). This doc defines the target; the Phase 0 audit maps it onto
> the existing session and memory code.

---

## 1. Layers of context

| # | Layer | Content | Source | Budget (tokens) | Volatility |
|---|---|---|---|---|---|
| L0 | **System** | personality variant + honesty/brevity/language rules + untrusted-content rule | `personality/` | ~800 | stable (cached) |
| L1 | **Tools** | schemas for the selected domains + core tools | tool registry | ~1,500 | stable per domain set (cached) |
| L2 | **Skill** | active `SKILL.md` body, if one triggered | `skills/` | ≤ 600 | per task |
| L3 | **Profile** | ≤ 5 relevant durable memories (preferences, recurring entities) | memory retrieval | ≤ 300 | per turn |
| L4 | **Session summary** | structured summary of turns older than the verbatim window | summariser | ≤ 400 | every N turns |
| L5 | **Task state** | active `TaskState` JSON (goal, steps, files, errors) | working memory | ≤ 600 | per step |
| L6 | **Discourse state** | active topic, entities + salience, last outputs, artifacts, corrections, open questions | discourse tracker | ≤ 400 | per turn |
| L7 | **Environment** | date/time (Europe/Rome), foreground app, media state, mode, screen-watch on/off | environment probes | ≤ 150 | per turn |
| L8 | **Recent turns** | last `RECENT_TURNS_VERBATIM` (default 8) user/assistant turns | session | ≤ 2,000 | per turn |
| L9 | **Tool results** | this turn's results, truncated, untrusted ones wrapped | agent loop | ≤ 3,000 | per step |
| L10 | **Message** | current user message + **referent candidates** (§4) | input + resolver | — | per turn |

Typical voice turn: **~3–4k tokens**, about 2.3k of which is a cacheable prefix (L0+L1).
Agent step: up to ~10k.

### Ordering for prompt caching

Caches match on an exact prefix, so stable content goes first and volatile content last:

```
[L0 system][L1 tools]  ← cache breakpoint      (byte-stable: no timestamps here)
[L2 skill][L3 profile][L4 summary]  ← optional breakpoint
[messages: L8 recent turns ...]
[final user turn: L7 env + L6 discourse + L5 task + L10 message + candidates]
```

- **Never** put the date/time, turn ids or anything else volatile in L0/L1. They belong in L7, in the final turn.
- Keep tool order deterministic (sorted by name inside each domain) so the same domain set produces the same bytes.
- Caches are per model, which is one more reason for provider stickiness ([MODEL STRATEGY §5](JARVIS_V2_MODEL_STRATEGY.md#5-provider-stickiness)).

---

## 2. Discourse state

Lives in `session.discourse` and is persisted with the session.

```
Discourse = {
  active_topic: str | null,           # "organizzare domani: allenamento + latino"
  entities: [Entity],                  # salience-ranked, max ~20
  last_outputs: [Output],              # last 3 assistant deliverables, structured
  artifacts: [Artifact],               # files/links created this session
  alternatives: AlternativeSet | null, # last set of options offered/considered
  corrections: [Correction],           # "no, preferisco studiare prima"
  open_questions: [str],               # questions JARVIS asked and the user hasn't answered
}

Entity = {
  id, type: app|file|artifact|event|person|place|topic|url|media|window|task,
  label: str, ref: {...type-specific: path / event_id / app_id / url...},
  first_turn, last_turn, mentions: int, role: subject|object|result,
  salience: float                      # recomputed every turn
}

Output = { turn_id, kind: plan|answer|list|summary|artifact|action_report,
           structured: {...} | null, text_digest: str }

AlternativeSet = { turn_id, options: [entity_id], chosen: entity_id | null }

Artifact = { id, kind: pdf|md|txt|docx|drive_file|calendar_event,
             path?: str, drive_id?: str, url?: str, title, created_turn, verified: bool }
```

### Salience

```
salience = 0.5 · recency(turns_since_last_mention, half-life = 3 turns)
         + 0.2 · min(mentions, 5)/5
         + 0.2 · role_weight(result > object > subject)
         + 0.1 · in_active_task
```

The weights are starting values. Tune them with the intelligence eval, never by feel.

### Updating

After every turn:
1. Extract entities from the user message, tool calls/results and the assistant output. Cheap rules come first (tool results are already structured: an app opened, a file created, an event id). Free text only goes through the FAST tier, and only when needed.
2. Register artifacts from `ToolResult`s (they are authoritative).
3. Store the assistant's deliverable as a **structured `Output`** when it has structure (plans, lists, schedules). This is what makes "fai il contrario", "fallo più corto" and "no, preferisco studiare prima" work: the model edits the structure instead of guessing at prose.
4. Detect corrections (`no,`, `non quello`, `intendevo`, `preferisco`, `al contrario`, `sbagliato`) and record them. Two corrections in one task count toward escalation.

---

## 3. Working memory: `TaskState`

Separate from long-term memory. One per goal, persisted in SQLite (`tasks` table,
JSON columns) so it survives restarts and can be resumed.

```
TaskState = {
  task_id, session_id, created_at, updated_at,
  status: active | paused | done | partial | blocked | aborted,
  goal: str,                                # user's goal, in their words + normalised
  constraints: [str],                       # "PDF ordinato", "sul mio Drive", "in italiano"
  plan: [Step],                             # ordered
  current_step: step_id | null,
  completed_steps: [step_id],
  pending_steps: [step_id],
  important_entities: [entity_id],
  created_files: [ {path, kind, size, verified} ],
  source_urls: [ {url, title, used: bool} ],
  tool_outputs: [ {step_id, tool, summary, ok, verified} ],   # summaries, not raw dumps
  errors: [ {step_id, code, message, recoverable} ],
  verification_state: { step_id: {verified: bool, evidence: str} },
  model_trace: [ {step_id, tier, model} ],
  failure_counters: { misunderstanding, invalid_args, tool_selection, loops }
}

Step = { id, description, domain, tool?: str, essential: bool,
         depends_on: [id], expected_output: str, verify: str }
```

Example: "Cerca informazioni sui sommari, crea un PDF ordinato e caricalo sul mio Drive":

```
goal: "ricerca 'sommari' → PDF ordinato → upload su Drive"
plan:
  s1 research     web      essential  verify: ≥3 sources with content
  s2 synthesize   —        essential  verify: Document JSON with ≥3 sections + sources
  s3 create_pdf   documents essential verify: exists, size>0, readable, title in text
  s4 upload       drive    essential  verify: file id + md5 match
state after s3:
  completed_steps=[s1,s2,s3]; created_files=[{path:"…/2026-10/Sommari.pdf", verified:true}]
  pending_steps=[s4]
```

Rules:
- The brain **reads** `TaskState` every step and **proposes** updates. Only the agent loop **writes** it, from verified `ToolResult`s.
- `TaskState` is never summarised away. It is either active, in the context as L5, or paused and retrievable.
- On stop/interrupt: `status = paused`, with the current step recorded.

---

## 4. Referent resolution

Follow-ups that must work when the context makes them resolvable:
"no, quello di prima" · "fai il contrario" · "aprilo" · "continua" · "quello che ti ho detto ieri" ·
"mettilo sul calendario" · "fallo più corto" · "non quello, l'altro".

### Pipeline (deterministic first, then the model decides)

1. **Detect** referring expressions: clitic pronouns (`-lo/-la/-li/-le` on verbs: *aprilo, mettilo, caricalo*), demonstratives (`quello/quella/questo`), `l'altro/l'altra`, `di prima`, `continua`, `il contrario`, `più corto/lungo`, temporal anchors (`ieri`, `prima`, `stamattina`).
2. **Constrain by verb.** The verb's argument type filters candidates: `aprire` → app|file|artifact|url; `mettere sul calendario` → event|plan item|topic with a time; `caricare su Drive` → file|artifact; `abbassare/alzare` → media|volume target; `fare più corto` → last Output.
3. **Rank** the remaining entities by salience.
4. **Special forms:**
   - `l'altro/l'altra`: requires an `AlternativeSet`. With exactly 2 options and one chosen, it means the other one. With > 2 options, ask.
   - `quello di prima`: the previous `Output` or entity of the same type before the current one.
   - `il contrario`: the inverse of the last action or plan (`alza↔abbassa`, `apri↔chiudi`, swapped order in a plan).
   - `continua`: the active task, else the last paused task in this session, else the last `Output` (continue generating).
   - `ieri`/`l'altro giorno`: query persisted sessions/tasks by date (§6).
5. **Inject** the top candidates into L10 as data, so the model resolves with full context:

   ```
   <referents>
   "lo" in "mettilo anche su Drive" → candidates:
     1. artifact:pdf "Sommari.pdf" (created 1 turn ago, verified)   salience 0.91
     2. file:md "appunti.md" (mentioned 6 turns ago)                 salience 0.22
   </referents>
   ```
6. **Clarify only on real ambiguity.** Ask one concise question when the top two candidates are type-compatible and their salience gap is below `REFERENT_MARGIN` (default 0.15), or when no candidate exists. Never ask when the margin is clear.
   Example: *"Quale? Il PDF sui sommari o gli appunti di latino?"*

---

## 5. Context summarisation

Trigger: verbatim turns exceed `RECENT_TURNS_VERBATIM` or the L8 budget.

- Keep the last `RECENT_TURNS_VERBATIM` turns verbatim.
- Summarise older turns into a **structured** summary (the FAST tier is fine; it's extraction, not reasoning):

  ```
  Summary = { topics: [str], decisions: [str], commitments: [str],
              open_questions: [str], entities: [entity_id], artifacts: [artifact_id],
              user_corrections: [str], period: {from_turn, to_turn} }
  ```
- **Pinned and never summarised away:** active/paused `TaskState`, artifacts, unresolved questions, commitments ("ti ricordo alle 18"), and corrections that are still in force.
- Summaries are stored per session. At session close, the final summary becomes the session's resumable digest (§6).

---

## 6. Cross-session continuity ("continuiamo quello di ieri")

At session close, persist `{session_id, date, digest Summary, open/paused task_ids, artifacts}`.

Resolution of "quello di ieri" / "continuiamo":
1. Candidates: paused/active tasks and session digests with `last_active` in the referenced window (`ieri` = previous calendar day in Europe/Rome; no anchor = last 48 h).
2. Exactly one candidate: load its `TaskState` + digest into L4/L5 and resume. JARVIS says what it is resuming in one line: *"Riprendo: PDF sui sommari, manca il caricamento su Drive."*
3. Several candidates: one question listing ≤ 3 options.
4. None: say so plainly and offer to start fresh.

---

## 7. Acceptance tests traced

These are the brief's intelligence tests, traced through the engine. Each one becomes a
scripted conversation in the intelligence eval
([MODEL STRATEGY §8](JARVIS_V2_MODEL_STRATEGY.md#8-benchmark-plan)), with deterministic assertions.

### A. Plan revision

1. *"Domani ho allenamento alle 18 e devo studiare latino. Come mi organizzo?"*
   → SMART tier. The answer is stored as `Output{kind: plan, structured: [{17:00–18:00?…}]}`. Entities: `event:allenamento 18:00`, `topic:latino`.
2. *"No, preferisco studiare prima."*
   → correction detected; referent = the last plan Output; `prima` resolves relative to `allenamento`.
   → **Assert:** the revised plan puts latino *before* 18:00 and keeps the training at 18:00, with no unrelated changes. The fast path never handles this turn.

### B. Media context

1. *"Apri Spotify."* → fast path (unambiguous) → `apps.open(Spotify)` → verify window → L7 `media_app = Spotify`, entity `app:Spotify` salience high.
2. *"abbassa un po'."* → verb `abbassare` → target type volume; L7 has an active media app.
   → **Assert:** Spotify/app volume (or system volume if no per-app control) decreases by `VOLUME_STEP_SMALL` (default 10%), with no clarification question. It may stay on the fast path because the context supplies an unambiguous target.

### C. Artifact reuse

1. *"Cerca questa cosa e fammi un PDF."* → `questa cosa` resolves to the active topic, or a clarification if there is none → agent loop: research → synthesize → `create_pdf` → verify → `Artifact{kind: pdf, verified}`.
2. *"mettilo anche su Drive."* → `-lo` + `caricare su Drive` → candidate filter keeps file/artifact → top candidate is the PDF.
   → **Assert:** `drive_upload_file(path_of_that_pdf)` is called once, verified by md5, and the reply includes the link. No new PDF is generated; the ledger has no duplicate `create_pdf`.

### D. Resume yesterday

*"Continuiamo quello di ieri."*
→ §6. **Assert:** with exactly one paused task from yesterday, it resumes at the first pending step and says what it is resuming. With two, it asks one question listing both. With none, it says so.

### E. The other one

Setup: JARVIS offered two options (e.g. two Calendar slots or two files) and the user picked one.
*"Non quello, l'altro."* → `AlternativeSet` with 2 options and `chosen` = A → B.
→ **Assert:** acts on B without asking. With 3+ options, or no AlternativeSet, it asks exactly one question.

---

## 8. Long-term memory retrieval (bridge to Memory V2)

- Query = current message + active topic + task goal. Retrieve with FTS5 (Phase 6; until then, the existing `recall`).
- Inject ≤ 5 items, ≤ 300 tokens, each tagged with category and age.
- **The current user instruction always overrides stored memory.** This is stated in L0.
- Never store or inject secrets (API keys, tokens, passwords, OAuth material). The write filter rejects them, and the retrieval filter double-checks.
- Durable-memory candidates (repeated corrections, preferences, recurring entities) are *proposed* by the FAST tier at session close. Auto-saved only for low-risk categories (e.g. answer length); otherwise they are confirmed once.

---

## 9. Constants (all in one config block)

| Constant | Default | Notes |
|---|---|---|
| `RECENT_TURNS_VERBATIM` | 8 | tune with the eval |
| `CONTEXT_BUDGET_VOICE` | 4,000 | tokens, excluding tool results |
| `CONTEXT_BUDGET_AGENT` | 12,000 | |
| `PROFILE_MAX_ITEMS` | 5 | |
| `REFERENT_MARGIN` | 0.15 | salience gap below which JARVIS asks |
| `SALIENCE_HALF_LIFE_TURNS` | 3 | |
| `SESSION_IDLE_MINUTES` | 30 | session close → digest |
| `RESUME_WINDOW_HOURS` | 48 | "continuiamo" without a date anchor |
| `VOLUME_STEP_SMALL` | 10 % | "un po'" |
