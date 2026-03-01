await db.transaction(async (trx) => {
  await createOrder(trx);
  await deductInventory(trx);
  await chargePayment(trx);
}); // if any fails, all rollback
```

In microservices, each service has its **own database**. There's no shared transaction. If `chargePayment` service fails after `createOrder` already committed — you have inconsistent data.

---

### Solution 1: Saga Pattern ⭐ (Most Common)

A saga is a sequence of local transactions. If one step fails, compensating transactions undo the previous steps.

**Two flavours:**

**A) Choreography-based Saga** (event-driven)
Each service publishes events and reacts to events from other services. No central coordinator.
```
OrderService → publishes "OrderCreated"
  → InventoryService listens → reserves stock → publishes "StockReserved"
  → PaymentService listens → charges card → publishes "PaymentDone"
  → OrderService listens → marks order confirmed
```

If payment fails:
```
PaymentService → publishes "PaymentFailed"
  → InventoryService listens → releases stock
  → OrderService listens → cancels order
```

✅ Loose coupling, no single point of failure
❌ Hard to track overall flow, easy to create circular events

---

**B) Orchestration-based Saga** (centralized)
A dedicated orchestrator service tells each service what to do next.
```
SagaOrchestrator:
  1. Call OrderService → create order
  2. Call InventoryService → reserve stock
  3. Call PaymentService → charge
  4. If any fails → call compensating APIs in reverse
```

✅ Easy to visualize and debug, single source of truth
❌ Orchestrator becomes a bottleneck, coupling to it

---

### Solution 2: Outbox Pattern (Reliable Event Publishing)

A subtle but critical problem: what if your service saves to DB but then crashes before publishing the event?
```
// DANGEROUS - not atomic
await db.save(order);
await eventBus.publish('OrderCreated', order); // crash here = lost event
```

The **Outbox Pattern** solves this:
```
// SAFE - atomic
await db.transaction(async (trx) => {
  await saveOrder(trx);
  await saveToOutboxTable(trx, { event: 'OrderCreated', payload: order });
});
// Separate process polls outbox table and publishes events
// Once published, mark as processed
```

The outbox table acts as a reliable queue within your own DB.

---

### Solution 3: Two-Phase Commit (2PC) — Avoid in Practice

Theoretically correct but operationally painful. Requires all services to lock resources during a coordinator-driven commit. Too slow, too fragile for production microservices.

---

### The Interview Answer

> "I avoid distributed transactions where possible by designing services around bounded contexts so cross-service writes are rare. When unavoidable, I use the Saga pattern — choreography for simple flows, orchestration for complex ones. I also use the Outbox pattern to guarantee event delivery. The key is accepting **eventual consistency** rather than fighting for strong consistency across service boundaries."

---

---

## 2. 🏗️ Explaining PlayLeap's Architecture for Interviews

Here's a structured way to present it — use this as your 5-minute walkthrough.

### The Setup Line
> "PlayLeap is a white-label sports tech platform. The core challenge is that we serve multiple sports organisations — each with their own branding, feature toggles, and configurations — while sharing the same underlying infrastructure."

### Code Organization
> "We use a **monorepo with Turborepo** to manage everything in one place — multiple Expo apps for iOS/Android, Next.js for web, and shared packages like UI components, hooks, and utilities. Turborepo gives us task caching and parallel builds so we're not rebuilding what hasn't changed."

### Backend Architecture
> "The backend is **15+ microservices built in LoopBack 4**, each owning a specific domain — auth, user profiles, match data, notifications, media, etc. We're now introducing **NestJS with Fastify** for new services where performance and developer experience matter more."

### The Interesting Problems (pick 1-2)
> "One challenge we solved was **API backward compatibility** when we moved endpoints between services. We maintained the old routes as proxies temporarily while migrating clients, which let us move fast without breaking existing app versions in the wild."

> "Another was **white-labelling at scale** — rather than forking the codebase per client, we built a configuration layer where theme tokens, feature flags, and content are injected at build time and runtime, keeping one codebase for all clients."

### What You'd Do Differently
Always include this — it shows maturity:
> "If I were starting fresh, I'd use a **modular monolith** initially rather than jumping straight to 15 microservices. The operational overhead of managing that many services is significant, and some of our services are too fine-grained. I'd consolidate related domains and only split when there's a genuine scaling or team ownership reason."

---

---

## 3. 🧱 Modular Monolith Pattern

This is the **underrated middle ground** that many experienced engineers now prefer as a starting point.

### What It Is
A single deployable application internally structured with strict module boundaries — like microservices, but without the network overhead or operational complexity.
```
src/
  modules/
    auth/
      auth.controller.ts
      auth.service.ts
      auth.repository.ts  ← only auth touches this DB table
    orders/
      orders.controller.ts
      orders.service.ts
      orders.repository.ts
    notifications/
      ...
