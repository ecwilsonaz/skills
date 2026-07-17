---
name: panel
description: Use when the user wants a strategic, judgment, or factual question pressure-tested from several independent perspectives — advisory panel, "run it by different lenses", go/no-go and tradeoff decisions, strategy calls, "what am I missing", or a fact-heavy question where one answer isn't enough. For non-code questions; use code-review-panel for reviewing code.
---

# Advisor Panel

You are convening a **panel of advisors** to pressure-test a strategic or factual
question from several independent perspectives. This is the non-code sibling of the
`code-review-panel` skill and shares its philosophy: a **hybrid** design where the
**main conversation owns the human decision gates** (seeding plan, panel roster, which
cruxes to dig into, whether to run another round) and a **lean Workflow owns the machine
work between them** (parallel fan-out, optional adversarial cross-examination). You — the
orchestrator running this conversation — do the persona *design* and the final
*synthesis* yourself; the Workflow just runs the fan-out and keeps the raw persona dumps
out of this context so you get the signal, not five essays.

Unlike a code review, perspectives here are **not commensurable** — you do not "dedup"
a historian's take against a quant's, and "is this take real?" has no truth value. The
deliverable is a **convergence/divergence map**: where independent lenses *agree* (trust
it) versus where they *split* (that's the real decision), plus what each uniquely
surfaced and a bottom-line recommendation.

## The one knob — persona model default

```
PERSONA_MODEL = opus
```

This is the model each persona sub-agent runs on by default. Edit this line to your
taste: `opus` (deepest, default), `sonnet` (fast/cheap, fine for generating a stance),
or `fable`. It is overridden per-invocation by the model directives below.

**Orchestration** (persona design + synthesis) runs on *this* session's model — a skill
cannot reassign the main-loop model. This skill is at its best driven by a strong
reasoning model (**Fable or Opus**). If you are running under a lighter model, say so
once: synthesis quality benefits from Fable/Opus, but do not hard-block.

## User Input

```text
$ARGUMENTS
```

`$ARGUMENTS` carries the **question** plus optional directives:

| Directive | Effect |
|-----------|--------|
| `whole hog` / `spread across models` / `matrix` | **Matrix path**: assign personas to *different* local models (Opus, Sonnet, Fable, Haiku) for model-diversity on top of stance-diversity — see the matrix path under Phase 1. |
| `on opus` / `personas on sonnet` / `agents: fable` | **Uniform override**: run *all* personas on the named model this invocation. |
| `--debate` / "have them argue it out" | Add the adversarial cross-examination round (Phase 2b). |
| `--research` / `--deep` | Force a research seeding pass (`--deep` = the heavy multi-source pass). |
| `--grill` | Force a clarification pass before designing the panel. |
| `--auto` | Skip the roster-confirmation gate; design and fan out without asking. |
| `--save` | Save the final brief without asking. |

**Model resolution — first match wins:** matrix directive → uniform directive →
`PERSONA_MODEL`. Absence of any directive means `PERSONA_MODEL`; do not ask.

If `$ARGUMENTS` is empty, ask the user for the question before proceeding.

---

## Phase 0 — Seeding (triaged; you propose, the user confirms)

Read the question and decide what grounding the panel needs *before* you design it. Do
NOT reflexively do both research and grilling — triage, then propose a short seeding plan
and let the user trim it. A crisp judgment call may need nothing; a fact-laden or
time-sensitive question may need current numbers; an under-specified question may need
sharpening. Announce your plan in one or two lines ("This is fact-sensitive — I'll do a
quick research pass on X, and I have two clarifying questions") and proceed once
confirmed. `--auto` skips confirmation; `--grill`/`--research`/`--deep` force a component
on.

### Clarification (bundled — do not depend on any external skill)

When the question is under-specified or the *real* decision is unclear, grill the user
first. The method:

> Interview the user about the question until you understand what they actually need to
> decide and what a useful answer looks like. Walk down the branches one at a time —
> **one question per turn**, never a barrage — and for each, offer your recommended
> answer so they can just confirm. Facts you can look up, look up; the *decisions* are
> theirs. Stop as soon as the question is sharp enough to design a panel around; this is
> seeding, not the whole engagement.

Capture the resulting sharpened question and any constraints — they go into every persona
briefing.

### Research (self-contained — assume no other research skill exists)

When the answer depends on facts the personas shouldn't invent (current figures, recent
events, how something actually works, prior art), run a research pass yourself with
`WebSearch` and `WebFetch` and distill the findings into a **fact pack** that seeds every
persona.

- **Light pass (default):** 3–6 targeted `WebSearch` queries, `WebFetch` the few best
  sources, extract the concrete facts (with dates and source URLs), and write a tight
  fact pack (a dozen bullets, each traceable to a source). Note what you *couldn't*
  verify — personas must know the edges of the evidence.
- **Deep pass (`--deep`, or when you judge the question genuinely research-grade):** widen
  to 10+ sources across independent origins, corroborate each load-bearing claim against
  at least two, prefer primary/authoritative sources over aggregators, timestamp
  everything, and separate established fact from contested claim in the pack. If the user
  has a heavier research skill (e.g. `deep-research`) installed you may invoke it, but do
  not assume it — the above is sufficient on its own.

The fact pack is shared context handed to **every** persona (prepended to each briefing),
so the panel argues over the same facts and diverges on *judgment*, not on trivia.

---

## Phase 1 — Design the panel (dynamic; propose, then confirm)

Design **3–6 personas tailored to THIS question** — not a fixed roster. The right lenses
differ per question: a hiring call summons different advisors than a pricing change or a
"is this claim true" question. Aim for lenses that will genuinely *disagree* — a panel
that agrees by construction is theater. Useful archetypes to draw from (adapt, don't just
copy): the Skeptic (attacks the premise), the Steelman Advocate (strongest case *for*),
the Contrarian (the unpopular read), a domain expert or two specific to the question
(quant/financial, legal/regulatory, technical, historical-precedent), the
Stakeholder/End-User voice, the Second-Order-Effects thinker, the Base-Rates/outside-view
forecaster.

For each persona write: a **short label**, a **one-line stance description**, and the
**mandate** it will be given (what to optimize for, what to attack, what evidence to
bring). Distinctiveness is the whole game — if two personas would say the same thing,
merge or replace one.

**Present the slate** to the user as a compact list (label — one-liner each) and let them
swap, add, drop, or reweight before fan-out. This is a decision gate, same as the code
panel's scope selection. `--auto` skips it.

Under the **matrix path**, also assign a `model` to each persona here — spread the local
models (`opus` / `sonnet` / `fable` / `haiku`) across them so different lenses run on
different models. Otherwise every persona uses the resolved uniform/default model.
(TODO — not yet wired: OpenRouter/external-hosted persona models. The fan-out runs each
persona through the Workflow `agent()`, whose `model` option only accepts local Claude
tiers; adding OpenRouter would mean porting `code-review-panel`'s `opencode` shell-out
wrapper. Until then, do NOT put an OpenRouter id in a persona's `model` — it would fail
to run and the lens would silently drop.)

---

## Phase 2 — Fan out (Workflow)

Invoke the **Workflow** tool with the script below. Pass `args` as an **actual JSON
object**, not a JSON-encoded string — the `personas` array especially must arrive as a
real array. The user invoking this skill IS the multi-agent opt-in. Tell them they can
watch live progress with `/workflows`. If a persona agent dies, relaunch with
`resumeFromRunId` — survivors return from cache; a single lost persona is also fine to
just drop (losing one lens is recoverable, unlike losing a code reviewer).

**Error returns.** The script returns `{ error, perspectives: [], panelSize: 0 }` on any
of three failures — never proceed to Phase 3 synthesis on an `error` (or on empty
`perspectives`); report it and recover by cause:

- **args mangled** (delivered as a string, unparseable, or `personas` missing — a known
  harness bug, observed in `code-review-panel`): bake the values into the script as a
  `const CFG = {…}` literal (replacing the `try/catch` normalization block) and relaunch
  via `scriptPath`, exactly as `code-review-panel` does.
- **all personas died** (transient model outage — the `error` says so): relaunch with
  `resumeFromRunId` (survivors return from cache), or fall back to plain Agent calls.

A *single* lost persona is not an error — it is dropped and the panel continues (losing
one lens is recoverable, unlike losing a code reviewer).

The Workflow returns **structured perspectives** (compact, schema-shaped) plus, if
`--debate` was set, each persona's **cross-examination** of the others. It does NOT write
the synthesis — you do that in Phase 3, under the strong orchestrator model, in
conversation.

```js
export const meta = {
  name: 'advisor-panel',
  description: 'Independent advisor personas reason about a question in parallel; optional adversarial cross-examination. Synthesis happens in the main conversation.',
  phases: [
    { title: 'Deliberate', detail: 'each persona reasons independently from its assigned stance' },
    { title: 'Cross-examine', detail: 'optional: each persona critiques the others (only when debate is on)' },
  ],
}

// Light structure — enough to synthesize cleanly, NOT a heavy findings schema.
const PERSPECTIVE_SCHEMA = {
  type: 'object',
  required: ['stance', 'position', 'key_points', 'strongest_argument', 'what_others_miss', 'confidence'],
  properties: {
    stance: { type: 'string', description: 'this persona label' },
    position: { type: 'string', description: 'bottom-line answer/recommendation from this lens, 1-2 sentences' },
    key_points: { type: 'array', items: { type: 'string' }, description: 'the reasoning, each point standalone' },
    strongest_argument: { type: 'string', description: 'the single most compelling point this lens brings' },
    what_others_miss: { type: 'string', description: 'the blind spot in other lenses this persona is positioned to see' },
    key_facts_relied_on: { type: 'array', items: { type: 'string' }, description: 'load-bearing facts (cite the fact pack or a source); flag anything assumed rather than known' },
    open_questions: { type: 'array', items: { type: 'string' }, description: 'what this lens would need to know to be more confident' },
    confidence: { enum: ['high', 'medium', 'low'], description: 'this lens\'s confidence in its own position' },
  },
}

const CRITIQUE_SCHEMA = {
  type: 'object',
  required: ['stance', 'critiques'],
  properties: {
    stance: { type: 'string' },
    critiques: {
      type: 'array',
      items: {
        type: 'object',
        required: ['target', 'assessment', 'verdict'],
        properties: {
          target: { type: 'string', description: 'which other persona this critiques' },
          assessment: { type: 'string', description: 'where their reasoning is strong or weak, and why' },
          verdict: { enum: ['strengthens', 'undercuts', 'refines', 'neutral'], description: 'net effect on that position after scrutiny' },
        },
      },
    },
    updated_position: { type: 'string', description: 'this persona\'s position after seeing the others — note if it moved and why' },
  },
}

// Defensive arg normalization — the harness sometimes delivers `args` as a JSON
// STRING rather than an object (observed in code-review-panel too); parse it so
// CFG.personas et al. resolve instead of crashing on `undefined.map`.
let CFG
try {
  CFG = typeof args === 'string' ? JSON.parse(args) : (args ?? {})
} catch {
  return { error: 'args arrived as an unparseable string — pass args as a real JSON object, or bake a `const CFG = {…}` literal into the script and relaunch via scriptPath.', perspectives: [], critiques: [], panelSize: 0 }
}
if (!Array.isArray(CFG.personas) || CFG.personas.length === 0) {
  return { error: `No personas delivered to the workflow (args arrived as ${typeof args}). The orchestrator must pass args.personas as a non-empty array.`, perspectives: [], critiques: [], panelSize: 0 }
}

const COMMON = `Question: ${CFG.question}
${CFG.factPack ? `\nShared fact pack (argue from THESE facts; flag anything you assume beyond them):\n${CFG.factPack}\n` : ''}${CFG.constraints ? `\nConstraints / what a useful answer must address: ${CFG.constraints}\n` : ''}${CFG.settled ? `\nSETTLED — do NOT rehash these; treat them as resolved and reason forward from them:\n${CFG.settled}\n` : ''}${CFG.priorSynthesis ? `\nWhere the panel stands after prior rounds (react to this; add, don't repeat):\n${CFG.priorSynthesis}\n` : ''}
Reason rigorously and independently FROM YOUR ASSIGNED STANCE — do not hedge toward a balanced center; that is the orchestrator's job, not yours. Bring the argument only your lens is positioned to make. Ground claims in the fact pack where one is given and mark speculation as speculation. It is fine to conclude the question is under-specified — say what you'd need.`

phase('Deliberate')
const PERSONAS = CFG.personas  // [{ id, label, mandate, model? }]
const perspectives = (await parallel(PERSONAS.map((p) => () =>
  agent(
    `You are the "${p.label}" advisor on a panel. Your mandate: ${p.mandate}\n\n${COMMON}`,
    { label: `panel:${p.id}`, phase: 'Deliberate', model: p.model || CFG.personaModel, schema: PERSPECTIVE_SCHEMA }
  ).then((out) => ({ id: p.id, out })).catch(() => null)
))).filter(Boolean)

if (perspectives.length === 0) return { error: 'All personas failed to report — relaunch with resumeFromRunId, or fall back to Agent calls.', perspectives: [], critiques: [], panelSize: 0 }
log(`${perspectives.length}/${PERSONAS.length} personas reported`)

let critiques = []
// Gate on >= 2 survivors: a lone persona has no one to cross-examine.
if (CFG.debate && perspectives.length >= 2) {
  phase('Cross-examine')
  // Each persona sees the OTHERS' positions and critiques them — self excluded, so
  // no one is handed its own take as an "other advisor". Fresh context: it gets the
  // others' structured perspectives, not a running chat.
  const byId = Object.fromEntries(perspectives.map((r) => [r.id, r]))
  const digestFor = (selfId) => perspectives.filter((r) => r.id !== selfId).map((r) => `### ${r.out.stance}\nPosition: ${r.out.position}\nStrongest: ${r.out.strongest_argument}\nKey points: ${r.out.key_points.join('; ')}`).join('\n\n')
  critiques = (await parallel(PERSONAS.filter((p) => byId[p.id]).map((p) => () =>
    agent(
      `You are the "${p.label}" advisor. Below are the OTHER advisors' positions on the same question. Critique them from your lens: where is each strong, where does the reasoning break, what are they missing? Then state whether your own position moves after reading them. Be adversarial but fair — the goal is to find what survives scrutiny.\n\nQuestion: ${CFG.question}\n\nOther advisors:\n${digestFor(p.id)}`,
      { label: `debate:${p.id}`, phase: 'Cross-examine', model: p.model || CFG.personaModel, schema: CRITIQUE_SCHEMA }
    ).then((out) => out).catch(() => null)
  ))).filter(Boolean)
}

return { perspectives: perspectives.map((r) => r.out), critiques, panelSize: perspectives.length }
```

`args` shape:

```json
{
  "question": "<the sharpened question>",
  "factPack": "<research fact pack, or ''>",
  "constraints": "<what a useful answer must address, or ''>",
  "settled": "<SETTLED questions from prior rounds, or '' — only when a round has settled ground to protect>",
  "priorSynthesis": "<running synthesis from prior rounds, or '' on round 1>",
  "personas": [ { "id": "skeptic", "label": "The Skeptic", "mandate": "...", "model": "opus" } ],
  "personaModel": "opus",
  "debate": false
}
```

`personaModel` is the resolved uniform/default model; per-persona `model` (matrix path
only) overrides it. Set `debate: true` when `--debate` was requested or when you expect
the perspectives to sharply conflict (offer it to the user if you didn't run it).

---

## Phase 3 — Synthesize and present (you, in conversation)

Read the returned perspectives (and critiques) and write a tight **decision brief**
directly in the conversation. Structure:

1. **Bottom line up front** — your synthesized recommendation in 2–4 sentences, with a
   confidence read. This is *your* judgment as orchestrator, informed by the panel — not
   a vote tally.
2. **Convergence** — where independent lenses agreed. Agreement across genuinely different
   stances is the strongest signal in the whole exercise; name it and say why it's
   trustworthy.
3. **Divergence — the real cruxes** — where they split, framed as the actual decisions
   *you* now face. For each crux: who's on which side, what would tip it, and (if
   `--debate` ran) what survived cross-examination. This is usually the most valuable
   section.
4. **Unique contributions** — the one thing each persona surfaced that no other did
   (`what_others_miss`), so no lens's distinctive value is lost in averaging.
5. **What the panel is unsure about** — open questions / low-confidence areas, and what
   would resolve them (this seeds any next round or further research).

Keep raw persona essays OUT — you have the structured perspectives; quote a persona
verbatim only when a specific line earns it. Lead with the brief; do not bury it under
process narration.

Then **converse** — do not force a triage queue. Offer the natural next moves and let the
user pick:
- dig into a specific crux (you can quote that persona's full reasoning),
- run the adversarial `--debate` round if it wasn't run and perspectives conflict,
- run **another round** (Phase 4),
- save the brief (Phase 5),
- or just talk it through.

If you're running under a lighter model and the synthesis was hard, say so and suggest
re-running the synthesis under Fable/Opus.

---

## Phase 4 — Another round (multi-round loop)

Strategic questions often need several rounds. Before each new round, **recommend a panel
adjustment** and let the user decide:

- **Reuse** the same personas (they react to new input — your reaction, a follow-up, a
  decision that crystallized) — best when the lenses are right but the *input* changed.
- **Supplement** — keep the panel and add lenses a gap revealed (e.g. "no one covered
  regulatory risk; add that lens").
- **Swap / retire** — replace personas hitting diminishing returns, or drop a lens that
  turned out irrelevant. Call diminishing returns honestly: if a persona added nothing
  new, say so and cut it.

Every round is a **fresh-context fan-out** — never persist living persona agents;
independence is the point and living agents drift toward anchoring and false consensus.
"Reuse the panel" means *same persona definitions, fresh instances*. You preserve
continuity by curating the **briefing packet** in `args`:

- `priorSynthesis` — where the panel landed, so this round adds rather than repeats.
- `settled` — a **SETTLED — do not rehash** list, included **only when the round has
  genuinely resolved ground to protect**. Use it to stop fresh instances from
  re-litigating closed cruxes; omit it when everything is still open.
- new input from the user or new research folded into `question`/`factPack`/`constraints`.

Then re-invoke the Workflow (Phase 2) and re-synthesize (Phase 3).

---

## Phase 5 — Save (opt-in)

Default to keeping everything in-conversation — many panels are one-off. When the session
did real work (multiple rounds, research, a decision you'll revisit) **offer** to save a
durable brief (`--save` skips the ask). This is a global skill that runs in arbitrary
directories, often non-repos — do **not** assume a git repo or `.claude/reviews/`. Save
to:

```
~/.claude/panels/<short-kebab-slug>-<YYYY-MM-DD>.md
```

`mkdir -p ~/.claude/panels` first (get the date from `date +%F` — do not hardcode it).
Write a human-readable archive: the question, the fact pack (with sources), the final
roster, and the synthesis brief, plus a one-line-per-round trail if it was multi-round.
Unlike the code panel, this is an archive to return to, not a control input for future
runs — there is no prior-disposition machinery to maintain. Tell the user the path.

---

## Fallback (Workflow tool unavailable)

Run the personas as parallel `Agent` tool calls in a single message (each with its
briefing and the resolved `model`), collecting their structured perspectives; if
`--debate`, do a second parallel round handing each the others' positions. Then synthesize
and present exactly as in Phase 3. You lose live `/workflows` progress, cheap resume, and
the context-isolation of raw dumps — so be disciplined about not letting five full essays
flood the conversation; ask each agent for the structured shape above.
