# Career Positioning Agent — Generic
## Technical Notes and Architecture

This document captures the architecture, design decisions, and rules built into `CareerAgent_Generic.html`. Intended for anyone maintaining or extending the agent.

---

## What the agent is

A self-contained HTML file that runs inside Claude.ai as an artifact. It makes direct calls to the Anthropic API from the browser. No server, no installation, no API key needed by the user — Claude handles authentication.

Users paste in their career materials and a job description. The agent analyzes fit, asks clarifying questions if needed, then generates tailored career documents.

---

## How to run it

1. Download `CareerAgent_Generic.html`
2. Go to [claude.ai](https://claude.ai) and start a new conversation
3. Attach the file via the paperclip icon
4. The agent UI renders as an interactive artifact

---

## Models used

| Model | Used for |
|---|---|
| `claude-haiku-4-5` | Analysis, scoring, fact inventory extraction |
| `claude-sonnet-4-6` | All resume and document generation |

---

## Outputs

| Output key | Label | Token budget |
|---|---|---|
| `tailoredResume` | Tailored Resume | 3,500 |
| `whatToChange` | What to Change | — |
| `masterResume` | Master Resume | — |
| `linkedin` | LinkedIn Updates | — |
| `execBio` | Executive Bio | — |
| `pitchDeck` | Pitch Deck | — |

---

## Input fields

| Field | `S` state key | Used in |
|---|---|---|
| About Me | `S.about` | All prompts via ctxBlock |
| Master Resume | `S.resume` | All prompts via ctxBlock |
| LinkedIn Profile | `S.linkedin` | LinkedIn prompt, ctxBlock (non-targeted mode) |
| Target Role (JD) | `S.role` | Analysis, all targeted prompts |
| Master Pitch Deck | `S.masterPitch` | ctxBlock (targeted mode only) |
| Pitch Positioning | `S.pitchPositioning` | Pitch deck prompt |
| Corrections / Context | `S.answers` | ctxBlock (last position) |

---

## Architecture

### State object (`S`)
All session state lives in a single object:
```js
{
  mode, level, type,
  role, about, resume, linkedin,
  masterPitch, pitchPositioning,
  analysis, answers, askedQuestions,
  factInventory,
  outputs
}
```

### Prompt cache (`promptCache()`)
Built once per generation run. Caches: `ctx` (ctxBlock output), `voice` (level+type framing), `order` (section ordering), `execMode`, `positioning`, `jdContext` (derived analysis fields), `lvl`, `typ`.

Invalidated by `invalidateCache()`, which also clears `_ctxCache`.

### ctxBlock memoisation
`ctxBlock()` is memoised in `_ctxCache`. Rebuilt only when `invalidateCache()` is called. Prevents redundant rebuilds across the multiple prompts that call it per generation cycle.

### Prompt caching (added 2026-07-17 — do not revert)
**Problem it fixed:** a full run fires 5-8 API calls (fact inventory,
analysis, one call per selected output, changelog). Before this change,
every call independently re-sent the full candidate materials block
(`ctxBlock()` — About Me, resume, LinkedIn, JD, corrections) and the
`STYLE` writing rules inline in the user prompt. None of it was cached
server-side, so a single run could burn tens of thousands of input tokens
even though the same ~7k-token block was being resent 5+ times unchanged.
On a Claude Pro plan this was enough to exhaust the usage window after only
a few runs — a real problem for anyone you share this agent with, since
their usage counts against their own plan.

**How it works now:**
- `cachedMaterials()` (defined just above the Fact Inventory section) wraps
  `ctxBlock()`'s output in a system content block:
  `{ type: 'text', text: ctxBlock(), cache_control: { type: 'ephemeral' } }`.
- `callOutput()` sends `system: [cachedMaterials(), {type:'text', text: STYLE}]`
  — materials first, STYLE second. Every output call in a run
  (Master Resume, Tailored Resume, What to Change, LinkedIn, Exec Bio,
  Pitch Deck) sends this identical prefix, so the 2nd through 5th+ calls in
  the run hit the cache instead of paying full input-token price again.
- `callAnalysis()` sends `system: [cachedMaterials(), {json-instruction}]` —
  same materials block, so analysis/re-analysis share the cache with the
  output calls when run back to back.
- Every `*Prompt()` function that used to interpolate `${c.ctx}` / `${ctx}`
  directly (`masterResumePrompt`, `tailoredResumePrompt`,
  `whatToChangePrompt`, `linkedinPrompt`, `execBioPrompt`,
  `pitchDeckPrompt`, `analysisPrompt`, `reanalysisPrompt`) now has a one-line
  placeholder note instead — materials arrive via the system block. Do not
  re-add `${c.ctx}` to any of these; it would duplicate materials in both
  the system and user turns and silently defeat the cache (the request
  would still work, just cost full price again).
- Cache is ephemeral, ~5 minute TTL. It only helps within one run or
  quick back-to-back runs — it will not reduce cost for someone who runs
  the agent once, closes the tab, and returns later. That's expected
  behavior, not a bug to fix.
- Corrections (`S.answers`) are part of `ctxBlock()`, so submitting a
  correction changes the cached block and correctly forces a fresh cache
  write on the next run. Don't try to "optimize" this — it needs to
  invalidate when corrections change, or stale corrections would get
  served from cache.

**If extending this agent further:** any content that should be reused
across the parallel output calls belongs in the `system` array passed to
`callAPI()`, marked with `cache_control` on the block after which caching
should apply — not interpolated into the individual `*Prompt()` return
strings.

### Fact inventory (`buildFactInventory()`)


Runs once before `runOutputs()` when both About Me and Resume are provided. Uses Haiku to extract per-role structured JSON:
```json
{
  "roles": [{"company","title","dates","facts","metrics","technologies","scope"}],
  "unattributed_facts": [],
  "education": [],
  "conflicts": []
}
```
All output prompts use inventory-based context when available — eliminates cross-role synthesis errors and saves tokens vs raw documents.

**Invalidation:** Only cleared when About Me or Resume text changes (detected in `collectInputs()`). Adding corrections via `S.answers` does NOT rebuild the inventory — corrections travel via the prompt context, not the inventory.

**Uses `callJSON()`** — a generic JSON caller that does parse + repair without assuming any specific schema. Does NOT use `callAnalysis()` (which is shaped for the analysis response).

### Context ordering (critical for correction reliability)
Corrections (`S.answers`) are positioned LAST in ctxBlock, after the JD. This ensures the model encounters them as the most recent context before generation begins. Middle-context instructions are deprioritised by the model in long contexts.

Corrections are wrapped with strong delimiters:
```
=== USER CORRECTIONS & CLARIFICATIONS — THESE OVERRIDE EVERYTHING ABOVE ===
[content]
=== END CORRECTIONS ===
```

### Execution strategy — parallel vs sequential
- Starts in parallel mode (`_useSequential = false`) — optimistic for paid accounts
- First 429 rate limit → `_useSequential = true` for the session
- `_runParallel()` fires all tasks via `Promise.allSettled()`
- `_runSequential()` runs one task at a time with retry backoff
- Sequential retry uses label-based DOM lookup (not index) to update correct progress indicators when running as a subset of the original task list
- Both functions collect non-abort errors into `taskErrors[]` for surface to the error panel

### JSON parse resilience (`callAnalysis`)
4-stage repair chain:
1. Strip fences, direct parse
2. Extract outermost `{}`, parse
3. Structural repairs: curly quotes → straight, unescaped newlines, trailing commas, single-quoted keys
4. Field-by-field regex extraction — recovers individual fields when JSON is too broken to parse as a whole

Partial results (`_partial: true`) set `canProceed: false` — the user must confirm before generation proceeds, since a partial analysis means `positioningStrategy`, `evidenceMap`, and `jdContext` will be incomplete.

---

## Token optimisation

| Optimisation | Saving |
|---|---|
| `ctxBlock()` memoised | Eliminates 2–3 redundant rebuilds per run |
| `ACCURACY_RULES` subset for `whatToChange` | ~440 tokens vs full TAILORING_RULES |
| `jdContext` omits `roleAnalysis` (already in ctxBlock) | ~180 tokens per tailored call |
| About Me truncated to 10,000 chars | Saves ~500 tokens vs 12,000 limit |
| Role JD truncated to 4,000 chars | Saves ~500 tokens |
| About Me in fact inventory: 8,000 chars | Saves ~1,000 tokens vs 12,000 |
| Fact inventory not rebuilt on correction-only regen | Saves one full Haiku call |
| **Net saving per tailored resume run** | **~2,200–3,000 tokens** |

---

## Analysis prompt — intent-based framework

The analysis prompt explicitly instructs the model NOT to do surface-level keyword matching. It classifies JD requirements by intent:

**Step 1 — Role intent:** Classify every requirement as true blocker / strong signal / nice to have / aspirational-keyword-heavy. Identify role type.

**Step 2 — Candidate evidence:** Map to intent, not keywords. Identify adjacency strength.

**Step 3 — Calibration rules:**
- "Disqualifying gap" only if candidate is clearly unrealistic
- Group related gaps, penalise once (e.g. ML+LLM+GPU = one AI domain gap)
- Separate exact domain match from operating model fit
- Label adjacency: strong / moderate / weak / not transferable

**Step 4 — Scoring:** Cold + referral-adjusted for each score

**Step 5 — Gap classification (5 labels):**
- `disqualifyingGaps` — do not apply
- `seriousDomainGaps` — referral or strong adjacency can overcome
- `interviewRisks` — prepare deeply
- `positioningGaps` — fixable through tailoring
- `minorGaps` — not material

**Step 6 — Final recommendation:**
- `finalRecommendation`: Strong apply / Apply with tailoring / Apply only with referral / Low-probability reach / Do not apply
- `fitClassification`: core fit / adjacent fit / stretch fit / poor fit

### Analysis outputs injected into tailoring
`promptCache()` builds `jdContext` from analysis results, injecting them into the tailoring prompt:
- `positioningAngle.leadWith` / `.compress` / `.cut`
- `recruiterView`, `hiringManagerView`
- `winningAngles`
- Evidence map (JD requirement → candidate proof point)
- `seriousDomainGaps`, `positioningGaps`, `interviewRisks`

This ensures the deep JD understanding from analysis is actively used during generation, not just summarised in one line.

---

## Tailored resume prompt — key rules

### JD-first reading
The prompt instructs the model to start from the JD and ask what from the resume answers it — not start from the resume and find matches. Every JD requirement is classified as true blocker / strong signal / nice to have / aspirational before any writing begins.

### Seniority-level bullet framing
Per level:
- **Junior/mid:** Individual contribution, growth signals (promotions, expanded scope), no scope inflation
- **Senior IC:** Influence without authority, ownership and judgment, "why this person over another senior" answerable from bullets alone
- **Manager/lead:** Team outcomes (not individual contributions), technical credibility, delivery through others
- **Director/head:** Function transformation, exec stakeholder relationships, headcount/budget where accurate
- **VP/exec:** Largest defensible scale, board visibility, skip tool lists

### People leadership placement
Determined by reading the JD for direct-reports language:
- IC role → people leadership last
- Leadership/management role → people leadership 2nd or 3rd
- Ambiguous → second half of bullets

### Tailoring decisions display
The tailored resume output uses delimiters:
```
=== TAILORING DECISIONS ===
[3–5 bullet reasoning block]
=== RESUME ===
[full resume]
```
The renderer splits on these delimiters and shows the reasoning in a note-box above the resume. The `<pre>` box and copy button contain only clean resume text.

---

## Writing rules (STYLE constant — system prompt on all output calls)

- Matter-of-fact, grounded voice. Confident through specificity.
- Good verbs: led, built, coordinated, partnered, designed, improved, reduced, supported, delivered, aligned, drove, defined, introduced, shaped, scaled
- Banned: spearheaded, leveraged, synergy, robust, best-in-class, passionate about, results-driven, streamlined (without specifics), visionary, transformational, guru, rockstar, elite, thought leader
- No semicolons in bullet body text — split into two bullets
- No three consecutive sentences starting with the same word
- Implied first person throughout — never mix with explicit first person
- Vary sentence length deliberately — cadence uniformity is the primary AI tell
- 11 AI writing pattern checks: tricolon overuse, abstraction, smoothness, vocabulary fingerprints, setup-payoff openers, colon-as-reveal, etc.

---

## Error handling

| Error type | Handling |
|---|---|
| API auth error | `friendlyError()` shows auth message |
| Rate limit (429) | Switches to sequential mode, shows retry message |
| JSON parse failure | 4-stage repair chain, graceful `_partial` fallback |
| Corrupt stored profile | Distinguishes `SyntaxError` from storage unavailable, shows named notice |
| Empty source on regen | `regenOutputs()` validates before running |
| Task errors in parallel | Collected in `taskErrors[]`, surfaced to error panel |
| Abort vs real failure | `wasAborted()` captured before `_abortCtrl = null` — prevents false "nothing produced" error |

---

## Storage

Key: `career-agent-profile`
Saves: `about`, `resume`, `linkedin`, `answers`, `savedAt`
Auto-loads on `initAgent()`

**Clear cache button** (always visible in header) calls `clearProfile()`, which:
1. Deletes the storage key
2. Calls `resetAll()` — full UI and state reset
3. Shows a green confirmation banner in the inputs phase for 6 seconds

---

## UI phases

1. **Inputs** — mode pills, materials fields, JD, output checkboxes
2. **Analysis** — scores with rationale, evidence map, positioning angle, gap analysis, recruiter/HM view, recommendation
3. **Clarifications gate** — Q&A before generation (shown when fit < 70 and material questions remain)
4. **Outputs** — tabs per output, copy buttons, regen section with corrections field

### Persistent header buttons
- **💾 Save Profile** — saves materials to storage
- **+ Save Answers → About Me** — appends answers to About Me, saves, clears `S.answers`
- **🗑 Clear cache** — full wipe with confirmation banner

---

## Known limitations

- Runs inside Claude.ai artifact sandbox — `localStorage` and `sessionStorage` are not available; uses `window.storage` (Claude's artifact storage API)
- No file upload — all materials must be pasted as text
- Token limits: About Me capped at 10,000 chars, resume at 8,000, JD at 4,000 — very long documents will be truncated
- Free tier: parallel generation not available (rate limits trigger sequential fallback immediately)

---

## Files in this repo

| File | Description |
|---|---|
| `CareerAgent_Generic.html` | The agent — self-contained, runs in Claude.ai |
| `README.md` | User-facing instructions |
| `CareerAgent_Generic_Notes.md` | This file — architecture and technical notes |
