# Tech Lead Interview Prep — Architecture Deep Dive

> Topics: Monolith vs Microservices · Monorepo vs Polyrepo · Turborepo · Nx · Distributed Transactions · Modular Monolith

---

## Table of Contents

1. [Two Dimensions — Code vs Runtime](#1-two-dimensions)
2. [Monolith](#2-monolith)
3. [Microservices](#3-microservices)
4. [Monorepo vs Polyrepo](#4-monorepo-vs-polyrepo)
5. [Turborepo](#5-turborepo)
6. [Nx vs Turborepo](#6-nx-vs-turborepo)
7. [Distributed Transactions in Microservices](#7-distributed-transactions-in-microservices)
8. [Modular Monolith Pattern](#8-modular-monolith-pattern)
9. [Explaining Your Architecture in Interviews](#9-explaining-your-architecture-in-interviews)
10. [The Big Picture Mental Model](#10-the-big-picture-mental-model)

---

## 1. Two Dimensions

People often confuse these concepts because they operate on **two different axes**:

| Dimension | Options |
|---|---|
| **Code organization** | Monorepo vs Polyrepo |
| **Runtime architecture** | Monolith vs Microservices |

> Turborepo and Nx are **build tools for monorepos** — not architectures themselves. You can have a monorepo with a monolith, or a monorepo with microservices.

---

## 2. Monolith

A single deployable unit where all features live in one codebase and run as one process.

### Pros
- Simple to develop and debug early on
- No network overhead between modules
- Easy transactions — everything is in-process
- Straightforward deployment

### Cons
- Hard to scale specific parts independently as it grows
- A bug in one module can crash everything
- Deployment risk increases — changing one thing requires redeploying everything
- Team scaling becomes painful (merge conflicts, tight coupling)

### When It Makes Sense
Early-stage startups, small teams, or internal tools where speed matters more than scale.

---

## 3. Microservices

The application is split into independently deployable services, each owning a specific domain (auth, payments, notifications, etc.).

### Pros
- Scale individual services independently
- Teams can own and deploy services independently
- Fault isolation — one service failing doesn't bring down everything
- Freedom to use different tech per service

### Cons
- Network latency between services
- Distributed transactions are hard (saga pattern, eventual consistency)
- Operational overhead — managing 15 deployments instead of 1
- Debugging across services requires distributed tracing

### When It Makes Sense
When you have clear domain boundaries, multiple teams, and genuine scale requirements.

---

## 4. Monorepo vs Polyrepo

This is about **where your code lives**, not how it runs.

### Polyrepo
Each service/app lives in its own repository.

**Pros:** Clear ownership, independent versioning, isolated CI/CD

**Cons:** Sharing code is painful (npm packages, versioning hell), cross-cutting changes require PRs across multiple repos

### Monorepo
All code lives in one repository — but each app/service is still independently deployable.

**Pros:**
- Shared packages without publishing to npm
- Atomic commits across multiple apps
- Easier to refactor shared code
- Single place to enforce linting and testing standards
- Visibility across the whole codebase

**Cons:**
- Slow CI if not optimized (running all tests for every change)
- Requires tooling discipline
- Git history can get noisy

---

## 5. Turborepo

A **high-performance build system for JavaScript/TypeScript monorepos**. Solves the biggest pain point of monorepos — slow builds.

### What It Does
- **Task caching** — if nothing changed in a package, it skips rebuilding (local + remote cache)
- **Parallel execution** — runs tasks across packages concurrently based on dependency graph
- **Affected-only runs** — only runs tests/builds for packages affected by a change
- **Pipeline definition** — task dependencies declared in `turbo.json`

### Basic `turbo.json` Config

```json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".next/**", "dist/**"]
    },
    "test": {
      "dependsOn": ["build"],
      "outputs": []
    },
    "lint": {
      "outputs": []
    },
    "dev": {
      "cache": false,
      "persistent": true
    }
  }
}
```

> `^build` means "run `build` in all dependencies of this package before running my `build`".

### Key Optimizations

**Remote Caching** — biggest win for teams:
```bash
npx turbo login
npx turbo link
```

**Filtering — run only what you need:**
```bash
# Only build packages affected by changes since main
turbo build --filter=...[origin/main]

# Only run for a specific app and its dependencies
turbo build --filter=@playleap/web...

# Exclude a package
turbo build --filter=!@playleap/legacy-app
```

**Pruning for Docker — keeps images lean:**
```bash
turbo prune --scope=@playleap/auth-service --docker
```

### Common Pitfalls
- Not declaring all `outputs` correctly → cache misses
- Forgetting `"cache": false` on `dev` → stale dev servers
- Not using `--filter` in CI → rebuilding everything on every PR

---

## 6. Nx vs Turborepo

| | **Turborepo** | **Nx** |
|---|---|---|
| **Learning curve** | Low — simple config | Higher — more concepts |
| **Language support** | JS/TS focused | JS, Go, Java, Python, .NET |
| **Build caching** | ✅ Local + Remote | ✅ Local + Remote (Nx Cloud) |
| **Code generation** | ❌ None built-in | ✅ Powerful generators & schematics |
| **Affected detection** | ✅ File-based | ✅ More sophisticated (project graph) |
| **Plugin ecosystem** | Small but growing | Very rich (React, NestJS, Next.js, etc.) |
| **Enforced boundaries** | ❌ Manual | ✅ Module boundary rules built-in |
| **Best for** | Frontend-heavy JS monorepos | Large orgs, mixed tech, strict boundaries |

### When to Choose Turborepo
- Your stack is primarily JS/TS
- You want minimal config and fast onboarding
- You don't need code generation
- Frontend-heavy monorepos (Expo + Next.js + Node)

### When to Choose Nx
- You have multiple backend languages
- You want enforced module boundaries with linting rules
- You want code generators for scaffolding NestJS modules, React components, etc.
- You have a large team and need governance tooling

### Nx Module Boundary Enforcement

```json
// project.json — tag your packages
{
  "tags": ["scope:auth", "type:service"]
}
```

```json
// .eslintrc — enforce what can depend on what
{
  "depConstraints": [
    { "sourceTag": "scope:auth", "onlyDependOnLibsWithTags": ["scope:shared"] }
  ]
}
```

> This prevents `auth` module from accidentally importing from `payments` module. ESLint will throw an error.

---

## 7. Distributed Transactions in Microservices

One of the **hardest problems** in microservices and a favourite interview topic.

### The Problem

In a monolith, transactions are simple:
```js
await db.transaction(async (trx) => {
  await createOrder(trx);
  await deductInventory(trx);
  await chargePayment(trx);
}); // if any fails, all rollback
```

In microservices, each service has its **own database**. If `chargePayment` fails after `createOrder` already committed — you have inconsistent data.

---

### Solution 1: Saga Pattern ⭐ (Most Common)

A saga is a sequence of local transactions. If one step fails, **compensating transactions** undo the previous steps.

#### A) Choreography-based Saga (event-driven)

Each service publishes events and reacts to events from other services. No central coordinator.

**Happy path:**
```
OrderService        → publishes "OrderCreated"
InventoryService    → listens → reserves stock → publishes "StockReserved"
PaymentService      → listens → charges card   → publishes "PaymentDone"
OrderService        → listens → marks order confirmed
```

**Failure path:**
```
PaymentService      → publishes "PaymentFailed"
InventoryService    → listens → releases stock
OrderService        → listens → cancels order
```

✅ Loose coupling, no single point of failure  
❌ Hard to track overall flow, easy to create circular events

---

#### B) Orchestration-based Saga (centralized)

A dedicated orchestrator service tells each service what to do next.

```
SagaOrchestrator:
  1. Call OrderService      → create order
  2. Call InventoryService  → reserve stock
  3. Call PaymentService    → charge card
  4. If any fails           → call compensating APIs in reverse
```

✅ Easy to visualize and debug, single source of truth  
❌ Orchestrator becomes a bottleneck, coupling to it

---

### Solution 2: Outbox Pattern (Reliable Event Publishing)

**The problem:** what if your service saves to DB but crashes before publishing the event?

```js
// DANGEROUS — not atomic
await db.save(order);
await eventBus.publish('OrderCreated', order); // crash here = lost event
```

**The fix — Outbox Pattern:**
```js
// SAFE — atomic
await db.transaction(async (trx) => {
  await saveOrder(trx);
  await saveToOutboxTable(trx, { event: 'OrderCreated', payload: order });
});
// Separate process polls outbox table and publishes events
// Once published, mark as processed
```

The outbox table acts as a reliable queue within your own DB. Guarantees at-least-once delivery.

---

### Solution 3: Two-Phase Commit (2PC) — Avoid in Practice

Theoretically correct but operationally painful. Requires all services to lock resources during a coordinator-driven commit. Too slow and too fragile for production microservices.

---

### The Interview Answer

> "I avoid distributed transactions where possible by designing services around bounded contexts so cross-service writes are rare. When unavoidable, I use the Saga pattern — choreography for simple flows, orchestration for complex ones. I also use the Outbox pattern to guarantee event delivery. The key is accepting **eventual consistency** rather than fighting for strong consistency across service boundaries."

---

## 8. Modular Monolith Pattern

The **underrated middle ground** — a single deployable application internally structured with strict module boundaries, like microservices, but without the network overhead or operational complexity.

### Structure

```
src/
  modules/
    auth/
      auth.controller.ts
      auth.service.ts
      auth.repository.ts   ← only auth touches this DB table
    orders/
      orders.controller.ts
      orders.service.ts
      orders.repository.ts
    notifications/
      ...
```

### The Rules (Must Enforce)

1. **No cross-module direct DB access** — `OrdersModule` never queries the `users` table directly. It calls `AuthModule`'s service or an internal event.
2. **Communication via interfaces, not internals** — modules expose a public API (service methods or internal events). They never reach into each other's repositories.
3. **Clear bounded contexts** — each module owns its domain completely.

### Why It's Powerful

- You get microservice-like boundaries without network calls, distributed tracing, or separate deployments
- When you do need to extract a service, the boundary is already clean — mostly a copy-paste + adding an HTTP/message interface
- Much easier to debug — one process, one log stream, one deployment
- Transactions still work normally within a module

### In NestJS (Built for This Pattern)

```ts
@Module({
  imports: [TypeOrmModule.forFeature([Order])],
  controllers: [OrdersController],
  providers: [OrdersService, OrdersRepository],
  exports: [OrdersService], // only this is accessible externally
})
export class OrdersModule {}
```

Modules explicitly declare what they export. Nothing else is accessible. This is the modular monolith pattern baked into the framework.

### Modular Monolith vs Microservices Decision

| | **Modular Monolith** | **Microservices** |
|---|---|---|
| Team size | Small–medium | Large, multiple teams |
| Deployment | Single unit | Independent per service |
| Transactions | Simple (in-process) | Complex (Saga, Outbox) |
| Debugging | Easy | Needs distributed tracing |
| Scaling | Scale whole app | Scale individual services |
| Starting point? | ✅ Yes | Only when boundaries are clear |

---

## 9. Explaining Your Architecture in Interviews

Here's a structured 5-minute walkthrough template using a sports platform as example.

### The Setup Line
> "It's a white-label sports tech platform. The core challenge is serving multiple sports organisations — each with their own branding, feature toggles, and configurations — while sharing the same underlying infrastructure."

### Code Organization
> "We use a **monorepo with Turborepo** to manage everything in one place — multiple Expo apps for iOS/Android, Next.js for web, and shared packages like UI components, hooks, and utilities. Turborepo gives us task caching and parallel builds so we're not rebuilding what hasn't changed."

### Backend Architecture
> "The backend is **15+ microservices**, each owning a specific domain — auth, user profiles, match data, notifications, media, etc. We're now introducing **NestJS with Fastify** for new services where performance and developer experience matter more."

### The Interesting Problems (Pick 1–2)

**API backward compatibility:**
> "One challenge we solved was API backward compatibility when we moved endpoints between services. We maintained the old routes as proxies temporarily while migrating clients, which let us move fast without breaking existing app versions in the wild."

**White-labelling at scale:**
> "Rather than forking the codebase per client, we built a configuration layer where theme tokens, feature flags, and content are injected at build time and runtime — keeping one codebase for all clients."

### What You'd Do Differently (Shows Maturity)
> "If I were starting fresh, I'd use a **modular monolith** initially rather than jumping straight to many microservices. The operational overhead of managing that many services is significant, and some services are too fine-grained. I'd consolidate related domains and only split when there's a genuine scaling or team ownership reason."

---

## 10. The Big Picture Mental Model

How all these concepts connect:

```
Codebase (Monorepo managed by Turborepo or Nx)
    │
    ├── Apps (Expo, Next.js)
    ├── Shared packages (ui, utils, types)
    └── Services
          ├── Modular Monolith (NestJS modules with strict boundaries)
          │     → evolves into microservices when needed
          └── Microservices (LoopBack 4 / NestJS + Fastify)
                → communicate via events (Saga pattern)
                → use Outbox pattern for reliability
                → accept eventual consistency
```

### The Tech Lead Mindset

Never pick a side — always talk about tradeoffs:

> "It depends on team size, domain clarity, and scale requirements. I'd start with a **modular monolith** to move fast and extract services only when a specific domain has clear boundaries and independent scaling needs."

---

## Quick Reference — When to Use What

| Scenario | Recommendation |
|---|---|
| Early-stage product, small team | Modular Monolith |
| Multiple teams, clear domain ownership | Microservices |
| JS/TS monorepo, frontend-heavy | Turborepo |
| Large org, mixed tech, strict boundaries needed | Nx |
| Cross-service writes needed | Saga Pattern (Orchestration) |
| Reliable event publishing | Outbox Pattern |
| Microservices, but still collocated code | Monorepo + Turborepo/Nx |

---

*Last updated: March 2026*
