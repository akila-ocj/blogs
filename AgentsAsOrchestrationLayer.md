# Make the Skirt 8 cm Longer: Agents as the Orchestration Layer

A solo founder types one sentence:

> "Make the skirt 8 cm longer and regenerate the manufacturing pattern."

Here is what actually has to happen for that sentence to produce a DXF file a factory can cut from:

```
User:  "Make the skirt 8 cm longer and regenerate the manufacturing pattern."
                              │
                              ▼
                           Codex                    ← intent, planning, recovery
                              │
                              ▼
                        MCP Server                  ← the contract
                 get_garment()
                 modify_pattern(length_delta=80)
                 run_simulation()
                 validate_collision()
                 export_dxf()
                              │
                              ▼
                         CLO API                    ← the vendor's surface
                              │
                              ▼
                           CLO                      ← the engine that actually knows cloth
```

Most of the excitement about this picture goes to the top box and the bottom box: a model that understands a fashion brief, and a 3D garment simulator that understands how silk drapes. The interesting engineering is in the two boxes in the middle, and almost nobody talks about them.

This post is about those two boxes: what a good tool surface looks like, why `modify_pattern(length_delta=80)` is the right altitude and `click(x=412, y=288)` is not, and what you have to build so that an agent driving professional software produces something you would send to a factory rather than something that merely looks finished.

> **Where this comes from.** Lenny Rachitsky's *How I AI* episode with Yana Welinder ([How a solo founder used Codex and ChatGPT to launch a fashion brand without engineers](https://www.lennysnewsletter.com/), Aug 2026) describes exactly this shape: Yana, a non-engineer, uses Codex to operate CLO — professional 3D fashion design software — and produce CAD files she needs, without having learned CLO herself. Her line about the near-term future is the thesis of this post: *"SaaS is not disappearing; it is gaining a new kind of user."* Her detailed workflow write-ups are at [chatprd.ai/how-i-ai](https://www.chatprd.ai/how-i-ai/workflows-for-an-ai-native-fashion-brand). The diagram above is not a screenshot of her setup — it is the architecture her workflow implies, and the one worth designing for deliberately.

---

## Table of contents

1. [The claim: the agent is the orchestrator, not the engine](#1-the-claim-the-agent-is-the-orchestrator-not-the-engine)
2. [Read the five calls again — the vocabulary *is* the product](#2-read-the-five-calls-again--the-vocabulary-is-the-product)
3. [Seven rules for designing agent-facing tools](#3-seven-rules-for-designing-agent-facing-tools)
4. [The verification loop is the whole thing](#4-the-verification-loop-is-the-whole-thing)
5. [What breaks in production](#5-what-breaks-in-production)
6. [If you sell software: you have a new kind of user](#6-if-you-sell-software-you-have-a-new-kind-of-user)
7. [If you are the solo builder: run both paths](#7-if-you-are-the-solo-builder-run-both-paths)
8. [Where else this pattern is sitting unbuilt](#8-where-else-this-pattern-is-sitting-unbuilt)
9. [When not to do this](#9-when-not-to-do-this)
10. [Checklist](#10-checklist)

---

## 1. The claim: the agent is the orchestrator, not the engine

There is a persistent fantasy that a sufficiently good model will one day emit the DXF directly. Skip CLO. Skip the simulator. Just generate the file.

It will not, and the reason is not model capability. It is that the DXF is downstream of physics. Whether a panel hangs correctly depends on fabric weight, grainline, seam allowance, how the pieces interact when the body inside them moves, and whether two surfaces intersect in a way that is fine in a rendering and impossible in cloth. CLO knows those things because someone spent years encoding them. A language model that "knows" them has memorized what the output usually looks like, which is a different and much worse property when your output goes to a cutting table.

So the division of labour is:

| Layer | Owns | Fails at |
| --- | --- | --- |
| **Agent** | Intent, decomposition, sequencing, retry, interpreting failures, explaining results | Anything requiring ground truth it cannot compute |
| **Tool layer (MCP)** | Vocabulary, validation, units, state handles, error semantics | Nothing, if you design it well — it is glue with opinions |
| **Specialist app** | Physics, domain constraints, file formats, correctness | Knowing what the user meant |

The agent is not replacing CLO. It is replacing the *twelve months of learning CLO* that used to sit between having a design idea and being able to execute it. That is the actual unlock, and it is much bigger than it sounds: in most industries the bottleneck is not the tool's capability, it is the tax of becoming fluent in the tool.

The Ruth Asawa–inspired gown from the episode is the sharpest version of this. It was not impossible to make before. 3D printing existed. What did not exist was a way to get through the enormous volume of CAD work required to prepare it — enough work that the idea was not worth starting. The bottleneck was never creative and never physical. It was the labour cost of translating an idea into a machine-readable form, and that is exactly the cost an orchestration layer removes.

**Every industry has ideas parked behind that same tax.** They do not look like missing capability. They look like "we'd need someone to spend three weeks in the modelling tool for a maybe."

---

## 2. Read the five calls again — the vocabulary *is* the product

```
get_garment()
modify_pattern(length_delta=80)
run_simulation()
validate_collision()
export_dxf()
```

Five calls. Look at what is *not* in that list.

There is no `open_menu()`. No `select_tool("edit")`. No `click(x, y)`. No `wait_for_dialog()`. The tool surface is not a transcription of the GUI. It is a set of verbs at the altitude the user actually thinks in.

That distinction is the single highest-leverage decision in the whole architecture, so it is worth being precise about why.

**A GUI-shaped tool surface makes the agent responsible for the application's internal state machine.** If your tools are clicks and menus, the agent must know that the pattern editor has to be focused before the length field is editable, that the field is in millimetres, that committing requires a different button depending on whether you are in draft mode, and that a modal will appear if the garment has unsaved simulation results. Every one of those facts is a chance to fail, and the agent learns them by trial and error, at your inference cost, in a way that does not generalize to the next version of the UI.

**A domain-shaped tool surface makes the *application* responsible for its own state machine, which is where that responsibility already lived.** `modify_pattern(length_delta=80)` either works or returns an error that says why. The agent's job shrinks to what it is genuinely good at: deciding *what* to do and reacting when it does not work.

Here is the same task expressed both ways.

```python
# Bad: the GUI, transcribed. The agent is now a fragile macro recorder.
focus_window("CLO")
click_menu("Pattern")
click_menu_item("Edit Length")
wait_for_dialog("Length Editor")
select_field("delta")
type_text("80")
click_button("Apply")           # or "OK", depending on version
screenshot()                    # ...and now ask a vision model if it worked
```

```python
# Good: the domain, exposed. The agent states intent; the app enforces reality.
garment = get_garment(id="skirt-v7")
result  = modify_pattern(
    garment_id = garment.id,
    target     = "skirt_panel_group",
    op         = "extend_length",
    delta      = {"value": 80, "unit": "mm"},
    preserve   = ["hem_allowance", "grainline", "notch_positions"],
)
```

The second version is also *auditable*, which matters more than it seems. When the factory asks why the hem allowance changed, there is a call log with an answer. The first version leaves you with a screen recording.

### The screenshot-and-click trap

Computer-use agents that drive a GUI by looking at pixels are genuinely impressive and are the correct answer for exactly one situation: software with no automation surface and no prospect of getting one. For everything else they are the expensive fallback. They are slow (a screenshot round-trip per step), non-deterministic (the same intent produces different click sequences), unauditable (the log is a video), and brittle in a specific ugly way — a UI redesign silently changes behaviour rather than breaking loudly.

If you control the software, or the software has a scripting API, or the vendor has any automation surface at all, spend the day wrapping it. The day pays for itself in the first debugging session.

---

## 3. Seven rules for designing agent-facing tools

These are the rules I would hand someone building the MCP box in that diagram, in rough order of how much pain they save.

### 3.1 Name verbs at the user's altitude, not the implementation's

The test: **could the user have said this sentence?** "Extend the skirt panels by 8 cm" — yes. "Set field `pnl_len_d` on the active pattern node" — no. If the user could not have said it, the tool is at the wrong altitude and the agent will have to invent the mapping every time.

The corollary is that you will have *fewer* tools than you expect. Five good verbs beat forty faithful ones. Every tool you expose is a thing the agent must choose between, and choice is where errors live.

### 3.2 Make units explicit in the schema, never in a comment

Look at the original call again: the user said **8 cm**, the call says **80**. Somewhere in there a unit conversion happened silently, inside a model, based on a convention documented in prose.

That is a whole class of expensive failure, and it is the one that ends with a factory cutting 8 mm instead of 80 mm. Do not accept a bare number:

```jsonc
// Bad — "everything is mm" is a convention, and conventions lose.
{ "length_delta": { "type": "number" } }

// Good — the unit is part of the value, and wrong units fail loudly.
{
  "length_delta": {
    "type": "object",
    "required": ["value", "unit"],
    "properties": {
      "value": { "type": "number" },
      "unit":  { "enum": ["mm", "cm", "in"] }
    }
  }
}
```

If you cannot change the shape, put the unit in the *name*: `length_delta_mm`. A parameter name is read by the model on every single call. A comment in your docs is read once, maybe.

The same logic applies to every ambiguous scalar in your domain: currency (minor units or major?), angles (degrees or radians?), timestamps (UTC or local?), sizes (US or EU?). If two reasonable people could read the number differently, the schema must decide.

### 3.3 Return handles and state, not prose

`get_garment()` is first in the list for a reason. It is how the agent learns the shape of the world before touching it.

Make it return structured state with stable identifiers:

```json
{
  "garment_id": "skirt-v7",
  "revision": 12,
  "units": "mm",
  "panels": [
    { "id": "skirt_front", "length": 620, "grainline": "warp", "seams": ["side_l", "side_r", "waist", "hem"] },
    { "id": "skirt_back",  "length": 620, "grainline": "warp", "seams": ["side_l", "side_r", "waist", "hem"] }
  ],
  "groups": { "skirt_panel_group": ["skirt_front", "skirt_back"] },
  "last_simulation": { "revision": 11, "status": "stale" }
}
```

Two things earn their keep here. **Stable IDs** mean the agent refers to `skirt_front`, not "the second panel", so a reordering does not silently retarget an edit. **`revision`** turns every mutation into an optimistic-concurrency check: `modify_pattern` takes the revision it expects, and a mismatch is a clean error instead of an edit applied to a garment that changed underneath it.

And notice `last_simulation.status: "stale"`. The state tells the agent that the physics is now out of date. That is a hint the agent can act on without you having to explain the dependency in a system prompt.

### 3.4 Separate cheap preview from expensive commit

`run_simulation()` is not free. In this domain it is seconds to minutes of compute; in others it is a build, a render farm job, or a bill.

Give the agent a way to check its work cheaply before spending:

- `modify_pattern(..., dry_run=true)` → returns the diff and any validation errors, changes nothing
- `run_simulation(quality="draft" | "final")` → the agent iterates on draft and commits once on final
- `estimate_cost(op)` → for anything where the agent's choice has a real price

An agent that can look before it leaps behaves dramatically better than one that can only leap and apologise. This is also the cheapest guardrail you will ever ship against runaway loops.

### 3.5 Write errors for a reader who will act on them

An agent reads your error message and immediately decides what to do next. This makes error text a functional interface, not a diagnostic afterthought.

```
Bad:   Error: operation failed (code 0x8007)

Bad:   ValidationError: constraint violated on panel skirt_front

Good:  ExtendLengthRejected: panel 'skirt_front' extended to 700mm but
       hem_allowance is 15mm and the fabric roll width is 1400mm.
       At 700mm the marker no longer nests two panels per width.
       Options: reduce delta to <= 62mm, or set nesting='single' and
       accept ~18% more fabric per unit.
```

The third one contains the diagnosis, the constraint that was violated, the numbers involved, and the exits. An agent given that message fixes the problem in one turn. An agent given the first message retries the identical call, twice, and then tells the user it did not work.

The rule of thumb: **write errors as if the reader is competent, has no access to your source, and must decide in the next five seconds.** That describes an agent, and it also describes a new engineer on your team at 2 a.m., which is why this rule was always good practice.

### 3.6 Make validation a first-class callable, not a side effect

`validate_collision()` is a separate tool, and that is the correct design.

If collision checking only happened implicitly inside `export_dxf()`, the agent would find out about problems at the very end, after the expensive steps, with no way to isolate the cause. As a standalone verb, the agent can call it after every change, bisect which change introduced a problem, and — crucially — call it *before* export as a self-check.

Give it output an agent can reason about: not `false`, but *what* collides, *where*, and by *how much*.

```json
{
  "ok": false,
  "collisions": [
    {
      "between": ["skirt_front", "left_leg_avatar"],
      "max_penetration_mm": 4.2,
      "at_frame": 37,
      "pose": "walk_cycle",
      "severity": "warning"
    }
  ]
}
```

`max_penetration_mm: 4.2` at one frame of a walk cycle is a different decision from a static 40 mm intersection. Boolean validators throw away the information the agent needs to make that call, and the agent will then either ignore real problems or block on trivial ones.

### 3.7 Make every mutation reversible, and say so in the response

Agents take wrong turns. The question is not whether, but how much a wrong turn costs.

Return an undo token from every mutating call, and expose `revert(token)`:

```json
{ "ok": true, "revision": 13, "undo_token": "rev-12->13-a41f", "changed": ["skirt_front.length", "skirt_back.length"] }
```

The `changed` array is worth as much as the undo token. It lets the agent confirm the blast radius matched its intent — and catch the case where a "length" edit also moved notches it promised to preserve.

---

## 4. The verification loop is the whole thing

Strip the diagram down and there are really only two kinds of call in it:

```
mutate:  modify_pattern, export_dxf
verify:  get_garment, run_simulation, validate_collision
```

Three of the five are verification. That ratio is not an accident, and it is the part most people leave out when they build their first version.

An agent's fundamental weakness is that it cannot tell the difference between "I did this correctly" and "I produced something that looks like a correct result." It has no privileged access to truth about the physical world. Left alone it will generate a plausible pattern, describe it confidently, and be wrong in a way no reader can detect from the text.

The simulator fixes this. It is a **ground-truth oracle**: an external, non-negotiable check the agent cannot talk its way past. Cloth either intersects the body or it does not. The panels either close or they do not. And because the oracle is callable, the agent can run the loop itself:

```
propose change → apply → simulate → validate → 
    if fail: read the specific failure, adjust, repeat
    if pass: export
```

This is the same reason coding agents got useful when they got a test runner and a compiler rather than better prose. The model did not suddenly become more truthful. It got an oracle that says no.

**So the design question for any domain is: what is my oracle, and can the agent call it?** If you cannot answer that, you do not have an agent workflow yet — you have a very confident intern with no supervisor. And you should be honest about the gap: an oracle covers what it covers. CLO tells you the cloth does not intersect the avatar. It does not tell you the garment is beautiful, that the seams are sewable by the factory you actually use, or that the pattern grades sensibly to size 18. The episode is refreshingly clear about this: turning designs into accurate sewing patterns is still *the hard unsolved part*. The loop above gets you a defensible draft, not a finished product.

---

## 5. What breaks in production

The happy path in the diagram is five calls. Here is what the fifth attempt looks like.

**Session and state.** `get_garment()` implies something is open. Which document? In whose session? If two agent runs touch the same project, whose revision wins? Decide early whether your tool layer is stateless (every call carries a garment ID and revision) or session-bound (there is an open document and calls act on it). Stateless is more verbose and vastly easier to debug, resume, and run concurrently. Prefer it unless the underlying app makes it impossible.

**Long-running operations.** A final-quality simulation can outlive an agent's tool-call timeout. Do not make the agent sit in a blocking call. Return a job handle immediately and expose polling:

```
run_simulation(quality="final") → { "job_id": "sim-8812", "status": "queued", "eta_seconds": 240 }
get_job(job_id)                 → { "status": "running", "progress": 0.4 }
                                → { "status": "done", "result": {...} }
```

Then let the agent do something useful while it waits — which, as the episode notes, is precisely what makes this work for a solo founder: kick off the long job, go drape actual fabric, come back to the result. The asynchronous shape is not a technical nicety; it is what expands what fits in one person's day.

**Partial failure.** `export_dxf()` succeeds; the write to shared storage fails. Now there is a revision the agent believes is exported and a factory that never received it. Every tool that crosses a boundary needs to be either idempotent (safe to call twice — take a client-supplied idempotency key) or transactional (nothing happened). "Probably fine" is a third option that shows up in incident reviews.

**Silent coercion.** The failure mode that will actually hurt you is not an exception; it is your API accepting `length_delta=80` when the agent meant 8 cm and quietly clamping, rounding, or reinterpreting it. Validate hard at the boundary and reject. An error costs one turn. A silent coercion costs a production run of garments.

**Non-determinism in the log.** The same prompt does not produce the same call sequence twice. If the output goes anywhere consequential, log the *calls*, not the conversation, and treat that log as the artifact of record. Then a bad garment is debuggable: you can replay the exact five calls without the model in the loop, and find out whether the agent was wrong or the tool was.

**Cost and runaway loops.** An agent that can call `run_simulation` in a retry loop can spend real money on compute while it converges. Cap it: a per-run budget, a maximum call count per tool, and a hard stop that returns a specific error the agent understands (`BudgetExceeded: 6 of 6 simulations used; summarize and ask the user`). Make the cap a tool-layer concern, not a prompt instruction — prompts are advisory, code is not.

---

## 6. If you sell software: you have a new kind of user

This is the part to take seriously if you build professional software of any kind.

For twenty years the moat around specialist software was partly capability and substantially **fluency**. CLO, AutoCAD, Ableton, Cadence, ArcGIS, SAP, Avid — each has a real learning cliff, and that cliff produced a professional class whose expertise was partly in the domain and partly in the tool. The cliff protected the vendor: switching cost was measured in retraining.

An agent that can drive the tool competently changes who can buy it. Yana never learned CLO and still produced CAD files with it. That is not a story about CLO being threatened. It is a story about CLO's addressable market expanding to include everyone who has the design problem but never had six months to spend on the interface.

That expansion is not automatic. It goes to whoever ships the tool surface. Concretely:

**Expose a domain API, not a UI transcript.** Everything in section 3. If the only automation path through your product is pixels, the agent-driven experience of your product will be bad, and the comparison will be against a competitor where it is good.

**Make headless a supported mode.** A per-seat licence with an interactive login is a hard blocker for automation. If a run needs a human to click a dialog, it is not a workflow. Decide what an agent seat is, what it costs, and how it authenticates — before a customer asks, because a customer who asks and gets "not supported" has just been told to evaluate alternatives.

**Ship the validators, loudly.** Your differentiation against a model that fabricates plausible output is that you can *prove* things about the result. `validate_collision()` is not a utility function; it is the reason to route through you at all. Expose every check you have as a callable verb.

**Log for audit.** When a customer's agent produces a bad artifact, the first question is whether the tool did what it was told. If you can answer that with a call log, you keep the customer. If the answer is a shrug, you become the suspect by default.

**Price the ceiling, not the seat.** Agents make more calls than humans by orders of magnitude. Metering that maps to your actual cost (simulations, exports, compute-minutes) survives contact with automation. Per-seat pricing quietly encourages your best customers to share a seat with a bot, which is worse for both of you.

The one-line version, from the episode: *SaaS is not disappearing; it is gaining a new kind of user.* The vendors who notice will find the new user is a lot less price-sensitive about the parts they cannot fake.

---

## 7. If you are the solo builder: run both paths

One more thing from the episode is worth extracting, because it is a genuinely good operating principle and it is not about tooling at all.

The hardest open problem in Yana's workflow is turning designs into accurate sewing patterns. Her response is not to bet the brand on Codex solving it, and not to conclude that AI cannot do it. She has **human patternmakers and Codex working the same problem in parallel**, and she lets the results decide.

This looks wasteful and is not, for a specific reason: the cost of running the second path is now small, and the cost of being wrong about which path works is large. When you cannot tell in advance whether the automated path will clear the bar, and the automated path is cheap to attempt, running both and comparing is simply better information than arguing about it. The comparison also gives you something an opinion never does — a concrete measure of the gap, which tells you whether the automated path is one iteration away or three years away.

The engineering version of this: when adopting an agent workflow for something consequential, keep the existing process running alongside it until the outputs agree. Not as a rollback plan — as the evaluation.

---

## 8. Where else this pattern is sitting unbuilt

The shape — intent model → tool contract → specialist engine with an oracle — is not about fashion. It fits anywhere the following three things are true:

1. A specialist application encodes real domain constraints that a language model cannot compute.
2. That application has, or could have, a programmatic surface.
3. There is a check that can say "this is wrong" without a human.

That describes a lot of industry:

| Domain | Engine | Oracle |
| --- | --- | --- |
| Mechanical design | CAD/CAM | FEA, interference checks, DFM rules |
| Electronics | EDA tools | DRC/LVS, timing closure, SPICE |
| Architecture | BIM | Clash detection, code compliance, energy models |
| Audio | DAW | Loudness/true-peak analysis, spectral checks |
| Video | NLE / compositor | Codec conformance, colour-space validation |
| Geospatial | GIS | Topology validation, projection checks |
| Bioinformatics | Lab instruments, pipelines | Assay controls, replicate agreement |
| Finance ops | ERP / ledger | Reconciliation, invariant checks, zero-sum |

In every row, the same two boxes are missing from the middle, and in every row the value is the same: it deletes the fluency tax without deleting the rigour.

The rows where this *fails* are the ones where the third column is weak. If your only validator is a human eye, you can still get an agent to do the work — you just cannot get it to check the work, and you have moved the bottleneck rather than removed it.

---

## 9. When not to do this

Some honest limits, because the pattern is being oversold:

**No oracle, no autonomy.** If you cannot programmatically detect a wrong result, keep the human in the loop on every output, and be realistic that the human will rubber-stamp by week three. Build the check first, then the automation.

**Irreversible physical actions.** The whole chain above ends at a *file*. A human still decides to cut fabric. Keep it that way for anything where the undo is expensive: the agent should produce artifacts and recommendations, and a person should authorise the step that spends material, money, or trust.

**Taste is not delegable.** ChatGPT Images following Yana's sketches closely, rather than generating something impressive-but-generic, is the thing she values — precisely because the point of view is hers. An orchestration layer executes a vision faster. It does not supply one, and the output of a workflow with no point of view is competent and forgettable.

**Regulated outputs need a named human.** In domains where somebody signs, the agent's job is to prepare the submission, not to be the signature. Design the workflow so the reviewable artifact is the deliverable.

**If the task is genuinely one-off, skip the layer.** Building a good tool surface is a real investment. If you will run the workflow twice, drive the GUI by hand. The layer pays off on repetition and on the ability to trust the result — not on novelty.

---

## 10. Checklist

For anyone about to build the middle two boxes:

**Tool surface**
- [ ] Every tool is a verb the user could have said out loud
- [ ] Fewer than ~10 tools; each one distinct enough that choosing is obvious
- [ ] Units are in the schema or the parameter name — never only in prose
- [ ] Objects have stable IDs; mutations carry an expected `revision`
- [ ] Every mutation returns what changed and an undo token

**Verification**
- [ ] There is at least one ground-truth oracle the agent can call
- [ ] Validators return structured detail (what, where, how much), never a bare boolean
- [ ] Validation is callable standalone, not only implicit at export
- [ ] State exposes staleness (`last_simulation: stale`) so the agent knows what to re-run

**Operations**
- [ ] Long jobs return handles and are polled, not blocked on
- [ ] Boundary-crossing calls are idempotent or transactional
- [ ] Invalid input is rejected, never coerced
- [ ] Per-run budget and call caps enforced in code, not in the prompt
- [ ] The call log — not the chat transcript — is the artifact of record and is replayable

**Errors**
- [ ] Each message states the diagnosis, the constraint, the numbers, and the exits
- [ ] No error requires reading your source to act on

---

## Closing

The sentence at the top of this post — "make the skirt 8 cm longer and regenerate the manufacturing pattern" — is the kind of request that used to require a person fluent in a specific expensive application, and therefore used to require either being that person or hiring one.

It does not require the model to know how cloth behaves. It requires the model to know *who does*, and to be able to ask them precisely: five well-named verbs, explicit units, a simulator that will say no, and an error message worth reading.

That middle layer is unglamorous, and it is where the leverage is. The models will keep getting better on their own. The contracts will not build themselves.

---

### Sources

- Lenny's Newsletter — *How I AI*: [How a solo founder used Codex and ChatGPT to launch a fashion brand without engineers](https://www.lennysnewsletter.com/) with Yana Welinder (August 2026)
- [Workflows for an AI-Native Fashion Brand](https://www.chatprd.ai/how-i-ai/workflows-for-an-ai-native-fashion-brand)
- [How to Use AI for Fashion Design and Visualization](https://www.chatprd.ai/how-i-ai/workflows/how-to-use-ai-for-fashion-design-and-visualization)
- [How to Prototype Complex Garments Using AI and 3D Modeling](https://www.chatprd.ai/how-i-ai/workflows/how-to-prototype-complex-garments-using-ai-and-3d-modeling)
- [How to Build and Run an E-Commerce Business with AI as a Technical Co-Founder](https://www.chatprd.ai/how-i-ai/workflows/how-to-build-and-run-an-e-commerce-business-with-ai-as-a-technical-co-founder)
