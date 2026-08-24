# SaaS isn't disappearing. It's gaining a new kind of user.

A solo founder types one sentence:

> "Make the skirt 8 cm longer and regenerate the manufacturing pattern."

And here's what happens underneath it:

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
                           CLO                      ← the software that actually knows cloth
```

That's a real workflow. Yana Welinder is building an AI-native fashion brand solo, without an engineering team, and she uses Codex to operate CLO — professional 3D fashion design software — to produce the CAD files she needs. She never learned CLO. She just describes what she wants.

It's a great story. But the fashion part is the least interesting thing about it, and honestly, CLO isn't the point either.

The point is a line Yana says almost in passing:

> **"SaaS is not disappearing; it is gaining a new kind of user."**

That's the most important idea in the whole episode. And it's not really a claim about fashion, or about AI. It's a claim about how software gets built from here on out.

So let's talk about what it actually means, and what you'd do differently on Monday.

> **Where this comes from:** Lenny Rachitsky's *How I AI* episode with Yana Welinder, [How a solo founder used Codex and ChatGPT to launch a fashion brand without engineers](https://www.lennysnewsletter.com/) (August 2026). Her full workflow write-ups are at [chatprd.ai/how-i-ai](https://www.chatprd.ai/how-i-ai/workflows-for-an-ai-native-fashion-brand). The diagram above isn't a screenshot of her setup — it's the shape her workflow implies, and the shape worth building on purpose.

---

## The stack is growing a layer

For about thirty years, software has been designed for exactly one shape:

```
Human
  ↓
GUI
  ↓
Software
```

Everything follows from that. The GUI is where the product lives. It's where the workflow logic ends up, where validation happens, where the "are you sure?" dialog saves someone from themselves, and where you demo the value. If there's an API, it's the side door — built later, by fewer people, with less love.

That shape is now growing a layer. Two versions of it are showing up:

```
Human                      Human
  ↓                          ↓
AI agent                   AI agent
  ↓                          ↓
MCP / API / CLI            GUI computer-use
  ↓                          ↓
Software                   Software
```

Both work today. But they're not equally good, and the gap is going to widen. **The left path — where the agent talks to a contract — is the one that gets dramatically more reliable over time.** The right path, where the agent looks at your screen and clicks your buttons, is the fallback. It's genuinely impressive, and it stays a fallback.

Here's the part I'd underline: the real change isn't that a box got added. **It's that the translation from "what the human wants" into "what the software does" moved.** It used to happen inside a person's head, guided by an interface you designed. Now it happens inside a model, guided by whatever surface you happened to expose. If the only surface you expose is a GUI, you've quietly decided that every agent touching your product has to come in through the least reliable door available.

CLO is just where this got visible first, because 3D garment simulation has a steep learning curve and a scriptable engine underneath. But it's not a fashion thing. It's what happens to every category of software with real depth to it.

---

## Why one path gets much better and the other doesn't

"More reliable" is easy to assert, so here's the actual reasoning.

**A GUI is a lossy encoding of the operations underneath it.** Your interface takes something like "extend these panels by 80mm" and encodes it as: focus a window, open a menu, wait for a modal, click a field, type digits, press a button whose label changed in v9. A computer-use agent has to decode all that *back* into the original operation, through pixels, every single time. The API path just skips the round trip. You can't out-engineer the fact that one path is doing strictly more work to end up in the same place.

**Failures are loud on one side and silent on the other.** Rename a field in your API and the call breaks — you get an error, in CI, before a customer sees it. Move a button and nothing breaks. The agent just clicks the wrong thing and keeps going, confidently. Silent wrongness is the expensive kind, and the GUI path invites it.

**One path is replayable. The other produces a video.** Same intent through an API gives you the same handful of calls, which you can log, diff, and re-run without the model in the loop. Same intent through computer-use gives you a different click sequence every run. When a customer asks "why did your system do that?", only one of those answers keeps the customer.

**Cost and speed compound.** Every computer-use step is a screenshot, a vision pass, and a decision. A domain call is a few hundred tokens. On a five-step task that's a nuisance. On the hundred-step workflows people actually want agents to run, it's the difference between viable and not.

**Permissions are the one people haven't thought about yet.** A computer-use agent inherits whatever the logged-in human can do. Full session, full blast radius, no scoping. An API-driven agent can hold a token good for three specific verbs, rate-limited, with every call attributed to it in your audit log. One of those you can walk into a security review with. The other one you can't.

| | Agent → API / MCP / CLI | Agent → computer-use |
| --- | --- | --- |
| How good it can get | Limited by your contract | Limited by perception |
| When it fails | Loudly (an error) | Silently (a wrong click) |
| What you can audit | A call log you can replay | A screen recording |
| Cost per step | Low | Screenshot + vision pass |
| Permissions | Scoped per agent | Whatever the human has |
| Concurrency | Many sessions | One desktop |
| What breaks it | Changing your contract | Changing your CSS |

To be clear: computer-use isn't a mistake. It's the right answer in a few specific situations, which I'll get to at the end. But if you own the software, or you can get an automation surface, this isn't a close call.

---

## So if you're building vertical SaaS today

Here's roughly how I'd architect it:

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

Notice what moved. **The domain API is the product now.** The web UI is a client of it — the first one, the one humans use, and no longer the special one. The CLI, the MCP server, and the webhooks are three more clients, all thin, all speaking the same vocabulary.

This isn't a new idea. It's the old API-first argument, and plenty of teams nodded along and then shipped a UI with all the business logic in the controllers anyway, because nothing ever forced the issue.

**What changed is that something now forces the issue.** An agent can't use logic that only exists in your React components. If your workflow rules live in the UI, then to the fastest-growing category of user you'll ever have, your product simply doesn't do those things.

Three reasons this shape is worth the discipline:

**Every rule gets implemented once.** If `approve_invoice` enforces the two-signature policy inside the domain API, then the web UI, the CLI, the agent, and the webhook retry all enforce it. If that policy lives in a submit handler, three of those four paths are a compliance incident waiting to be found.

**Audit gets simple.** Every change goes through one place, so "who did this, through what, with what arguments" has one answer — whether the actor was a person, a cron job, or Codex acting on someone's behalf.

**The edges get cheap.** Once the domain API exists, an MCP server is a few hundred lines of schema. A CLI is an argument parser. Adding whatever protocol replaces MCP in three years is an afternoon instead of a rewrite. That's the real hedge here: nobody knows which agent protocol wins, and if your middle layer is clean, you don't have to care.

The short version: **build the middle box first, and treat every interface as a rendering of it.**

---

## The trap almost everyone is about to walk into

This is the one I'd flag hardest, because it's easy to think you've already solved it.

Most vertical SaaS already has an API. It's usually a CRUD reflection of the database — `GET /invoices`, `POST /invoices`, `PATCH /invoices/{id}` — built for integrations, generated from your models, technically complete. It's very easy to look at that and conclude the agent story is handled.

It isn't, because **your CRUD API exposes your tables, and your product is your workflows.** What "approving an invoice" actually means — which fields move together, which policy applies, what side effects fire, what makes it invalid — lives in a controller behind the UI. Sometimes in the UI itself.

Hand an agent the CRUD API and you've asked it to reimplement your business logic from the outside:

```python
# What the agent has to do against a CRUD API.
# Every line is a chance to get your domain wrong.
inv = GET("/invoices/8812")
GET("/approval_policies?org=42")     # ...which one applies here?
GET("/users/me/permissions")         # ...am I even allowed?
PATCH("/invoices/8812", {"status": "approved", "approved_by": "u_17", ...})
POST("/ledger_entries", {...})       # did the UI also do this? probably?
POST("/notifications", {...})        # and this?
```

It'll work in testing and be wrong in production, in ways that show up a quarter later during reconciliation. And look at those last two lines — the agent is *guessing* at your side effects. Anything your submit handler did that the agent doesn't know about just won't happen.

Here's the domain version:

```python
approve_invoice(
    invoice_id      = "8812",
    approver        = "u_17",
    note            = "matches PO 4471",
    idempotency_key = "codex-run-9f2a",
)
# → policy check, ledger entry, notification, audit record — all of it, once,
#   exactly the way the web UI does it, because it's the same code path.
```

One call. Your rules, your side effects, your audit trail. The agent can't get the policy wrong, because the agent isn't the one implementing the policy.

**Here's a quick test.** Pick the three things your customers actually come to your product to do. Is each one a single call? If "approve an invoice," "onboard a customer," and "close the period" each require the caller to orchestrate six writes in the right order, you don't have a domain API. You have a database with HTTP in front of it. That's a fine integration surface and a bad agent surface.

Which, by the way, is exactly what those five CLO calls get right. `modify_pattern(length_delta=80)` is a domain verb. The CRUD version would expose pattern nodes and let you mutate vertex coordinates — technically more powerful, and completely useless to anyone who isn't already a CLO expert. Which is the entire audience you're trying to reach.

---

## What a good agent-facing API actually looks like

Seven things, roughly in order of how much pain they save. These apply whether you're wrapping CLO or building your own product.

**1. Name your verbs the way a user would say them.** The test: could a competent customer say this sentence out loud? "Extend the skirt panels by 8 cm" — yes. "Set field `pnl_len_d` on the active pattern node" — no. "Approve invoice 8812" — yes. "Update invoice status and insert two ledger rows" — no. You'll end up with fewer verbs than you expect, and that's the win. Five good ones beat forty faithful ones, because every extra tool is one more decision the agent has to get right.

Notice what's *missing* from the CLO list: no `open_menu()`, no `select_tool()`, no `click(x, y)`. It isn't a transcription of the interface. If your tool list reads like a walkthrough of your own UI, you've handed the agent responsibility for your app's internal state machine — a job that already had an owner.

**2. Put units in the schema, never in a comment.** Read that original call again. The user said **8 cm**. The call says **80**. A unit conversion happened silently, inside a model, based on a convention documented in prose somewhere. That's the class of bug that ends with a factory cutting 8mm instead of 80mm. So don't accept a bare number — take `{value: 8, unit: "cm"}`, or at minimum name the parameter `length_delta_mm`. A parameter name gets read on every single call. Your docs get read once, maybe. Same goes for currency, angles, timestamps, sizes: if two reasonable people could read the number differently, your schema should settle it.

**3. Return state the agent can hold onto.** `get_garment()` is first in that list for a reason — it's how the agent learns the shape of the world before touching anything. Give it stable IDs, so the agent targets `skirt_front` rather than "the second panel" and a reordering can't silently retarget an edit. Give it a revision number, so every change carries "here's the version I expected" and a conflict comes back as a clean error instead of an edit applied to something that moved. And tell it what's stale: a field like `last_simulation: {status: "stale"}` teaches the agent it needs to re-run the physics without you explaining that in a prompt. That trick generalizes — anywhere you have a "this needs recomputing" concept, put it in the payload.

**4. Let the agent look before it leaps.** `run_simulation()` isn't free — it's seconds to minutes of compute here, and in other products it's a build, a render job, or a literal bill. So give it a cheap version: `dry_run=true` that returns the diff without changing anything, a `quality="draft"` mode for iteration, an `estimate_cost()` wherever the agent's choice has a price. An agent that can preview behaves enormously better than one that can only act and apologize. It's also the cheapest protection you'll ever ship against a runaway loop.

**5. Write error messages for someone who's about to act on them.** An agent reads your error and immediately decides what to do next, which makes error text a functional interface rather than a diagnostic afterthought. Compare:

```
Bad:   Error: operation failed (code 0x8007)

Good:  ExtendLengthRejected: panel 'skirt_front' extended to 700mm but
       hem_allowance is 15mm and the fabric roll width is 1400mm.
       At 700mm the marker no longer nests two panels per width.
       Options: reduce delta to <= 62mm, or set nesting='single' and
       accept ~18% more fabric per unit.
```

Diagnosis, constraint, numbers, and the way out. An agent given the second one fixes it in a single turn. Given the first, it retries the identical call twice and then tells the user it didn't work. The rule of thumb: write errors as if the reader is smart, has no access to your source code, and has five seconds to decide. That's an agent — and it's also a new engineer at 2 a.m., which is why this was always good practice.

**6. Make validation something you can call on its own.** `validate_collision()` being its own verb, instead of something that quietly happens inside `export_dxf()`, is one of the best decisions in that whole list. As a standalone call, the agent can check after every change, figure out which change caused the problem, and self-check before the expensive step. Buried inside export, problems only surface at the very end with no way to isolate them. And give it real output — not `false`, but what collided, where, and by how much. "4.2mm at one frame of a walk cycle" is a completely different decision from "40mm, standing still." A boolean throws away exactly the information the agent needs.

**7. Make changes reversible, and say what you changed.** Agents take wrong turns; the only question is what a wrong turn costs. Return an undo token from every mutation, and return the list of what actually changed. That second part matters more than it sounds — it's how the agent notices that its "just change the length" edit also moved the notches it promised to leave alone.

---

## The most underrated piece: give the agent something that can tell it no

Strip the CLO surface down and there are only two kinds of call:

```
mutate:  modify_pattern, export_dxf
verify:  get_garment, run_simulation, validate_collision
```

Three of the five are verification. That ratio isn't an accident, and it's the part most teams leave out of v1.

Here's the thing about agents: **they can't tell the difference between "I did this correctly" and "I produced something that looks like a correct result."** Left alone, an agent will generate a plausible pattern, describe it with total confidence, and be wrong in a way you cannot detect by reading the text.

The simulator fixes that. It's an oracle — an outside, non-negotiable check the agent can't talk its way around. The cloth either intersects the body or it doesn't. And because the check is callable, the agent runs its own loop: propose, apply, verify, read the specific failure, adjust, repeat, and only then export.

This is the same reason coding agents got useful the moment they got a test runner and a compiler. The models didn't suddenly become more honest. They got something that says no.

**So the question for your product is: what's my oracle, and can an agent call it?** In vertical SaaS it's rarely physics. It's your validation rules, your reconciliation job, your eligibility engine, your policy checks. You almost certainly already have one — and it's almost certainly only reachable by submitting a form. Expose it.

Just be honest about where it stops. CLO can tell you the cloth doesn't intersect the avatar. It can't tell you the garment is beautiful, that your actual factory can sew those seams, or that the pattern grades sensibly to a size 18. Yana is refreshingly clear that turning designs into accurate sewing patterns is still the hard unsolved part — and her response is to put human patternmakers and Codex on it *at the same time* and let the results decide. That's the right instinct for anything that matters: keep the existing process running next to the agent until the two agree. Not as a rollback plan. As the evaluation.

---

## Why three surfaces instead of one

The bottom row of that architecture diagram isn't redundant. Each surface answers a different question.

**CLI, for coding agents and people in terminals.** Codex, Claude Code, and whatever comes next are extremely good at shell. A decent CLI over your domain API gets you agent compatibility with zero protocol work, and it composes with everything else. Just make it script-grade: `--json` output, non-zero exit codes on failure, no interactive prompts, every verb represented.

**MCP, for conversational agents that need to discover you.** What MCP adds over "here's our OpenAPI spec" is that the agent can enumerate what's available, with schemas and descriptions, at runtime. That's the right surface when the agent is figuring out *which* operation to use rather than running a known script. It's also where your schema discipline pays off most, because the descriptions you write become prompt material on every call.

**Webhooks, so the agent doesn't have to poll.** The other two are inbound. Webhooks are how your product tells an agent that something happened — an invoice arrived, a simulation finished, a check failed. That's what turns an agent from a tool into a participant.

The discipline that matters: **all three stay thin.** No business logic, no validation that only one of them does, no verb the CLI has and MCP doesn't. The moment an edge starts making decisions, you're back to four implementations of your product — and this time one of them is only reachable by robots.

---

## The stuff that breaks once real usage hits

The happy path is five calls. The fifth attempt looks different.

**Long-running work needs a handle, not a blocked call.** A final-quality simulation can outlive a tool-call timeout. Return `{job_id, status, eta_seconds}`, let the agent poll, fire a webhook when it's done. And notice what that unlocks — this is exactly what makes the whole thing work for a solo founder. Start the long job, go drape actual fabric, come back to the result. The async shape isn't a technical nicety. It's what expands how much fits into one person's day.

**Partial failure will bite you.** The export succeeds, the upload to shared storage fails, and now there's a revision the agent thinks went to the factory and a factory that never got it. Anything crossing a boundary needs to be idempotent (take a key from the caller) or transactional. "Probably fine" is a third option that shows up in incident reviews.

**Silent coercion is worse than an error.** The failure that actually hurts isn't an exception — it's your API cheerfully accepting `80` when the agent meant 8cm, and quietly clamping or reinterpreting it. Reject at the boundary. An error costs one turn. A silent coercion costs a production run.

**Log the calls, not the conversation.** The same prompt won't produce the same call sequence twice. Treat the call log as the record, and a bad outcome becomes debuggable — you can replay the exact five calls without the model and find out whether the agent was wrong or your tool was.

**Cap the spending in code, not in the prompt.** An agent that can call `run_simulation` in a retry loop can burn real money converging. Set a per-run budget and a max call count, and return something the agent understands when it hits the wall. Prompts are advisory. Code isn't.

**Decide what an agent *is* in your permission model, now.** It should be a first-class principal acting on behalf of a user, with its own scoped token and its own line in the audit log — not a human's session borrowed by a script. Retrofitting that after your first enterprise security review is a much worse week.

---

## What this means if you sell software

There's a P&L behind that "new kind of user" line.

For twenty years, the moat around specialist software was partly capability and substantially **fluency**. CLO, AutoCAD, Ableton, ArcGIS, SAP, Avid — each has a real learning cliff, and that cliff created a professional class whose expertise was part domain and part *tool*. The cliff protected the vendor. Switching cost was measured in retraining.

An agent that can drive the tool competently changes who's able to buy it. Yana never learned CLO and still produced CAD files with it. That isn't CLO getting disrupted — that's CLO's market expanding to include everyone who has the design problem but never had six months to spend on the interface. **The same expansion is sitting there for every product whose adoption is currently gated on "someone has to learn this first."**

It doesn't happen automatically, though. It goes to whoever ships the surface:

**Make headless a supported mode.** A per-seat license with a mandatory interactive login is a hard blocker. If a run needs a human to dismiss a dialog, it isn't a workflow. Figure out what an agent seat is, what it costs, and how it authenticates before a customer asks — because "not supported" and "go evaluate alternatives" are the same sentence.

**Ship your validators loudly.** Your edge over a model that just fabricates plausible output is that you can *prove* things about the result. `validate_collision()` isn't a utility function. It's the reason to route through you at all.

**Log for audit.** When a customer's agent produces something bad, the first question is whether your tool did what it was told. Answer that with a call log and you keep the customer. Shrug and you're the suspect by default.

**Price the ceiling, not the seat.** Agents make orders of magnitude more calls than humans. Metering that tracks your real cost survives contact with automation. Per-seat pricing quietly encourages your best customers to share one seat with a bot, which is worse for everybody.

**Write docs an agent can read.** Your API reference is now runtime context. Complete schemas, real examples, explicit units, and a proper error catalogue are worth more than a beautiful docs site full of prose.

---

## When the GUI path is still the right call

Computer-use isn't a mistake. It's the right answer when:

- **The software has no automation surface and never will.** Legacy internal tools, abandoned vendor products, anything where "let's add an API" isn't a conversation you can have.
- **You're not the vendor and the vendor won't budge.** Sometimes pixels are the only door.
- **It's genuinely one-off.** Building a good tool surface is a real investment that pays off on repetition. If you'll run it twice, just drive the GUI.
- **You're still figuring out the workflow.** Computer-use is a decent way to discover which verbs you actually need. Then go build them.

And a few limits that apply to all of this, API path included:

**No oracle, no autonomy.** If you can't detect a wrong result programmatically, keep a human on every output — and be realistic that the human will start rubber-stamping by week three. Build the check first.

**Irreversible things stay behind a person.** That whole CLO chain ends at a *file*. A human still decides to cut fabric. Keep it that way anywhere undo is expensive: agents produce artifacts and recommendations, people authorize the step that spends material, money, or trust.

**Taste doesn't delegate.** What Yana values is that the image model follows *her* sketches closely rather than generating something impressive but generic. An orchestration layer executes a point of view faster. It doesn't supply one — and work without a point of view is competent and forgettable.

---

## A checklist, if you want one

**Architecture**
- [ ] The domain API is its own layer, and the web UI calls it like any other client
- [ ] The three things customers actually do are each *one* call, not six ordered writes
- [ ] No business rule is implemented in more than one place
- [ ] CLI, MCP, and webhooks are thin adapters with no unique logic

**Contract**
- [ ] Every verb is something a customer could say out loud
- [ ] Units live in the schema or the parameter name, never only in prose
- [ ] Objects have stable IDs; changes carry an expected revision
- [ ] Mutations return what changed and an undo token
- [ ] State says what's stale, so the agent knows what to re-run

**Verification**
- [ ] There's at least one check the agent can call that can tell it no
- [ ] Validators return what, where, and how much — never a bare boolean
- [ ] Validation is callable on its own, not just implicit at commit

**Operations**
- [ ] Long jobs return handles, are pollable, and fire webhooks
- [ ] Anything crossing a boundary is idempotent or transactional
- [ ] Bad input gets rejected, never quietly coerced
- [ ] Budgets and call caps are enforced in code, not in a prompt
- [ ] Agents are first-class principals with scoped tokens and audit identity
- [ ] The call log — not the chat transcript — is the record, and it replays

**Errors**
- [ ] Every message gives the diagnosis, the constraint, the numbers, and the way out
- [ ] No error requires reading your source code to act on

---

## The takeaway

That sentence at the top — "make the skirt 8 cm longer and regenerate the manufacturing pattern" — used to require being fluent in an expensive piece of software, or hiring someone who was. Now it's five calls.

But the fashion story is the illustration, not the lesson. The lesson is that the stack grew a layer, and that layer talks to contracts, not buttons.

**Every product now has two kinds of user: the one who looks at your interface, and the one who reads your schema.** The second one is growing faster, is far more literal, is much less forgiving of anything ambiguous, and can't see a single thing you didn't expose.

SaaS isn't disappearing. It's gaining a new kind of user — and the companies that notice early are going to find that user is a lot less price-sensitive about the parts they can't fake.

---

### Sources

- Lenny's Newsletter — *How I AI*: [How a solo founder used Codex and ChatGPT to launch a fashion brand without engineers](https://www.lennysnewsletter.com/) with Yana Welinder (August 2026)
- [Workflows for an AI-Native Fashion Brand](https://www.chatprd.ai/how-i-ai/workflows-for-an-ai-native-fashion-brand)
- [How to Use AI for Fashion Design and Visualization](https://www.chatprd.ai/how-i-ai/workflows/how-to-use-ai-for-fashion-design-and-visualization)
- [How to Prototype Complex Garments Using AI and 3D Modeling](https://www.chatprd.ai/how-i-ai/workflows/how-to-prototype-complex-garments-using-ai-and-3d-modeling)
- [How to Build and Run an E-Commerce Business with AI as a Technical Co-Founder](https://www.chatprd.ai/how-i-ai/workflows/how-to-build-and-run-an-e-commerce-business-with-ai-as-a-technical-co-founder)
