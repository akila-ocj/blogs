# Keeping The Monolith Inside The Domain: DDD In A next-forge Codebase

[next-forge](https://github.com/vercel/next-forge) is a very good answer to a question that is not about your domain. It wires up auth, payments, email, observability, analytics, feature flags, rate limiting, a design system, and a database client, all in a Turborepo, all deployable on day one. That is a delivery problem solved, and solved well.

It has, deliberately, no opinion at all about your domain. Look at what the template ships as its data layer:

```
packages/database/
  index.ts                  # exports `database` (PrismaClient) and re-exports the generated client
  prisma/schema.prisma      # one file, one stub `Page` model
```

```tsx
// apps/app/app/(authenticated)/page.tsx  — the template's own example
const App = async () => {
  const pages = await database.page.findMany();
  const { orgId } = await auth();
  ...
```

A Prisma query inside a React page component. For a template demonstrating data fetching, that is exactly right. As a pattern at forty models and eight feature areas, it is the shortest path to the invisible distributed system — a schema acting as an unversioned global contract, with no layer anywhere that would force you to write down what anything means.

This post is about the middle path: keep next-forge, keep the single deployment and the single Postgres, and put real domain-driven boundaries inside it. Concretely, for the kind of product this comes up for most.

> **Assumed product shape.** I am taking "harbr-like" to mean a multi-tenant B2B data-collaboration/marketplace product: organizations publish **data products**, list them on an **exchange**, consumers **request access**, someone **approves** it, the data gets **delivered** into the consumer's environment, **usage** is metered, and it is **billed** — with an audit trail over all of it. Substitute your own nouns; the mechanics below do not change, only the context map in section 3.

---

## Table of contents

1. [Where next-forge's grain runs against DDD](#1-where-next-forges-grain-runs-against-ddd)
2. [The target shape: bounded contexts as packages](#2-the-target-shape-bounded-contexts-as-packages)
3. [The context map](#3-the-context-map)
4. [Anatomy of one context](#4-anatomy-of-one-context)
5. [Data ownership with a single Prisma schema](#5-data-ownership-with-a-single-prisma-schema)
6. [Enforcement: three layers that actually say no](#6-enforcement-three-layers-that-actually-say-no)
7. [The composition root: RSC and Server Actions](#7-the-composition-root-rsc-and-server-actions)
8. [Cross-context work: one transaction or an event](#8-cross-context-work-one-transaction-or-an-event)
9. [The queries you lose, and how to get them back](#9-the-queries-you-lose-and-how-to-get-them-back)
10. [Where next-forge's own packages fit](#10-where-next-forges-own-packages-fit)
11. [A migration path that keeps shipping](#11-a-migration-path-that-keeps-shipping)
12. [Anti-patterns](#12-anti-patterns)
13. [The audit, for this stack](#13-the-audit-for-this-stack)

---

## 1. Where next-forge's grain runs against DDD

Four properties of the template work against domain boundaries. Know them before you fight them.

**Everything is a peer.** `packages/*` is a flat list of workspace packages, every one importable as `@repo/<name>` from anywhere. There is no notion of a package that is *internal to* another. `@repo/database` is as reachable from a marketing page as from your billing logic.

**One schema, one client, globally typed.** `packages/database/index.ts` ends with:

```ts
export * from "./generated/client";
```

That single line makes every model type in your entire schema an ambient, importable fact for every consumer of the package. This is the database-as-a-global-variable problem expressed in TypeScript: not only can any module query any table, any module can *name* any table's shape and quietly depend on it.

**There is no transport boundary.** With RSC and Server Actions, the UI and the data access live in the same function, in the same file, in the same process. In a system with an HTTP hop you are forced, at least once, to write down a request and response type. Here nothing forces you. The absence of ceremony is the selling point of the stack, and it is also why your boundaries have to be deliberate — the framework will never ask you for one.

**Your identity and payment models are somebody else's.** Clerk owns users, organizations, and memberships. Stripe owns subscriptions and invoices. These are external bounded contexts you do not control and cannot migrate. If `clerkOrgId` and Stripe object shapes leak into every context, you have coupled your whole domain to two vendors' models. They need anti-corruption layers, not direct use.

Now the good news — three things the template gives you that make this much easier than it would be in a hand-rolled Next.js repo:

**`turbo boundaries` is already wired up.** The root `package.json` has:

```json
"boundaries": "turbo boundaries"
```

Turborepo's tag-based dependency enforcement, already scripted, and doing nothing by default because no package declares tags. This is the single highest-leverage unused feature in the template. We use it in section 6.

**`relationMode = "prisma"` means no database-level foreign keys.** The template's datasource ships with it (it is what Neon's pooled/serverless setup wants). Which means cross-model references are already just ID columns with no FK constraint. The single hardest mechanical obstacle to per-context data ownership — foreign keys stitching every table to every other table — does not exist in your schema. The template accidentally pre-committed you to the right thing. Do not undo it.

**`import "server-only"` is an established convention** in the codebase (`packages/database/index.ts` opens with it). You get a build-time error if domain code is pulled into a client bundle. Use it on every context's public entry point.

## 2. The target shape: bounded contexts as packages

Add one tier to `packages/`: a context per bounded context, each a real workspace package.

```
apps/
  app/                      # authenticated UI — presentation + composition ONLY
  api/                      # webhooks, cron, outbox drain — the other composition root
  web/  docs/  email/  storybook/  studio/
packages/
  kernel/                   # shared kernel: Id, Money, Result, DomainEvent, Clock
  domain/
    identity/               # tenants, members  (ACL over Clerk)
    catalog/                # data products, assets, versions, schemas
    exchange/               # listings, publication, discovery
    entitlements/           # access requests, grants, policies
    delivery/               # connections, egress jobs, materialization
    metering/               # usage events, aggregates
    billing/                # plans, contracts, revenue share  (ACL over Stripe)
    governance/             # audit log, approvals, lineage
  read/
    marketplace-view/       # cross-context read models (section 9)
  database/                 # UNCHANGED next-forge package — now a persistence adapter
  auth/ payments/ email/ notifications/ analytics/ observability/ ...   # infra adapters
  design-system/ seo/ next-config/ typescript-config/                  # platform
```

**Why packages and not folders.** A folder boundary is a preference. A package boundary is checkable: it has a `package.json` whose `dependencies` are a written-down dependency map, and it has a `turbo.json` that can carry tags the build enforces. The whole argument of the previous post — a boundary nobody can violate accidentally — needs a machine that says no, and in this repo the machine works on packages.

Two rules govern the tier:

- **One public entry point per context.** `packages/domain/<ctx>/index.ts` is the only file any other package may import. Everything else is internal.
- **The dependency direction is fixed.** `apps/*` → `domain/*` → `kernel`. Contexts may depend on `kernel` and on named other contexts' public APIs. Nothing in `domain/*` ever depends on `apps/*`, on `@repo/design-system`, or on another context's internals.

Keep the context list short and drawn on business seams, not technical layers. If a new requirement routinely touches five contexts, the seams are in the wrong place and no amount of enforcement will rescue that.

## 3. The context map

Write this table down in the repo — `docs/architecture/context-map.md` — and keep it honest. It is the artifact that makes ownership arguable.

| Context | Aggregates | Owns tables | Publishes | Depends on |
|---|---|---|---|---|
| `identity` | Tenant, Member | `tenant`, `membership`, `tenant_settings` | `tenant.provisioned`, `member.joined` | — (ACL over Clerk) |
| `catalog` | DataProduct, Asset, Version | `data_product`, `asset`, `asset_version`, `schema_def` | `product.published`, `version.created` | `identity` |
| `exchange` | Listing | `listing`, `listing_price`, `listing_visibility` | `listing.published`, `listing.withdrawn` | `catalog` |
| `entitlements` | AccessRequest, Grant, Policy | `access_request`, `grant`, `policy` | `access.requested`, `access.granted`, `access.revoked` | `identity`, `catalog` |
| `delivery` | Connection, EgressJob | `connection`, `egress_job`, `egress_attempt` | `delivery.completed`, `delivery.failed` | `entitlements` |
| `metering` | UsageEvent, UsagePeriod | `usage_event`, `usage_rollup` | `usage.recorded` | — |
| `billing` | Contract, RevenueShare | `contract`, `contract_line`, `payout` | `contract.activated`, `invoice.issued` | `identity`, `metering` (ACL over Stripe) |
| `governance` | AuditEntry, Approval | `audit_entry`, `approval` | — (append-only sink) | — |

Note the shape of the dependency column: it is a shallow DAG, and the busiest context (`entitlements` — the heart of an access-brokering product) depends on two others and is depended on by one. No cycles. If `catalog` needed to import `entitlements` to render "do I have access to this?", that is a read-model concern, not a domain dependency — section 9.

**External contexts get an ACL, always.** `identity` is not "the Clerk wrapper". It owns a local `tenant` table, populated by Clerk webhooks in `apps/api`, and exposes *your* `Tenant` type. Then `clerkOrgId` appears in exactly one column in one table, and the seven other contexts speak `TenantId`. The day you leave Clerk, you rewrite one context instead of grepping the monorepo. Same for Stripe inside `billing`.

## 4. Anatomy of one context

Here is `entitlements` in full. This is the pattern you copy seven more times.

```
packages/domain/entitlements/
  package.json
  turbo.json                     # tags — section 6
  index.ts                       # THE public API
  src/
    domain/                      # pure. no prisma, no clerk, no next, no react
      grant.ts
      access-request.ts
      policy.ts
      errors.ts
    application/                 # use cases + ports
      request-access.ts
      approve-request.ts
      revoke-grant.ts
      check-access.ts
      ports.ts
    infra/                       # the ONLY place @repo/database is imported
      grant.repository.ts
      access-request.repository.ts
      mappers.ts
    events.ts                    # versioned outbound event contracts
    dto.ts                       # what crosses the boundary
```

**The domain layer is pure and knows nothing.**

```ts
// src/domain/grant.ts
import type { AssetId, TenantId } from "@repo/kernel";
import { DomainError } from "./errors";

export type GrantStatus = "active" | "expired" | "revoked";

export class Grant {
  private constructor(
    readonly id: string,
    readonly assetId: AssetId,
    readonly granteeTenantId: TenantId,
    readonly scope: readonly string[],
    readonly expiresAt: Date | null,
    private status: GrantStatus
  ) {}

  static issue(input: {
    id: string;
    assetId: AssetId;
    granteeTenantId: TenantId;
    scope: readonly string[];
    expiresAt: Date | null;
    now: Date;
  }): Grant {
    if (input.scope.length === 0) {
      throw new DomainError("a grant must carry at least one scope");
    }
    if (input.expiresAt && input.expiresAt <= input.now) {
      throw new DomainError("a grant cannot be issued already expired");
    }
    return new Grant(
      input.id,
      input.assetId,
      input.granteeTenantId,
      input.scope,
      input.expiresAt,
      "active"
    );
  }

  revoke(): void {
    if (this.status !== "active") {
      throw new DomainError(`cannot revoke a ${this.status} grant`);
    }
    this.status = "revoked";
  }

  permits(scope: string, now: Date): boolean {
    if (this.status !== "active") return false;
    if (this.expiresAt && this.expiresAt <= now) return false;
    return this.scope.includes(scope);
  }
}
```

Every invariant of a grant lives in one class, enforced on construction and mutation. `permits()` is the business rule that would otherwise be reimplemented — slightly differently — in a page component, a Server Action, and a cron job.

**The application layer defines what it needs and depends on nothing concrete.**

```ts
// src/application/ports.ts
import type { Grant } from "../domain/grant";
import type { AccessRequest } from "../domain/access-request";

export type GrantRepository = {
  findById(id: string): Promise<Grant | null>;
  findActiveFor(assetId: string, tenantId: string): Promise<Grant[]>;
  save(grant: Grant): Promise<void>;
};

export type AccessRequestRepository = {
  findById(id: string): Promise<AccessRequest | null>;
  save(request: AccessRequest): Promise<void>;
};

export type EventPublisher = {
  publish(events: readonly { topic: string; payload: unknown }[]): Promise<void>;
};

export type Clock = { now(): Date };
export type IdGenerator = { next(): string };

export type Deps = {
  grants: GrantRepository;
  requests: AccessRequestRepository;
  events: EventPublisher;
  clock: Clock;
  ids: IdGenerator;
};
```

```ts
// src/application/approve-request.ts
import { Grant } from "../domain/grant";
import { DomainError } from "../domain/errors";
import type { Deps } from "./ports";
import type { GrantDto } from "../dto";
import { toGrantDto } from "./mappers";

export const approveRequest =
  (deps: Deps) =>
  async (input: {
    requestId: string;
    approvedBy: string;
    ttlDays: number | null;
  }): Promise<GrantDto> => {
    const request = await deps.requests.findById(input.requestId);
    if (!request) throw new DomainError("access request not found");

    const now = deps.clock.now();
    request.approve(input.approvedBy, now);          // invariant lives in the aggregate

    const grant = Grant.issue({
      id: deps.ids.next(),
      assetId: request.assetId,
      granteeTenantId: request.requesterTenantId,
      scope: request.requestedScope,
      expiresAt: input.ttlDays
        ? new Date(now.getTime() + input.ttlDays * 86_400_000)
        : null,
      now,
    });

    await deps.requests.save(request);
    await deps.grants.save(grant);
    await deps.events.publish([
      {
        topic: "entitlements.access.granted.v1",
        payload: {
          grantId: grant.id,
          assetId: grant.assetId,
          granteeTenantId: grant.granteeTenantId,
          scope: grant.scope,
          expiresAt: grant.expiresAt?.toISOString() ?? null,
        },
      },
    ]);

    return toGrantDto(grant);
  };
```

Two things this buys immediately. The use case is testable with three object literals and no database — which matters more than it sounds, because a domain test suite that runs in 200ms is a suite people run. And the event payload is now a *written contract* with a version in its name, rather than whatever fields the consumer happened to read off a Prisma row.

**The infra layer is the only place Prisma exists.**

```ts
// src/infra/grant.repository.ts
import { database } from "@repo/database";
import { Grant } from "../domain/grant";
import type { GrantRepository } from "../application/ports";
import { toDomain, toRow } from "./mappers";

export const prismaGrantRepository = (db = database): GrantRepository => ({
  async findById(id) {
    const row = await db.grant.findUnique({ where: { id } });
    return row ? toDomain(row) : null;
  },
  async findActiveFor(assetId, tenantId) {
    const rows = await db.grant.findMany({
      where: { assetId, granteeTenantId: tenantId, status: "active" },
    });
    return rows.map(toDomain);
  },
  async save(grant) {
    const row = toRow(grant);
    await db.grant.upsert({ where: { id: row.id }, create: row, update: row });
  },
});
```

The `db = database` default parameter is what lets a use case run inside a caller's transaction later (section 8) without the domain layer knowing what a transaction is.

**The public API is small, explicit, and server-only.**

```ts
// index.ts
import "server-only";

import { prismaGrantRepository } from "./src/infra/grant.repository";
import { prismaAccessRequestRepository } from "./src/infra/access-request.repository";
import { outboxPublisher } from "./src/infra/outbox.publisher";
import { approveRequest } from "./src/application/approve-request";
import { requestAccess } from "./src/application/request-access";
import { revokeGrant } from "./src/application/revoke-grant";
import { checkAccess } from "./src/application/check-access";

const deps = () => ({
  grants: prismaGrantRepository(),
  requests: prismaAccessRequestRepository(),
  events: outboxPublisher(),
  clock: { now: () => new Date() },
  ids: { next: () => crypto.randomUUID() },
});

export const entitlements = {
  requestAccess: (i: Parameters<ReturnType<typeof requestAccess>>[0]) =>
    requestAccess(deps())(i),
  approveRequest: (i: Parameters<ReturnType<typeof approveRequest>>[0]) =>
    approveRequest(deps())(i),
  revokeGrant: (i: Parameters<ReturnType<typeof revokeGrant>>[0]) =>
    revokeGrant(deps())(i),
  checkAccess: (i: Parameters<ReturnType<typeof checkAccess>>[0]) =>
    checkAccess(deps())(i),
};

export type { GrantDto, AccessRequestDto } from "./src/dto";
export { DomainError } from "./src/domain/errors";
```

Note what is **not** exported: `Grant`, the repositories, the Prisma row types, the mappers. Callers get use cases and DTOs. That is the whole contract, and it is a contract you can keep stable while you rewrite everything behind it.

And `dto.ts` is plain data — no class instances, no Prisma models, nothing lazily loaded:

```ts
// src/dto.ts
export type GrantDto = {
  id: string;
  assetId: string;
  granteeTenantId: string;
  scope: string[];
  status: "active" | "expired" | "revoked";
  expiresAt: string | null;      // ISO — serializes cleanly across the RSC boundary
};
```

The RSC boundary is unforgiving about this: a Prisma `Decimal` or a class instance handed to a client component is a runtime serialization error. Returning DTOs is not just DDD hygiene here, it is what makes the stack work.

## 5. Data ownership with a single Prisma schema

One database, one Prisma client, one migration history — and still per-context ownership. Four mechanics, in increasing order of strictness. Most teams should do the first three.

**5.1 Split the schema into a folder, one file per context.** Prisma supports pointing `schema` at a directory instead of a file, and next-forge already has a `prisma.config.ts` to change:

```ts
// packages/database/prisma.config.ts
import { defineConfig } from "prisma/config";

export default defineConfig({
  schema: "prisma/schema",              // was "prisma/schema.prisma"
  migrations: { path: "prisma/migrations" },
  datasource: { url: process.env.DATABASE_URL ?? "" },
});
```

```
packages/database/prisma/schema/
  _datasource.prisma        # generator + datasource block only
  identity.prisma
  catalog.prisma
  exchange.prisma
  entitlements.prisma
  delivery.prisma
  metering.prisma
  billing.prisma
  governance.prisma
```

Then make ownership social as well as structural, with `CODEOWNERS`:

```
/packages/domain/entitlements/                        @org/access-team
/packages/database/prisma/schema/entitlements.prisma  @org/access-team
/packages/database/prisma/schema/_datasource.prisma   @org/platform
```

Now a schema change to someone else's tables cannot merge without their review. That is the cheapest useful boundary in this whole post, and it takes twenty minutes.

**5.2 No cross-context relations. IDs only.** Within `entitlements.prisma`, relations between `access_request` and `grant` are fine. Across files, never — store the ID and resolve through the owning context's API:

```prisma
// entitlements.prisma
model Grant {
  id               String    @id @default(uuid())
  assetId          String    // -> catalog.Asset. No @relation. Deliberate.
  granteeTenantId  String    // -> identity.Tenant. No @relation. Deliberate.
  scope            String[]
  status           String
  expiresAt        DateTime?
  createdAt        DateTime  @default(now())

  @@index([granteeTenantId, assetId, status])
}
```

Because next-forge ships `relationMode = "prisma"`, you were never getting database-enforced foreign keys anyway. The cost of the boundary is close to zero and the benefit is real: no Prisma `include` can silently traverse from `entitlements` into `catalog`'s tables and make a schema change over there break something over here.

**5.3 A test that keeps ownership honest.** Model access is the thing that erodes fastest. Assert it:

```ts
// packages/domain/entitlements/__tests__/ownership.test.ts
import { readdirSync, readFileSync } from "node:fs";
import { join } from "node:path";
import { expect, test } from "vitest";

const OWNED = ["grant", "accessRequest", "policy"];
const infra = join(__dirname, "../src/infra");

test("entitlements only touches its own models", () => {
  const offenders: string[] = [];
  for (const file of readdirSync(infra)) {
    const src = readFileSync(join(infra, file), "utf8");
    for (const [, model] of src.matchAll(/\bdb\.([a-zA-Z]+)\./g)) {
      if (!OWNED.includes(model)) offenders.push(`${file}: db.${model}`);
    }
  }
  expect(offenders, `foreign model access:\n${offenders.join("\n")}`).toEqual([]);
});
```

Crude, and it will catch the 3am shortcut. Pair it with a second test asserting `@repo/database` is imported nowhere outside `src/infra/`.

**5.4 Optional: Postgres schemas and grants.** If you want the database itself to enforce ownership, Prisma can map models to separate Postgres schemas (`@@schema("entitlements")`) — check whether that is still behind a preview flag in your Prisma version. With separate schemas you can `GRANT` per role and, if your connection setup allows per-context credentials, get failures in staging instead of silent cross-context reads in production.

Be honest about the cost on a serverless Postgres like Neon: multiple connection strings means multiple pools, and next-forge's single `database` singleton becomes several. Worth it for a regulated product where "prove module X cannot read table Y" is an audit question. Overkill for most.

## 6. Enforcement: three layers that actually say no

Three mechanisms, because each sees something the others cannot. Anything you leave to code review alone will drift.

**Layer 1 — `package.json` dependencies are the dependency map.** Write the context map from section 3 as actual declared dependencies:

```json
{
  "name": "@repo/domain-entitlements",
  "private": true,
  "main": "./index.ts",
  "types": "./index.ts",
  "dependencies": {
    "@repo/kernel": "workspace:*",
    "@repo/database": "workspace:*",
    "@repo/domain-identity": "workspace:*",
    "@repo/domain-catalog": "workspace:*",
    "server-only": "^0.0.1"
  }
}
```

No `@repo/design-system`, no `@repo/auth`, no `next`, no `react`. The file is now a reviewable statement of what this context is allowed to know about. Bun hoists `node_modules`, so an undeclared import may still resolve at runtime — which is exactly why this layer alone is not enough.

**Layer 2 — `turbo boundaries`, the package-graph enforcer.** Tag each package. In `packages/domain/entitlements/turbo.json`:

```json
{ "extends": ["//"], "tags": ["domain"] }
```

Tag `packages/database` as `persistence`, `packages/kernel` as `kernel`, `packages/design-system` as `ui`, apps as `entry`. Then in the root `turbo.json`:

```json
{
  "boundaries": {
    "tags": {
      "persistence": {
        "dependents": { "allow": ["domain", "read"] }
      },
      "domain": {
        "dependencies": { "allow": ["domain", "kernel", "persistence", "infra-adapter"] }
      },
      "kernel": {
        "dependencies": { "allow": [] }
      }
    }
  }
}
```

Read those three rules in plain English:

- Only `domain` and `read` packages may depend on the database. **An app can no longer import `@repo/database` at all** — which single-handedly kills the template's `database.page.findMany()`-in-a-page pattern across the whole repo.
- A domain context may depend on other contexts, the kernel, persistence, and infra adapters — never on `ui` or on an app.
- The shared kernel depends on nothing. Keeping it that way is what stops it from becoming the god object every context is coupled through.

Run it in CI, and note that Turborepo applies these transitively — a sneaky re-export chain does not launder a violation:

```yaml
- run: bun run boundaries      # already scripted in next-forge's root package.json
```

Boundaries is still marked experimental; treat it as one of your layers, not the whole story.

**Layer 3 — Biome, the import-specifier enforcer.** `turbo boundaries` reasons about the package graph. It cannot see a deep path import that reaches *past* a package's front door. Biome can, per file, and next-forge already lints with Biome + Ultracite. Add to `biome.jsonc`:

```jsonc
{
  "extends": ["ultracite/core", "ultracite/react", "ultracite/next"],
  "overrides": [
    {
      "includes": ["apps/**"],
      "linter": {
        "rules": {
          "style": {
            "noRestrictedImports": {
              "level": "error",
              "options": {
                "paths": {
                  "@repo/database": "Apps must not touch the database. Call a use case from @repo/domain-*."
                },
                "patterns": [
                  {
                    "group": ["@repo/domain-*/src/**"],
                    "message": "Import a context's public API only: @repo/domain-<ctx>."
                  }
                ]
              }
            }
          }
        }
      }
    },
    {
      "includes": ["packages/domain/**"],
      "linter": {
        "rules": {
          "style": {
            "noRestrictedImports": {
              "level": "error",
              "options": {
                "patterns": [
                  {
                    "group": ["@repo/design-system/**", "@repo/domain-*/src/**"],
                    "message": "Domain code has no UI, and reaches no context's internals."
                  }
                ]
              }
            }
          }
        }
      }
    }
  ]
}
```

Between the two, the interesting violations are covered: layer 2 catches "this package should not depend on that package", layer 3 catches "this file reached past the front door". A short architecture test (section 5.3, extended to check `@repo/database` imports live only under `src/infra/`) mops up what neither sees.

Order matters when you adopt these. Turn layer 3 on with an `includes` list of the directories that are already clean, and widen it as you migrate. A rule that fails on 200 existing files gets switched off within a week.

## 7. The composition root: RSC and Server Actions

`apps/app` becomes presentation and composition. It renders, it authenticates, it validates input, it calls use cases. It does not know what a table is.

The template's page, rewritten:

```tsx
// apps/app/app/(authenticated)/catalog/page.tsx
import { auth } from "@repo/auth/server";
import { notFound } from "next/navigation";
import { catalog } from "@repo/domain-catalog";
import { entitlements } from "@repo/domain-entitlements";
import { ProductGrid } from "./components/product-grid";

const CatalogPage = async () => {
  const { orgId } = await auth();
  if (!orgId) notFound();

  const products = await catalog.listPublished({ tenantId: orgId });
  const access = await entitlements.checkAccess({          // batched, one call
    tenantId: orgId,
    assetIds: products.map((p) => p.primaryAssetId),
  });

  return <ProductGrid products={products} access={access} />;
};

export default CatalogPage;
```

Two context calls, both batched, both returning DTOs. No `database` import anywhere — which is now enforced, not merely intended.

Server Actions are the write side. Validate at the edge with Zod (already a dependency), map domain errors to user-facing results, and let the use case own the rules:

```ts
// apps/app/app/actions/entitlements/approve.ts
"use server";

import { z } from "zod";
import { auth } from "@repo/auth/server";
import { entitlements, DomainError } from "@repo/domain-entitlements";
import { revalidatePath } from "next/cache";

const schema = z.object({
  requestId: z.string().uuid(),
  ttlDays: z.number().int().min(1).max(365).nullable(),
});

export const approveAccessRequest = async (input: unknown) => {
  const { userId, orgId } = await auth();
  if (!userId || !orgId) return { error: "unauthorized" as const };

  const parsed = schema.safeParse(input);
  if (!parsed.success) return { error: "invalid input" as const };

  try {
    const grant = await entitlements.approveRequest({
      requestId: parsed.data.requestId,
      approvedBy: userId,
      ttlDays: parsed.data.ttlDays,
    });
    revalidatePath("/access-requests");
    return { data: grant };
  } catch (error) {
    if (error instanceof DomainError) return { error: error.message };
    throw error;
  }
};
```

Note the split of responsibilities: the action does auth, shape validation, cache invalidation, and error mapping — all genuinely presentation concerns. "Can this request be approved twice?" is not in there, because it lives in the aggregate where it belongs.

**`apps/api` is your second composition root**, and it matters more than it looks in this architecture. It hosts:

- **Clerk webhooks** → `identity` ACL. Organization created in Clerk becomes `identity.provisionTenant()`, writing your `tenant` row.
- **Stripe webhooks** → `billing` ACL. Same shape.
- **The outbox drain** (section 8) on a cron schedule.
- **Scheduled domain work**: expiring grants, rolling up usage, retrying failed egress jobs.

Both apps import the same context packages, in the same process model, against the same database. One domain, two entry points — still a monolith, and now one with a domain layer that neither entry point can bypass.

## 8. Cross-context work: one transaction or an event

Because it is one Postgres, you *can* transact across contexts. Sometimes you should. Decide per case, using the sort from the previous post: must-be-synchronous, must-happen-but-not-now, nice-to-have.

**When it must be one atomic write**, pass the transaction handle down. Prisma's `$transaction` plus `AsyncLocalStorage` gives a unit of work without the domain layer ever hearing the word "transaction":

```ts
// packages/database/tx.ts
import { AsyncLocalStorage } from "node:async_hooks";
import { database } from "./index";
import type { PrismaClient } from "./generated/client";

type Tx = Omit<PrismaClient, "$transaction" | "$connect" | "$disconnect">;
const store = new AsyncLocalStorage<Tx>();

export const db = (): Tx => store.getStore() ?? database;

export const inTransaction = <T>(fn: () => Promise<T>): Promise<T> =>
  database.$transaction((tx) => store.run(tx, fn));
```

Repositories call `db()` instead of importing `database` directly, and the composition root decides the boundary:

```ts
// approving a request AND writing the audit entry must be atomic
await inTransaction(async () => {
  const grant = await entitlements.approveRequest({ requestId, approvedBy, ttlDays });
  await governance.record({ action: "access.approved", subject: grant.id, actor: approvedBy });
  return grant;
});
```

A purist would object that two aggregates in two contexts changed in one transaction. The purist is describing a constraint of distributed systems that you do not have. Use it where atomicity is genuinely required — and keep the list short, because every cross-context transaction is a coupling you would have to unpick if that context ever moved out.

**When it must happen but not now, use an outbox.** Same transaction as the write, so the event cannot be lost or invented:

```prisma
// governance.prisma  (or its own outbox.prisma)
model OutboxEvent {
  id          String    @id @default(uuid())
  topic       String
  payload     Json
  createdAt   DateTime  @default(now())
  publishedAt DateTime?
  attempts    Int       @default(0)

  @@index([publishedAt, createdAt])
}
```

```ts
// packages/domain/entitlements/src/infra/outbox.publisher.ts
import { db } from "@repo/database/tx";
import type { EventPublisher } from "../application/ports";

export const outboxPublisher = (): EventPublisher => ({
  async publish(events) {
    if (events.length === 0) return;
    await db().outboxEvent.createMany({
      data: events.map((e) => ({ topic: e.topic, payload: e.payload as object })),
    });
  },
});
```

Because `db()` resolves to the ambient transaction, the outbox insert commits with the grant or not at all. That is the whole point, and it is the bug people ship when they enqueue to a real queue right after committing.

The drain lives in `apps/api` on a cron route — start there and move to a real queue when volume demands it, not before:

```ts
// apps/api/app/cron/drain-outbox/route.ts
export const POST = async () => {
  const batch = await database.outboxEvent.findMany({
    where: { publishedAt: null },
    orderBy: { createdAt: "asc" },
    take: 100,
  });

  for (const event of batch) {
    try {
      await dispatch(event.topic, event.payload);   // fan out to subscribing contexts
      await database.outboxEvent.update({
        where: { id: event.id },
        data: { publishedAt: new Date() },
      });
    } catch {
      await database.outboxEvent.update({
        where: { id: event.id },
        data: { attempts: { increment: 1 } },
      });
    }
  }
  return Response.json({ processed: batch.length });
};
```

Applied to the harbr-like flow, approving an access request sorts like this:

| Step | Bucket | Why |
|---|---|---|
| Validate request, issue grant | sync, same tx | Correctness. Nothing else may proceed without it. |
| Write audit entry | sync, same tx | Regulatory. A grant with no audit trail is a defect. |
| Provision delivery connection | async, outbox | Minutes are fine; needs its own retries. |
| Notify requester | async, outbox | At-least-once, idempotent on grant id. |
| Reindex for search | async, outbox | Eventually consistent by nature. |
| Increment usage/metering counters | async, outbox | Aggregate, not transactional. |

Failure domain of "approve": two contexts, not six. And three non-negotiables come with the async half — **idempotent handlers** keyed on event id (at-least-once is the only delivery you get), **a visible queue depth and oldest-unpublished age**, and **a dead-letter path a human reads**. Silent async failure is worse than the loud synchronous version you replaced.

## 9. The queries you lose, and how to get them back

This is where per-context ownership meets a marketplace UI, and it is the pressure point in this whole architecture. Your catalog browse page wants: product name, owning organization, listing price, and "does my tenant have access?" — four contexts, and you have just banned the join that used to answer it.

Three options, in the order you should reach for them.

**Batch methods on the owning context.** Most apparent N+1s are a missing plural. `entitlements.checkAccess({ tenantId, assetIds })` returning a map is one query, not fifty, and it keeps the boundary intact. Do this first, and it solves most of the problem.

**A read-model package.** For a genuinely cross-context screen, build the projection explicitly. `packages/read/marketplace-view` owns its own denormalized table, updated by subscribing to the domain events already flowing through your outbox:

```ts
// packages/read/marketplace-view/src/projector.ts
export const onListingPublished = async (e: ListingPublishedV1) => {
  await db().marketplaceCard.upsert({
    where: { listingId: e.listingId },
    create: {
      listingId: e.listingId,
      productName: e.productName,
      ownerTenantId: e.ownerTenantId,
      ownerDisplayName: e.ownerDisplayName,
      priceCents: e.priceCents,
      publishedAt: new Date(e.occurredAt),
    },
    update: { productName: e.productName, priceCents: e.priceCents },
  });
};
```

One indexed table, one query, no joins across contexts, and a browse page that stays fast as the catalog grows. The cost is real and worth naming: it is eventually consistent, it needs a rebuild-from-scratch command for when a projector bug ships, and the events must carry enough denormalized data to build the card. Pay it for the two or three screens that need it, not for every list view.

**A view owned by the owning context.** When a report genuinely must be SQL, expose a versioned view (`catalog_public_v1`) that reads only that context's tables, and let others read the view. A view is a contract you can change deliberately; a table is not.

And for analytics, stop pretending: give it a read replica and a set of owned views. Ad-hoc analytics queries against production tables are the most common source of undocumented schema consumers in any product like this.

## 10. Where next-forge's own packages fit

An easy mistake is treating the template's packages as peers of your domain. They are not — they are **infrastructure adapters**, and the distinction changes where they are allowed to appear.

| Package | Role | Rule |
|---|---|---|
| `@repo/database` | persistence adapter | Only inside `domain/*/src/infra/**` and `read/*` |
| `@repo/auth` (Clerk) | identity adapter | Apps (session reading) + the `identity` ACL. Never in another context. |
| `@repo/payments` (Stripe) | payment adapter | The `billing` ACL only |
| `@repo/email`, `@repo/notifications` | delivery adapters | Injected as ports; called from outbox handlers |
| `@repo/analytics`, `@repo/observability` | telemetry | Anywhere, but never as a control-flow dependency |
| `@repo/design-system`, `@repo/seo` | presentation | Apps only. Never in `domain/*`. |
| `@repo/feature-flags` | configuration | Apps and application layer; never inside `domain/` |

The important one is the third column of the notification row. If `entitlements` imports `@repo/notifications` directly, then testing "approve a request" requires stubbing Knock, and your domain package has grown a vendor dependency. Instead the context declares a port, and the composition root injects the adapter:

```ts
// port, in the context
export type Notifier = { accessGranted(input: { to: string; grantId: string }): Promise<void> };

// adapter, wired in apps/api's outbox handler
import { notifications } from "@repo/notifications";
const notifier: Notifier = {
  accessGranted: ({ to, grantId }) =>
    notifications.trigger("access-granted", { recipients: [to], data: { grantId } }),
};
```

Same for feature flags: a flag is a decision made in the application layer and passed in, not a global your domain rules read. Domain code that reads flags directly cannot be tested for either branch without a flag server.

## 11. A migration path that keeps shipping

If you already have a next-forge app with thirty models and Prisma calls scattered through pages and actions, do not rewrite. Strangle it, one context at a time, and keep the product shipping.

1. **Week 1 — write the context map** (section 3) and split `schema.prisma` into a schema folder with a file per context. Zero behaviour change; entirely mechanical; instantly makes ownership visible. Add `CODEOWNERS`.
2. **Add `packages/kernel`** with `Id`, `Money`, `Result`, `DomainEvent`, `Clock`. Keep it under 200 lines and dependency-free. Resist everything that wants to live there.
3. **Extract the highest-value context first** — the one with the most business rules and the most bugs. For a harbr-like product that is `entitlements`, every time. Build it in the full shape from section 4. Do not touch the other twenty-nine models.
4. **Turn on enforcement narrowly.** Tag the new context and `@repo/database`, add the `persistence.dependents.allow` rule, and add the Biome override with `includes` scoped to the directories that are already clean. Widen the glob as you migrate — never start with `apps/**` if `apps/**` is dirty.
5. **Add the outbox** and move the first genuinely non-critical step (notifications, or search reindexing) off the request path. One step. Verify the dead-letter path works before adding a second.
6. **Repeat, cheapest-first.** Each context you extract makes the next one easier, because the tables it stops sharing are tables nobody else reads any more.

Two ratchets are what make this stick: `bun run boundaries` in CI from step 4 onward, and the Biome `includes` glob that only ever grows. Without them you will do this again in eighteen months.

## 12. Anti-patterns

**Splitting the deployment.** Nothing here needs a second service. You are buying boundaries, not network hops. The moment there is an HTTP call between your own contexts you have bought distributed transactions, partial failure, and a deploy ordering problem, and sold the compiler.

**A package per aggregate.** `packages/domain/grant` is too small. Contexts are drawn on business seams — the unit is "the part of the business that owns these rules", not "one class".

**DDD ceremony where there is no domain.** Notifications, feature flags, and file storage are CRUD over someone else's API. A repository, a factory, and a specification pattern around a Knock call is cost with no invariant to protect. Reserve the full shape for contexts with real rules: entitlements, billing, delivery.

**`@repo/database` as a "shared domain" package.** The strongest temptation, because Prisma types are right there and they typecheck. Once your `Grant` type *is* the Prisma row, the schema is your contract again and you have rebuilt the problem with extra steps.

**Clerk and Stripe objects as your aggregates.** `clerkOrgId` belongs in one column of one table. If it is a parameter on forty functions across eight contexts, you have made a vendor your domain model.

**A kernel that grows.** The shared kernel is the one package everything depends on, which makes it the highest-blast-radius code in the repo. Anything domain-specific that lands there couples every context to every other. Prefer duplicating a small type over sharing it — two similar 20-line types that change for different reasons are cheaper than one shared type with three callers who cannot all be right.

**Lint rules turned on repo-wide on day one.** They fail on 200 files, someone adds an ignore comment, and the boundary is decoration by Friday.

## 13. The audit, for this stack

Half a day, and it will change what your next architecture conversation is about.

1. `grep -rn "@repo/database" apps/` — every hit is a boundary violation you have already shipped. Count them. That number is your starting position.
2. Count models in `schema.prisma` and count how many distinct feature areas query each of the top five tables. That is your real coupling.
3. Draw the actual package dependency graph (`turbo run build --graph`) and diff it against your intended context map.
4. Find your deepest Server Action or page: how many distinct concerns does one user click touch? Anything past four wants a use case.
5. Ask what happens today if Knock, Resend, or your search provider is down during an access approval. If the answer is "the approval fails", you have found your first outbox candidate.
6. Ask how many files you would have to change to rename one column in your busiest table. If nobody can answer without grepping, the schema is your contract.

Then pick two things and fix them. Not all of it — two. And add the ratchet so they stay fixed.

## Closing

next-forge gives you a production-grade delivery platform and no domain architecture, which is the correct division of labour for a template. The mistake is reading the absence as a recommendation and letting `database.page.findMany()` in a page component become the house style at forty models.

You do not need to leave the template, split the deployment, or adopt every DDD pattern in the blue book. You need a context tier in `packages/`, one public entry point per context, tables owned by exactly one context, the enforcement Turborepo and Biome already ship, and a deliberate answer to "must this be synchronous?" at each boundary.

Do that and you keep everything the monolith is genuinely good at — one deploy, nanosecond calls, a compiler that catches your mistakes, refactoring across boundaries as a rename rather than a migration — while the domain stays somewhere you can find it.

---

*Sources: [next-forge](https://github.com/vercel/next-forge) (structure verified against v6.0.2), [next-forge docs](https://www.next-forge.com/docs), [Turborepo Boundaries](https://turborepo.dev/docs/reference/boundaries), [Biome `noRestrictedImports`](https://biomejs.dev/linter/rules/no-restricted-imports/), [Prisma schema location](https://www.prisma.io/docs/orm/prisma-schema/overview/location). Companion to [Keep The Monolith, Keep It Well](KeepTheMonolith.md).*
