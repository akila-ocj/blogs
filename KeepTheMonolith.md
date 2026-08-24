# Keep The Monolith, Keep It Well

There is a good argument going around — Arpit Bhayani makes it well in *Your Monolith Is Already A Distributed System* — that a large, long-lived monolith accumulates every coupling problem of a distributed system while collecting none of its benefits. Shared tables act as an unversioned contract. Module boundaries exist only on a diagram. Synchronous call chains grow deeper every quarter, and the failure domain of a request becomes the union of everything it touches.

The conclusion of that argument is not "split it up". For most teams the honest answer to microservices is still no. The conclusion is narrower and more demanding: **if you are going to keep the monolith, keep it well.**

That phrase is easy to nod along to and hard to act on. So this post is about the acting-on part. Four practices, in the order they pay off:

1. Define real boundaries inside the monolith.
2. Own the data per module.
3. Make the synchronous chains visible.
4. Consciously decide which parts could be async.

None of these require new infrastructure. All of them require you to be deliberate about things the monolith currently lets you be careless about.

## 1. Define Real Boundaries

A boundary that only exists in a diagram is not a boundary. It is a preference, and preferences lose to deadlines.

The test for whether a boundary is real is simple: **can someone violate it without noticing?** If yes, it is not real. The fix is to make violation loud — a failed build, a failed test, a blocked pull request. Somewhere the machine says no.

You have more options here than you think, and they are cheap.

**Make the public surface explicit.** Every module gets one entry point, and everything else in it is internal by convention and, ideally, by tooling.

```
billing/
  __init__.py        # the ONLY thing other modules import
  api.py             # public functions / commands / queries
  models.py          # internal
  repository.py      # internal — nobody outside billing touches this
  _reconciliation.py # internal
```

```python
# billing/__init__.py
from billing.api import (
    charge_invoice,
    get_invoice_summary,
    InvoiceSummary,
)

__all__ = ["charge_invoice", "get_invoice_summary", "InvoiceSummary"]
```

**Then enforce it.** A twenty-line test is enough to turn a convention into a contract:

```python
# tests/test_boundaries.py
import ast, pathlib

# who is allowed to depend on whom
ALLOWED = {
    "checkout": {"cart", "billing", "inventory", "notifications"},
    "billing":  {"ledger"},
    "cart":     set(),
    "ledger":   set(),
}

def imports_in(path):
    tree = ast.parse(path.read_text())
    for node in ast.walk(tree):
        if isinstance(node, ast.ImportFrom) and node.module:
            yield node.module

def test_module_dependencies():
    violations = []
    for module, allowed in ALLOWED.items():
        for path in pathlib.Path(module).rglob("*.py"):
            for imported in imports_in(path):
                top = imported.split(".")[0]
                if top in ALLOWED and top != module and top not in allowed:
                    violations.append(f"{path}: {module} -> {imported}")
    assert not violations, "illegal cross-module imports:\n" + "\n".join(violations)
```

That test also catches the second, subtler violation — reaching *past* a module's front door:

```python
# checkout/service.py

from billing import charge_invoice              # fine
from billing.repository import InvoiceRepo      # not fine — reaches past the front door
```

Extend the check to reject any import of a submodule that the target package does not re-export, and the "I just needed one field, so I imported the repository" shortcut stops being available.

Most ecosystems have a purpose-built tool for this if you would rather not hand-roll it: `import-linter` in Python, ArchUnit in Java, ESLint's `no-restricted-imports` or dependency-cruiser in JavaScript, Go's internal packages, Rust's module visibility, .NET's `InternalsVisibleTo`. Any of them beats a diagram.

**Where to put the boundaries.** Not around technical layers — a `services/` folder next to a `repositories/` folder is not a boundary, it is a filing system. Put them around things that change together for the same business reason: checkout, billing, inventory, identity, notifications. When a new requirement lands, it should mostly touch one module. If every feature touches five, your boundaries are drawn in the wrong place, and no amount of enforcement will fix that.

A useful nudge: **make the dependency graph a DAG and keep it shallow.** Cycles between modules mean you have one module wearing two names. If `billing` imports `checkout` and `checkout` imports `billing`, they will always deploy, break, and be reasoned about together — so either merge them or find the shared concept underneath and pull it out.

## 2. Own The Data Per Module

This is the highest-leverage practice on the list, and the one teams skip because it is the most work.

The rule: **one module writes a set of tables. Every other module reads that data through that module's interface, not through the tables.**

Not "reads carefully". Not "reads read-only". Does not touch the tables. The moment a second module issues `SELECT ... FROM orders`, the columns of `orders` have become a public API — one with no version, no deprecation path, and no list of consumers.

This is what makes the rename in the original argument so dangerous:

```sql
ALTER TABLE orders RENAME COLUMN customer_id TO user_id;
```

If one module owns `orders`, that statement plus one code change is the whole task, and the compiler or test suite tells you if you missed something. If six modules read `orders` directly, that statement is an undocumented breaking change broadcast to five teams, and you find out at 9am from the report that failed at 2am.

### What ownership looks like

Write down the ownership map, keep it in the repo, and keep it honest:

| Tables | Owner | Everyone else uses |
|---|---|---|
| `orders`, `order_items` | `orders` | `orders.get_order()`, `orders.list_for_user()` |
| `invoices`, `payments`, `refunds` | `billing` | `billing.get_invoice_summary()` |
| `users`, `user_profiles` | `identity` | `identity.get_user()` |
| `stock_levels`, `reservations` | `inventory` | `inventory.check()`, `inventory.reserve()` |

Then stop passing rows across boundaries. A module's interface should return its own types, not its ORM models — if `identity.get_user()` hands back a live ORM object, callers will walk its relationships straight back into the tables you were trying to protect, and lazy loading will do it invisibly.

```python
# identity/api.py
from dataclasses import dataclass

@dataclass(frozen=True)
class User:                      # the contract, not the table
    id: int
    email: str
    display_name: str
    country: str

def get_user(user_id: int) -> User:
    row = _repo.fetch(user_id)   # ORM model stays inside identity
    return User(
        id=row.id,
        email=row.email,
        display_name=row.full_name,   # column renames stop here
        country=row.address.country,
    )
```

Note what that mapping layer buys you: `full_name` can become `first_name`/`last_name` next quarter and no caller changes. That is exactly the freedom the shared-table version does not have.

### Enforcing it when the code won't

Code review catches most of it. For the rest, the database will help:

- **Grant per module.** If your modules can connect with different DB users, `GRANT SELECT, INSERT, UPDATE ON orders TO orders_module` and grant nothing else. The database becomes the enforcer, and violations fail in staging rather than surviving to production.
- **Views as a read contract.** When a cross-module read genuinely must happen in SQL — reporting, a join too expensive to do in the application — expose a view owned by the owning module (`orders_public_v1`) and let others read that. A view is a contract you can version and change deliberately; a table is not.
- **Grep for stray table names.** A test that asserts the string `orders` appears in raw SQL only under `orders/` is crude and remarkably effective.

### The joins you will lose

Be honest about the cost, because there is one. Ownership means some queries that used to be one clean join across `users`, `orders`, and `invoices` become two calls and a merge in application code. Sometimes that is slower. Sometimes it is an N+1 waiting to happen.

Three ways out, in order of preference:

1. **Give the owning module a batch method.** `identity.get_users([ids])` instead of N calls to `get_user()`. Most N+1s in a well-bounded monolith are a missing batch method, not a reason to abandon the boundary.
2. **Expose a purpose-built view or read method** for the specific expensive read.
3. **Let the read cross the boundary, deliberately and in writing** — a comment naming the owner and the reason, so the next schema change knows to look.

Option 3 is fine. Doing option 3 silently, fifty times, is how you got here.

Reporting and analytics deserve their own note: they are the most common source of undocumented reads, and they are also the easiest to solve. Give them a read replica and a set of owned views, and stop pretending the analytics module's queries are not part of your schema contract.

## 3. Make The Synchronous Chains Visible

A local function call costs nanoseconds, which is why teams compose deep chains of them without noticing. The chain from the original post is the archetype:

```python
def process_checkout(cart_id):
    cart = cart_service.get(cart_id)
    user = user_service.get(cart.user_id)
    inventory = inventory_service.check(cart.items)
    payment = payment_service.charge(user, cart.total)
    order = order_service.create(cart, payment)
    notification_service.send_confirmation(user, order)
    analytics_service.track_conversion(user, order)
    return order
```

Seven concerns, all synchronous, all required to succeed. Nothing in the code tells you that the last two steps have no business being able to fail the checkout.

Visibility comes first, before you change any of it. You cannot decide what to make async until you can see what is currently coupled.

**Read the real call graph, not the intended one.** Static analysis gets you a first draft in an afternoon — Python's `ast`, `jdeps` for Java, `go list -deps`, `madge` or dependency-cruiser for JS. Generate it in CI, commit the output, and let the diff show you when a chain gets deeper. A dependency graph that changes silently is how the diagram and the code drifted apart in the first place.

**Then instrument the boundaries at runtime**, because static analysis misses everything dynamic: event handlers, signals, DI, background jobs, ORM hooks. You do not need a tracing vendor for this. A decorator on each module's public functions is enough to get real spans:

```python
import functools, time, contextvars

_chain = contextvars.ContextVar("chain", default=())

def boundary(module: str):
    def decorator(fn):
        name = f"{module}.{fn.__name__}"
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            parents = _chain.get()
            token = _chain.set(parents + (name,))
            start = time.perf_counter()
            try:
                return fn(*args, **kwargs)
            finally:
                _chain.reset(token)
                elapsed_ms = (time.perf_counter() - start) * 1000
                log.info(
                    "boundary_call",
                    extra={
                        "module": module,
                        "call": name,
                        "depth": len(parents),
                        "caller": parents[-1] if parents else "entrypoint",
                        "duration_ms": round(elapsed_ms, 2),
                    },
                )
        return wrapper
    return decorator
```

If you already run OpenTelemetry, wrap module entry points in spans instead and get the same picture in a UI you already have.

Two queries over that data pay for the whole exercise:

- **Fan-out per entrypoint** — how many distinct modules does one request actually touch? Anything above four is worth a look.
- **Real callers per module** — who calls `billing`? Compare that list with your intended dependency map. The difference is your architecture debt, itemized.

**Make depth a number people see.** Put the module count of your top ten endpoints on a dashboard, or assert it in a test:

```python
def test_checkout_touches_at_most_four_modules():
    with record_boundary_calls() as calls:
        process_checkout(cart_id=1)
    modules = {c.module for c in calls}
    assert len(modules) <= 4, f"checkout fan-out grew: {sorted(modules)}"
```

That test will fail one day, on a pull request, in front of the person adding the eighth call. That is the entire point. A ratchet like this is worth more than a wiki page, because it argues at the moment the decision is being made.

**And write down the failure domain.** For each critical entrypoint, list what it currently requires to succeed. Nobody intends for a marketing analytics call to be able to fail a payment; it happens because no one ever wrote the list.

## 4. Consciously Decide What Could Be Async

Now the payoff. With the chains visible, go through each step and answer one question: **must this succeed before the user gets a response?**

Sort every step into three buckets:

**Must be synchronous** — the caller cannot proceed without the result, and a failure must abort. Inventory check, payment authorization, order creation. These are correctness.

**Must happen, but not now** — the work is required, but the user does not need to wait, and a retry a second later is fine. Confirmation emails, invoice PDFs, search index updates, webhook fan-out. These belong on a durable queue.

**Nice to have** — analytics, recommendation warming, audit enrichment. These should be incapable of failing the request, whether or not you queue them.

Checkout, re-sorted:

```python
def process_checkout(cart_id):
    cart = cart_service.get(cart_id)
    user = identity.get_user(cart.user_id)

    # synchronous: correctness depends on these
    reservation = inventory.reserve(cart.items)
    payment = billing.charge(user.id, cart.total)
    order = orders.create(cart, payment, reservation)

    # deferred: must happen, need not block the response
    outbox.enqueue("order.confirmed", {"order_id": order.id, "user_id": user.id})

    return order
```

The confirmation email, the analytics event, and the search index update all become subscribers to `order.confirmed`. Checkout's failure domain shrinks from seven concerns to four, and it shrank because someone made a decision rather than because someone drew a diagram.

### Do it properly or it will bite you

Moving work off the request path replaces one failure mode with a different one, and the new one is quieter. Three things are not optional:

**Enqueue in the same transaction as the write.** The classic bug is committing the order and then failing to enqueue the event — or enqueuing, then rolling back the order, and sending a confirmation for an order that does not exist. The transactional outbox pattern solves this: write the event to a table in the same transaction, and let a separate worker publish rows from it.

```python
with db.transaction():
    order = orders.create(cart, payment, reservation)
    db.execute(
        "INSERT INTO outbox (topic, payload) VALUES (%s, %s)",
        ("order.confirmed", json.dumps({"order_id": order.id})),
    )
# a relay process reads unpublished outbox rows and dispatches them
```

If you do not want a queue at all yet, the outbox table *is* the queue. A cron job draining it is a perfectly respectable v1.

**Make handlers idempotent.** At-least-once delivery is the only kind you get. Key each handler on the event id, or on a natural key, and make a second delivery a no-op. "We send two confirmation emails sometimes" is the polite version of this bug; "we refunded twice" is not.

**Give async work the same observability as the request path.** Every queue needs a visible depth, an age-of-oldest-message, a retry policy, and a dead letter queue that a human actually looks at. Async work that fails silently is strictly worse than the synchronous version that failed loudly in the user's face — at least that one got reported.

And keep the list short. Every step you move off the request path is one more thing that can be eventually-consistent in a way the UI has to explain. "Your order is confirmed, the receipt is on its way" is fine. Async is a tool for shrinking failure domains, not a badge.

## 5. Treat Shared Utilities Like Shared Libraries

One more, because it undoes the other four if you skip it.

The `BaseRepository` every module extends. The `ApplicationContext` singleton holding global config. The `EventEmitter` anything can subscribe to. These are your shared libraries, and in a monolith they are worse than the distributed equivalent, because there is no version number making the coupling visible. Every change to them is an unversioned simultaneous upgrade of every consumer.

Treat them accordingly:

- **Give them owners.** A shared utility with no owner accumulates every caller's special case until it is a god object with a helpful name.
- **Keep them boring and dependency-free.** A shared module that imports domain modules is not a utility, it is a hidden coupling superhighway. Utilities depend on the language and the standard library, and nothing of yours.
- **Prefer duplication over premature sharing.** Two similar 20-line functions in two modules that change for different reasons are cheaper than one 60-line function with a `mode` flag and three callers who cannot all be right. Sharing is a bet that the two things will keep changing together; make the bet consciously.
- **Add versions and deprecation paths where the change is real.** New behaviour goes behind a new function name, callers migrate module by module, the old one is deleted when the last one is gone. Exactly the discipline you would use to release a library, because that is what you are doing.
- **Watch the blast radius.** If a change to a "utility" reliably breaks unrelated test suites, that is your signal — the utility is doing domain work and needs to be split along the lines of who actually depends on it.

## An Audit You Can Run This Week

Concrete and time-boxed. A day or two of work, and it will change what you argue about in your next architecture discussion.

1. **List your modules and their intended dependencies.** Write the map down in the repo, even if it is aspirational for now.
2. **Generate the actual import graph** and diff it against that map. Every edge in the diff is a boundary violation you have already shipped.
3. **Build the table-ownership map.** For each table, one owner. Then grep for every module that queries a table it does not own. Count them. That number is your real coupling.
4. **Instrument boundaries for a day** and pull the top ten entrypoints by module fan-out.
5. **For the top three, sort each step** into must-be-sync / must-happen-later / nice-to-have. Notice how many nice-to-haves can currently fail a user-facing request.
6. **Pick the two worst things and fix them.** Not all of it. Two. Then add the ratchet — the import test, the fan-out assertion — so the fix does not quietly reverse next quarter.

Ordering matters, and the ordering is the point. Boundaries first, because without them nothing else holds. Data ownership second, because it is the coupling with the widest blast radius and the longest lead time. Visibility third, because it tells you where you actually are. Async last, because it is the only one that is easy to get wrong in a way that hurts users.

## The Point

A well-structured monolith is genuinely one of the best architectures available. The compiler catches your mistakes. Calls cost nanoseconds. There are no distributed transactions to debug, no partial failures between your own components, no service mesh to operate. Refactoring across a boundary is a rename, not a migration and a deprecation window.

Every one of those benefits comes from the boundaries being *internal* — cheap to cross, cheap to move, checked by tooling. Which is exactly why the boundaries have to exist. The monolith's advantage is not that it has no boundaries; it is that its boundaries are cheap to enforce and cheap to change. Not drawing them throws away the advantage and keeps the coupling.

The choice is not monolith versus microservices. It is deliberate versus accidental. Keep the monolith. Keep it well.

---

*Framing and the "your monolith is already a distributed system" argument: [Arpit Bhayani](https://arpitbhayani.me/).*
