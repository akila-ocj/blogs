# SaaS Is Gaining a New Kind of User

A solo founder types one sentence:

> "Make the skirt 8 cm longer and regenerate the manufacturing pattern."

Here is what has to happen for that sentence to produce a DXF file a factory can cut from:

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

That is a real workflow. Yana Welinder, a non-engineer building an AI-native fashion brand solo, uses Codex to operate CLO — professional 3D fashion design software — and produce CAD files she needs, without ever having learned CLO herself.

It is a good story. But the fashion part is the least interesting thing about it, and CLO is not really the point. The point is a sentence Yana says almost in passing:

> **"SaaS is not disappearing; it is gaining a new kind of user."**

That is the most important idea in the episode, and it is a claim about software architecture, not about fashion. This post is about what it actually means, and what you should do differently on Monday if you are building software today.

> **Source.** Lenny Rachitsky's *How I AI* episode with Yana Welinder, [How a solo founder used Codex and ChatGPT to launch a fashion brand without engineers](https://www.lennysnewsletter.com/) (August 2026). Her detailed workflow write-ups are at [chatprd.ai/how-i-ai](https://www.chatprd.ai/how-i-ai/workflows-for-an-ai-native-fashion-brand). The diagram above is not a screenshot of her stack — it is the architecture her workflow implies, and the one worth designing for deliberately.

---

## Table of contents

1. [The stack is growing a layer](#1-the-stack-is-growing-a-layer)
2. [Why the API path wins](#2-why-the-api-path-wins)
3. [So if you are building vertical SaaS today](#3-so-if-you-are-building-vertical-saas-today)
4. [The trap: your API is CRUD, your product is workflows](#4-the-trap-your-api-is-crud-your-product-is-workflows)
5. [Designing the domain API: seven rules](#5-designing-the-domain-api-seven-rules)
6. [The oracle: why three of CLO's five calls are verification](#6-the-oracle-why-three-of-clos-five-calls-are-verification)
7. [The three edges: CLI, MCP, webhooks](#7-the-three-edges-cli-mcp-webhooks)
8. [What breaks in production](#8-what-breaks-in-production)
9. [The commercial consequences](#9-the-commercial-consequences)
10. [When computer-use is still the right answer](#10-when-computer-use-is-still-the-right-answer)
11. [Checklist](#11-checklist)

---

## 1. The stack is growing a layer

For thirty years, software has been designed for one shape:

```
Human
  ↓
GUI
  ↓
Software
```

Everything follows from that shape. The GUI is where the product lives. It is where workflow logic accumulates, where validation happens, where the "are you sure?" dialog protects you, and where the value of the product is demonstrated in a demo. The API, if there is one, is a side door for integrations — built later, by a smaller team, with less care.

That shape is now growing a layer. Two variants are emerging:

```
Human                      Human
  ↓                          ↓
AI agent                   AI agent
  ↓                          ↓
MCP / API / CLI            GUI computer-use
  ↓                          ↓
Software                   Software
```

Both are real, both work today, and they are not equally good. The left path — the agent talking to a contract — is the one that gets dramatically more reliable over time. The right path — the agent driving your buttons by looking at pixels — is the fallback, and it stays a fallback.

The important structural change is not that a box got added. It is **where the human's intent gets translated into operations.** It used to happen in a human's head, expressed through a GUI that the vendor designed. Now it happens in a model, expressed through whatever surface the vendor exposes. If the only surface you expose is a GUI, you have decided that every agent interacting with your product must do so through the least reliable channel available.

The CLO example is just the first industry where this became visible, because 3D garment simulation happens to have a hard learning curve and a scriptable engine underneath. The pattern is not fashion-specific. It is what happens to every category of software with real domain depth.

---

## 2. Why the API path wins

"Likely to become much more reliable" deserves an argument, not an assertion. Here is the argument, in the order that matters.

**A GUI is a lossy encoding of the operations underneath it.** Your interface takes `extend_panel_length(panels, delta)` and encodes it as: focus a window, open a menu, wait for a modal, click a field, type digits, press a button whose label changed in v9. A computer-use agent has to *decode that back* into the operation, through pixels, every single time. The API path skips the encode/decode round trip entirely. You cannot out-engineer the fact that one path does strictly more work to arrive at the same place.

**Failures are loud on one side and silent on the other.** A renamed API field breaks the call — you get an error, in CI, before a customer does. A moved button does not break anything; it just makes the agent click the wrong thing and continue confidently. Silent misbehaviour is the expensive failure mode, and the GUI path is structurally prone to it.

**Determinism and replay.** The same intent through an API produces the same five calls, which you can log, diff, and replay without the model in the loop. The same intent through computer-use produces a different click sequence each run, and your audit artifact is a video. When a customer asks "why did the system do this?", one of those answers keeps the customer.

**Cost and latency.** Every computer-use step is a screenshot, a vision pass, and a decision. A domain call is a few hundred tokens. For a five-step task the difference is an annoyance; for the hundred-step workflows agents actually get asked to do, it is the difference between viable and not.

**Permissions.** This one is underrated and will eventually become the deciding factor. A computer-use agent inherits *whatever the logged-in human can do* — full session, full blast radius, no scoping. An API-driven agent can hold a token scoped to three verbs, rate-limited, with every call attributed to it in your audit log. The first arrangement is one you will have to explain to a security review. The second one is one you can pass.

**Concurrency and testability.** Ten agents can hold ten API sessions. Ten agents cannot share one desktop. And a tool contract can be tested in CI — you can assert that `modify_pattern` rejects bad units — whereas testing that an agent can still find the Apply button is a much sadder job.

| | Agent → API/MCP/CLI | Agent → computer-use |
| --- | --- | --- |
| Reliability ceiling | Bounded by contract quality | Bounded by perception |
| Failure mode | Loud (error) | Silent (wrong click) |
| Auditability | Call log, replayable | Screen recording |
| Cost per step | Low | Screenshot + vision pass |
| Permissions | Scopable per agent | Whatever the human has |
| Concurrency | N sessions | One desktop |
| Breaks when | You change the contract | You change the CSS |

None of this means computer-use is bad. It means it is the *fallback*, correct in the specific situation described in [section 10](#10-when-computer-use-is-still-the-right-answer). If you control the software, or you can get an automation surface, the choice is not close.

---

## 3. So if you are building vertical SaaS today

Here is how I would architect it:

```
                 Human users
                     │
                  Web UI
                     │
                     ▼
            ┌─────────────────┐
            │   Domain API    │
            │                 │
            │ customers       │
            │ assets          │
            │ workflows       │
            │ documents       │
            │ billing         │
            │ reporting       │
            └────────┬────────┘
                     │
         ┌───────────┼───────────┐
         ▼           ▼           ▼
        CLI         MCP       Webhooks
         │           │           │
       Codex       Agents    automation
```

Look at what moved. **The domain API is the product.** The web UI is a client of it — the first client, the one humans use, and no longer the privileged one. The CLI, the MCP server, and the webhooks are three more clients, all thin, all speaking the same vocabulary.

This is not a new idea. It is the old "API-first" argument, and plenty of teams nodded at it and then shipped a UI with the business logic in the controllers anyway, because nothing forced the issue. The thing that changed is that **something now forces the issue.** An agent cannot use logic that only exists in your React components. If your workflow rules live in the UI, your product is invisible to the fastest-growing category of user you will have.

Three properties make this architecture worth the discipline:

**One implementation of every rule.** If `approve_invoice` enforces the two-signature policy inside the domain API, then the web UI, the CLI, the agent, and the webhook retry all enforce it. If the policy lives in the UI's submit handler, three of those four paths are a compliance incident waiting to be discovered.

**Uniform audit.** Every mutation goes through one place, so "who changed this, through what surface, with what arguments" has one answer, whether the actor was a person, a cron job, or Codex acting for a person.

**The edges get cheap.** Once the domain API exists, an MCP server is a few hundred lines of schema. A CLI is an argument parser. Adding the next surface — whatever protocol replaces MCP in three years — is an afternoon, not a rewrite. That is the real hedge here: nobody knows which agent protocol wins, and if your domain layer is clean you do not need to.

The compressed version: **build the middle box first, and treat every interface as a rendering of it.**

---

## 4. The trap: your API is CRUD, your product is workflows

This is the failure I would expect most teams to walk into, so it is worth naming precisely.

Most vertical SaaS already has an API. It is usually a CRUD reflection of the database — `GET /invoices`, `POST /invoices`, `PATCH /invoices/{id}` — built for integrations, generated from models, and technically complete. Teams look at it and conclude the agent story is handled.

It is not handled, because the CRUD API exposes your *tables*, and your product is your *workflows*. The knowledge of what "approving an invoice" means — which fields change together, which policy applies, which side effects fire, what makes it invalid — lives in a controller behind the UI, or worse, in the UI itself.

Hand an agent the CRUD API and you have asked it to reimplement your business logic from the outside:

```python
# What the agent has to do against a CRUD API.
# Every line is an opportunity to get your domain wrong.
inv = GET("/invoices/8812")
GET("/approval_policies?org=42")            # ...which policy applies here?
GET("/users/me/permissions")                # ...am I even allowed?
PATCH("/invoices/8812", {"status": "approved",
                         "approved_by": "u_17",
                         "approved_at": "2026-08-24T09:14:00Z"})
POST("/ledger_entries", {...})              # did the UI also do this? probably?
POST("/notifications", {...})               # and this?
```

It will work in testing and be wrong in production, in ways that surface a quarter later during reconciliation. And note the last two lines: the agent is *guessing* at your side effects. Anything the UI's submit handler did that the agent does not know about simply will not happen.

The domain version:

```python
approve_invoice(
    invoice_id      = "8812",
    approver        = "u_17",
    note            = "matches PO 4471",
    idempotency_key = "codex-run-9f2a",
)
# → policy check, ledger entry, notification, audit record — all of it, once,
#   the same way the web UI does it, because it is the same code path.
```

One call. Your rules, your side effects, your audit trail. The agent cannot get the policy wrong because the agent is not implementing the policy.

The test for whether you have a domain API or a CRUD API: **pick the three things your customers actually do in your product, and see whether each is one call.** If "approve an invoice", "onboard a customer", "close a period" each require the caller to orchestrate six writes in the right order, you have a database with HTTP in front of it. That is a fine integration surface and a poor agent surface.

This is exactly what the CLO example gets right, and why those five calls are worth staring at. `modify_pattern(length_delta=80)` is a domain verb. The CRUD-shaped version of that API would expose pattern nodes and let the caller mutate vertex coordinates — technically more powerful, and useless to anyone who is not already a CLO expert. Which is the entire population you are trying to serve.

---

## 5. Designing the domain API: seven rules

These apply whether the box in the middle is wrapping CLO or backing your own vertical SaaS.

### 5.1 Name verbs at the user's altitude

The test: **could a competent user have said this sentence out loud?** "Extend the skirt panels by 8 cm" — yes. "Set field `pnl_len_d` on the active pattern node" — no. "Approve invoice 8812" — yes. "Update invoice status and insert two ledger rows" — no.

You will end up with *fewer* verbs than you expect, and that is the goal. Five good ones beat forty faithful ones. Every additional tool is a decision the agent has to make correctly, and decisions are where errors live.

Notice what is absent from the CLO list: no `open_menu()`, no `select_tool()`, no `click(x, y)`, no `wait_for_dialog()`. The surface is not a transcription of the GUI. If your tool list reads like a walkthrough of your own interface, you have made the agent responsible for your application's internal state machine — which is a responsibility that already had an owner.

### 5.2 Put units in the schema, never in a comment

Read the original call once more: the user said **8 cm**, the call says **80**. A unit conversion happened silently, inside a model, based on a convention documented in prose.

That is the class of bug that ends with a factory cutting 8 mm instead of 80 mm. Do not accept a bare number:

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

If you cannot change the shape, put it in the name: `length_delta_mm`, `amount_minor_units`, `timeout_seconds`. A parameter name is read on every call; your docs are read once, maybe. Apply this to every ambiguous scalar in your domain — currency, angles, timestamps, sizes. If two reasonable people could read the number differently, the schema decides.

### 5.3 Return handles and state, not prose

`get_garment()` is first in the list for a reason: it is how the agent learns the shape of the world before touching it. Every domain API needs its equivalent.

```json
{
  "garment_id": "skirt-v7",
  "revision": 12,
  "units": "mm",
  "panels": [
    { "id": "skirt_front", "length": 620, "grainline": "warp" },
    { "id": "skirt_back",  "length": 620, "grainline": "warp" }
  ],
  "groups": { "skirt_panel_group": ["skirt_front", "skirt_back"] },
  "last_simulation": { "revision": 11, "status": "stale" }
}
```

**Stable IDs** mean the agent targets `skirt_front`, not "the second panel", so a reordering cannot silently retarget an edit. **`revision`** turns every mutation into an optimistic-concurrency check — pass the revision you expect, get a clean conflict error instead of an edit applied to something that moved underneath you. And `last_simulation.status: "stale"` tells the agent the physics is out of date, which is a dependency it can act on without you explaining it in a system prompt.

That last trick generalizes further than it looks. Anywhere your state has a "this needs recomputing" concept — a report, a forecast, a search index, a compliance check — say so in the payload. It converts prompt engineering into schema.

### 5.4 Separate cheap preview from expensive commit

`run_simulation()` is not free — seconds to minutes of compute here, and in other domains a build, a render job, or a bill. Give the agent a way to check its work before spending:

- `modify_pattern(..., dry_run=true)` → returns the diff and validation errors, changes nothing
- `run_simulation(quality="draft" | "final")` → iterate cheaply, commit once
- `estimate_cost(op)` → wherever the agent's choice has a real price

An agent that can look before it leaps behaves far better than one that can only leap and apologise. It is also the cheapest guardrail you will ever ship against a runaway loop.

### 5.5 Write errors for a reader who will act on them

An agent reads your error and immediately decides what to do next. That makes error text a functional interface, not a diagnostic afterthought.

```
Bad:   Error: operation failed (code 0x8007)

Bad:   ValidationError: constraint violated on panel skirt_front

Good:  ExtendLengthRejected: panel 'skirt_front' extended to 700mm but
       hem_allowance is 15mm and the fabric roll width is 1400mm.
       At 700mm the marker no longer nests two panels per width.
       Options: reduce delta to <= 62mm, or set nesting='single' and
       accept ~18% more fabric per unit.
```

Diagnosis, constraint, numbers, exits. An agent given the third message fixes it in one turn. An agent given the first retries the identical call twice and then tells the user it did not work.

The rule: **write errors as if the reader is competent, has no access to your source, and must decide in five seconds.** That describes an agent, and it also describes a new engineer at 2 a.m., which is why this was always good practice.

### 5.6 Make validation a first-class callable

`validate_collision()` being its own verb — rather than something that happens implicitly inside `export_dxf()` — is one of the best decisions in that five-call list.

As a standalone verb, the agent can call it after every change, bisect which change caused a problem, and self-check before the expensive step. Buried inside export, problems surface at the end, with no way to isolate the cause.

Give it output an agent can reason about — not `false`, but what, where, and how much:

```json
{
  "ok": false,
  "collisions": [
    { "between": ["skirt_front", "left_leg_avatar"],
      "max_penetration_mm": 4.2, "at_frame": 37,
      "pose": "walk_cycle", "severity": "warning" }
  ]
}
```

4.2 mm at one frame of a walk cycle is a different decision from a static 40 mm intersection. A boolean throws away the information the agent needs, and it will then either ignore real problems or block on trivial ones. Your vertical SaaS equivalent is `validate_period_close()`, `check_eligibility()`, `dry_run_payroll()` — every rule engine you have, exposed as something callable rather than something that only runs on submit.

### 5.7 Make mutations reversible, and say what changed

Agents take wrong turns. The question is what a wrong turn costs.

```json
{ "ok": true, "revision": 13, "undo_token": "rev-12->13-a41f",
  "changed": ["skirt_front.length", "skirt_back.length"] }
```

The `changed` array earns as much as the undo token: it lets the agent confirm the blast radius matched its intent, and catches the case where a "length" edit also moved notches it promised to preserve.

---

## 6. The oracle: why three of CLO's five calls are verification

Strip the CLO surface down and there are only two kinds of call:

```
mutate:  modify_pattern, export_dxf
verify:  get_garment, run_simulation, validate_collision
```

Three of five are verification. That ratio is not an accident, and it is the part most teams leave out of version one.

An agent's fundamental weakness is that it cannot distinguish "I did this correctly" from "I produced something that looks like a correct result." Left alone it will generate a plausible pattern, describe it confidently, and be wrong in a way no reader can detect from the text.

The simulator fixes this. It is a **ground-truth oracle**: an external, non-negotiable check the agent cannot talk its way past. Cloth either intersects the body or it does not. And because the oracle is callable, the agent runs the loop itself:

```
propose → apply → verify →
    fail: read the specific failure, adjust, repeat
    pass: export
```

This is why coding agents got useful when they got a test runner and a compiler, not when they got more eloquent. The model did not become more truthful; it got something that says no.

**So the design question for any product is: what is my oracle, and can the agent call it?** In vertical SaaS it is rarely physics — it is your validation rules, your reconciliation job, your eligibility engine, your policy checks. You almost certainly have one. It is almost certainly only reachable by submitting a form. Expose it.

And be honest about its edges. CLO tells you the cloth does not intersect the avatar. It does not tell you the garment is beautiful, that the seams are sewable by your actual factory, or that the pattern grades sensibly to size 18. The episode is clear-eyed about this: turning designs into accurate sewing patterns is still the hard unsolved part, and Yana's response is to run human patternmakers and Codex on it *in parallel* and let the results decide. That is the right posture for anything consequential — keep the existing process running alongside the agent until the outputs agree, not as a rollback plan but as the evaluation.

---

## 7. The three edges: CLI, MCP, webhooks

The bottom row of the architecture diagram is three adapters, and they are not redundant. Each answers a different question.

**CLI — for coding agents and humans in terminals.** Codex, Claude Code, and their successors are extremely good at shell. A well-formed CLI over your domain API gets you agent compatibility with no protocol work at all, and it composes with everything else in a pipeline. Make it script-grade: `--json` output, non-zero exit codes on failure, no interactive prompts unless a TTY is attached, and every verb from the domain API represented.

**MCP — for conversational agents that need discovery.** The value MCP adds over "here is our OpenAPI spec" is that the agent can enumerate what is available, with schemas and descriptions, at runtime. That makes it the right surface when the agent is reasoning about *which* operation to use rather than executing a known script. This is where the schema discipline from section 5 pays off hardest — the description text you write is prompt material on every call.

**Webhooks — because the agent should not poll.** The other three edges are inbound. Webhooks are how your product tells an agent something happened, and they are what turns an agent from a tool into a participant: an invoice arrived, a simulation finished, a check failed. Long-running work in particular needs this shape (see the next section).

The discipline that matters: **all three are thin.** No business logic, no validation that only exists in one of them, no verb that the CLI has and MCP does not. The moment an edge starts making decisions, you have four implementations of your product again — and this time one of them is only reachable by robots.

---

## 8. What breaks in production

The happy path is five calls. Here is what the fifth attempt looks like.

**Session and state.** `get_garment()` implies something is open. Which document, in whose session, and who wins when two runs touch it? Prefer stateless — every call carries the object ID and expected revision. It is more verbose and vastly easier to debug, resume, and run concurrently.

**Long-running operations.** A final-quality simulation can outlive a tool-call timeout. Return a handle, expose polling, and fire a webhook on completion:

```
run_simulation(quality="final") → { "job_id": "sim-8812", "status": "queued", "eta_seconds": 240 }
get_job(job_id)                 → { "status": "running", "progress": 0.4 }
                                → { "status": "done", "result": {...} }
```

Then the agent can do something else while it waits — which is precisely what makes this work for a solo founder: start the long job, go drape actual fabric, come back to the result. The asynchronous shape is not a technical nicety; it is what expands what fits into one person's day.

**Partial failure.** `export_dxf()` succeeds; the write to shared storage fails. Now there is a revision the agent believes is exported and a factory that never got it. Every boundary-crossing call must be idempotent (take a client-supplied key) or transactional. "Probably fine" is a third option that shows up in incident reviews.

**Silent coercion.** The failure that will actually hurt you is not an exception — it is your API accepting `length_delta=80` when the agent meant 8 cm and quietly clamping or reinterpreting it. Reject at the boundary. An error costs one turn; a silent coercion costs a production run.

**Non-determinism in the log.** The same prompt does not produce the same call sequence twice. Log the *calls*, not the conversation, and treat that log as the artifact of record. Then a bad outcome is debuggable: replay the exact five calls without the model and find out whether the agent was wrong or the tool was.

**Runaway cost.** An agent that can call `run_simulation` in a retry loop can spend real money converging. Cap it in code — per-run budget, max calls per tool, and a hard stop that returns something the agent understands (`BudgetExceeded: 6 of 6 simulations used; summarize and ask the user`). Prompts are advisory; code is not.

**Agent identity.** Decide early what an agent *is* in your permission model. It should be a first-class principal acting on behalf of a user, with its own scoped token and its own line in the audit log — not a human's session borrowed by a script. Retrofitting this after your first enterprise security review is much less pleasant than deciding it now.

---

## 9. The commercial consequences

If you sell software, the "new kind of user" line has a P&L behind it.

For twenty years the moat around specialist software was partly capability and substantially **fluency**. CLO, AutoCAD, Ableton, Cadence, ArcGIS, SAP, Avid — each has a real learning cliff, and that cliff produced a professional class whose expertise was partly domain and partly *tool*. The cliff protected the vendor: switching cost was measured in retraining.

An agent that can drive the tool competently changes who can buy it. Yana never learned CLO and still produced CAD files with it. That is not a story about CLO being disrupted — it is CLO's addressable market expanding to everyone who has the design problem but never had six months to spend on the interface. **The same expansion is available to every vertical SaaS product whose adoption is currently gated on "someone has to learn this."**

It is not automatic. It goes to whoever ships the surface. Concretely:

- **Make headless a supported mode.** A per-seat licence with mandatory interactive login is a hard blocker. If a run needs a human to dismiss a dialog, it is not a workflow. Decide what an agent seat is, what it costs, and how it authenticates — before a customer asks and gets "not supported," which is the same sentence as "go evaluate alternatives."
- **Ship your validators loudly.** Your differentiation against a model that fabricates plausible output is that you can *prove* things about the result. `validate_collision()` is not a utility function; it is the reason to route through you at all.
- **Log for audit.** When a customer's agent produces a bad artifact, the first question is whether your tool did what it was told. Answer it with a call log and you keep the customer. Shrug and you are the suspect by default.
- **Price the ceiling, not the seat.** Agents make orders of magnitude more calls than humans. Metering that tracks your real cost — simulations, exports, compute-minutes — survives contact with automation. Per-seat pricing quietly encourages your best customers to share one seat with a bot, which is worse for everyone.
- **Write docs an agent can read.** Your API reference is now training input and runtime context. Complete schemas, real examples, explicit units, and error catalogues are worth more than a beautifully designed docs site with prose-only descriptions.

---

## 10. When computer-use is still the right answer

The GUI path is a fallback, not a mistake. It is correct when:

- **The software has no automation surface and never will.** Legacy internal tools, abandoned vendor products, anything where "add an API" is not a conversation you can have.
- **You are not the vendor and the vendor will not budge.** Sometimes pixels are the only door.
- **The task is genuinely one-off.** Building a good tool surface is a real investment; it pays off on repetition. If you will run it twice, drive the GUI.
- **You are prototyping the workflow before committing to the contract.** Computer-use is a decent way to discover which verbs you actually need — then go build them.

And some limits that apply to the whole pattern, API path included:

**No oracle, no autonomy.** If you cannot programmatically detect a wrong result, keep a human on every output — and be realistic that the human will rubber-stamp by week three. Build the check first.

**Irreversible actions stay behind a human.** The CLO chain ends at a *file*. A person still decides to cut fabric. Keep it that way wherever undo is expensive: agents produce artifacts and recommendations; people authorise the step that spends material, money, or trust.

**Taste is not delegable.** What Yana values is that the image model follows *her* sketches closely rather than generating something impressive-but-generic. An orchestration layer executes a point of view faster. It does not supply one, and work with no point of view is competent and forgettable.

---

## 11. Checklist

**Architecture**
- [ ] The domain API exists as its own layer, and the web UI calls it like any other client
- [ ] The three things customers actually do are each *one* call, not six ordered writes
- [ ] No business rule is implemented in more than one place
- [ ] CLI, MCP, and webhooks are thin adapters with zero unique logic

**Contract**
- [ ] Every verb is something a competent user could say out loud
- [ ] Units are in the schema or the parameter name — never only in prose
- [ ] Objects have stable IDs; mutations carry an expected revision
- [ ] Mutations return what changed and an undo token
- [ ] State exposes staleness so the agent knows what to recompute

**Verification**
- [ ] There is at least one ground-truth oracle the agent can call
- [ ] Validators return structured detail (what, where, how much), never a bare boolean
- [ ] Validation is callable standalone, not only implicit at commit

**Operations**
- [ ] Long jobs return handles, are pollable, and fire webhooks
- [ ] Boundary-crossing calls are idempotent or transactional
- [ ] Invalid input is rejected, never coerced
- [ ] Budgets and call caps are enforced in code, not in the prompt
- [ ] Agents are first-class principals with scoped tokens and audit-log identity
- [ ] The call log — not the chat transcript — is the artifact of record, and it replays

**Errors**
- [ ] Each message gives diagnosis, constraint, numbers, and exits
- [ ] No error requires reading your source to act on

---

## Closing

The sentence at the top — "make the skirt 8 cm longer and regenerate the manufacturing pattern" — used to require being fluent in an expensive application, or hiring someone who was. It is now five calls.

But the fashion story is the illustration, not the lesson. The lesson is that the stack grew a layer, and that layer talks to contracts, not buttons. Every product now has two kinds of user: the one who looks at your interface and the one who reads your schema. The second one is growing faster, is more literal, is much less forgiving of ambiguity, and cannot see a single thing you did not expose.

SaaS is not disappearing. It is gaining a new kind of user — and the vendors who notice will find that user is a lot less price-sensitive about the parts they cannot fake.

---

### Sources

- Lenny's Newsletter — *How I AI*: [How a solo founder used Codex and ChatGPT to launch a fashion brand without engineers](https://www.lennysnewsletter.com/) with Yana Welinder (August 2026)
- [Workflows for an AI-Native Fashion Brand](https://www.chatprd.ai/how-i-ai/workflows-for-an-ai-native-fashion-brand)
- [How to Use AI for Fashion Design and Visualization](https://www.chatprd.ai/how-i-ai/workflows/how-to-use-ai-for-fashion-design-and-visualization)
- [How to Prototype Complex Garments Using AI and 3D Modeling](https://www.chatprd.ai/how-i-ai/workflows/how-to-prototype-complex-garments-using-ai-and-3d-modeling)
- [How to Build and Run an E-Commerce Business with AI as a Technical Co-Founder](https://www.chatprd.ai/how-i-ai/workflows/how-to-build-and-run-an-e-commerce-business-with-ai-as-a-technical-co-founder)
