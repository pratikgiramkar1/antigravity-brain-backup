# Part 5 — Interview Preparation

> Covers **Phase 14** (Interview Questions), **Phase 15** (Knowledge Map), **Phase 16** (Interview Stories), and **Phase 17** (Consolidated Summary)

---

## 1. Interview Questions by Level (Phase 14)

### Level 1 — Fundamentals (Warm-up)

| # | Question | Key Points to Hit |
|---|----------|-------------------|
| 1 | **What does Vayu/Breeze do?** | One-click checkout SDK. Embeds on merchant sites. Handles login → address → offers → payment → order creation. |
| 2 | **What's the tech stack?** | Haskell, Servant (type-safe HTTP), Beam (type-safe SQL), PostgreSQL, Redis, EulerHS framework. |
| 3 | **How many platforms do you support?** | 7: Shopify, WooCommerce, Magento, VTEX, CS-Cart, PrestaShop, Independent. Pattern-matched via `ShopType` enum. |
| 4 | **How is the codebase organized?** | Three layers: Routes/Core.hs → Product/ (orchestration) → Services/ (Internal for domain logic, External for API calls). |
| 5 | **What's the code generation pipeline?** | OpenAPI YAML specs → Mustache templates → openapi-generator-cli → 2100+ Haskell type files, 75 query files, 39 network call files. Never manually edit generated code. |
| 6 | **What's ProcessTracker?** | DB-backed distributed task scheduler. Producer polls DB for eligible tasks, sends batches to Consumer via HTTP. Handles order reconciliation, abandonment, analytics. |
| 7 | **How is config managed?** | Three tiers: Shop Config (DB, per-merchant) → Global Config (Redis, all shops) → Env Config (startup, immutable). Non-null at higher tier wins. |
| 8 | **What databases do you use?** | PostgreSQL (75+ tables, ACID for financial data) + Redis (caching, locking, queues, sessions). |

### Level 2 — Architecture & Design (Most Asked)

| # | Question | Key Points to Hit |
|---|----------|-------------------|
| 9 | **Walk me through a checkout request.** | Cart creation → OTP login → address selection → offer application → start payment → Euler SDK → webhook/poll → order reconciliation → platform order creation. See Part 2 trace. |
| 10 | **How do you handle multiple e-commerce platforms?** | ADT pattern matching on `ShopType` in `Services/Internal/Platform/`. Compiler checks exhaustiveness. Adding a platform = implementing ~8 dispatch functions. |
| 11 | **Why code generation instead of manual types?** | 75+ tables, 200+ endpoints. Manual sync is error-prone. YAML is the single source of truth. Compiler catches all downstream breakage after regeneration. |
| 12 | **How does the payment flow work?** | Build HyperPay payload → frontend initiates payment via Euler SDK → Euler processes → webhook to Vayu or PT polls status → update order → create platform order. |
| 13 | **Explain your Product/Services layer separation.** | Product: orchestration, logging, auth, async spawning. Services/Internal: domain logic, DB queries. Services/External: API calls ONLY (no DB, no logic). Known violations exist in legacy code. |
| 14 | **How do you add a new API endpoint?** | 1. Add path in `doc/paths/X.yaml`. 2. Add schemas in `doc/schemas/X.yaml`. 3. Regenerate. 4. Implement handler in `Core.hs`. 5. Call Product layer. |
| 15 | **What's the difference between platform offers and Euler offers?** | Platform: backend applies via platform API, stored immediately. Euler: backend lists only, frontend applies via SDK, stored after payment succeeds. |
| 16 | **How do you handle feature flags?** | Shop-level config flags + OpenFeature (Superposition). Dual implementations coexist (e.g., `useLoginV2Flow`). |

### Level 3 — Distributed Systems (SDE2 Core)

| # | Question | Key Points to Hit |
|---|----------|-------------------|
| 17 | **How does your distributed task scheduler work?** | Producer (single instance, Redis lock) polls DB every 10s for NEW/PENDING tasks in a time window. Batches them. Sends via HTTP to Consumer (horizontal pods). Consumer acquires per-task Redis lock, executes workflow, updates DB. |
| 18 | **How do you prevent duplicate task execution?** | Three mechanisms: deterministic task IDs (DB unique constraint), consumer Redis locks (`pt:consumer:{taskId}`), and workflow state locks (`state:cart:{cartId}`). |
| 19 | **Explain exponential backoff in your system.** | `offset = backoffFactor^(retryCount+1) × 60s`. Default: 2min → 4min → 8min → 16min → 32min. Per-shop configurable via `orderReconConfig`. Max retries exceeded → auto-cancel + abandonment workflow. |
| 20 | **What happens if the ProcessTracker producer crashes?** | Redis lock (`PRODUCER_LOCK`, TTL=1000s) auto-expires. Next instance acquires it. In-flight consumer tasks continue independently. Unfinished tasks remain in NEW/PENDING → picked up on next cycle. |
| 21 | **How does the Supermoney master-worker pattern work?** | Master: queries pending txns, batches by 100, pushes to Redis queue, spawns N workers. Workers: pop from queue, call bulk API, update statuses, retry on failure. Crash recovery via processing keys. |
| 22 | **What's your crash recovery strategy for batch processing?** | Before processing: store batch in `processing:{queue}:{batchId}` key (TTL 1h). After: delete key. On crash: next master run scans for orphaned processing keys, re-queues them. |
| 23 | **How do you handle rolling deployments with enum changes?** | Two-step query: try without enum filter, if parse fails (unknown runner), retry with `WHERE runner IN (known values)`. Logs `UNKNOWN_RUNNER_ENUM_DETECTED` alert. |
| 24 | **What are the failure modes of your distributed locking?** | Redis failure → all locks lost (tasks may double-execute, mitigated by idempotent IDs). Lock not released → TTL auto-expires. Split brain → SETNX is atomic, only one holder. |

### Level 4 — Deep Technical (Senior SDE2)

| # | Question | Key Points to Hit |
|---|----------|-------------------|
| 25 | **Why not use Kafka/RabbitMQ instead of ProcessTracker?** | PT uses PostgreSQL — no extra infrastructure. Built-in retry, backoff, cascading, time-window filtering. Tradeoff: polling (not event-driven), unbounded table growth. |
| 26 | **How would you scale the database?** | Read replicas for read-heavy queries (order status, shipping rules). Shard by shopId if needed. ProcessTracker table needs periodic cleanup (external cron). |
| 27 | **What's the consistency model?** | Eventual consistency between Breeze order and platform order. PT ensures reconciliation. Cart price updates are synchronous (immediate consistency within Breeze). |
| 28 | **How do you handle payment atomicity?** | Payment is handled by Euler (external). Vayu's role is reconciliation. If payment succeeds but platform order fails, PT retries. If both fail, auto-cancel + abandonment notification. |
| 29 | **What would break if Redis went down?** | All locks lost (duplicate PT execution), OTP login fails, auth token cache misses (→ re-fetch), offer caching fails (→ re-compute), in-progress dedup fails (→ duplicate requests possible). |
| 30 | **How does the three-tier config system handle race conditions?** | Global config (Redis) is eventually consistent — SET is atomic. Shop config (DB) updates are per-row atomic. Config reads are non-transactional (stale reads possible during update). Mitigated by cache invalidation via Pub/Sub. |
| 31 | **Why are monetary values in paisa/cents?** | Avoids floating-point precision errors. `Scientific` type in Haskell gives arbitrary precision. All amounts are integers in smallest currency unit. |
| 32 | **How do you handle partial payments?** | `cart.amountPaid` tracks prepaid portion. `cart.partialPaymentRuleId` links to the rule. Supermoney sync derives `payment_mode = "PARTIAL_PAYMENT"` with split amounts. |

### Level 5 — System Design & Leadership (Staff-level)

| # | Question | Key Points to Hit |
|---|----------|-------------------|
| 33 | **If you were redesigning this system, what would you change?** | See Part 4, Section 6. Key: Temporal for workflows, DB transactions for writes, OpenTelemetry for observability, property-based testing. Keep: code gen, type-safe SQL, monolith (for this team size). |
| 34 | **How would you add a new e-commerce platform (e.g., BigCommerce)?** | 1. Add `BigCommerce` to `ShopType` enum. 2. Compiler errors show every dispatch function. 3. Implement ~8 External modules (Cart, Order, Offer, etc.). 4. Add platform-specific config. 5. Register in Platform dispatch modules. |
| 35 | **How would you handle 10x traffic increase?** | Horizontal scale API pods. Add PostgreSQL read replicas. Redis cluster mode. Increase PT batch size. Split heavy tables (processTracker, offer) by age/shopId. |
| 36 | **What's the blast radius of a bad deployment?** | Entire system — it's a monolith. Mitigated by: feature flags, canary deployments, enum resilience, dual implementations (old + new behind flags). |
| 37 | **How would you decompose this into microservices?** | Domain boundaries: Payment, Order, Notification, Identity, Shipping. Shared DB → event-driven. Start with strangling the monolith (extract Notification first — it's already a separate concern). |

---

## 2. Knowledge Map (Phase 15)

### Core Concepts Index

| Concept | Where to Find in Vayu | Interview Relevance |
|---------|----------------------|---------------------|
| **Distributed Task Scheduling** | ProcessTracker (Part 3) | HIGH — "Design a distributed job scheduler" |
| **Master-Worker Pattern** | Supermoney sync (Part 3) | HIGH — "Design a batch processing system" |
| **Distributed Locking** | Redis SETNX, three levels (Part 3) | HIGH — "How do you prevent duplicate execution?" |
| **Exponential Backoff** | PT retry config (Part 3) | MEDIUM — "How do you handle retries?" |
| **Idempotency** | Deterministic task IDs, sync status (Part 3) | HIGH — "How do you make operations idempotent?" |
| **Event-Driven Architecture** | PT cascading tasks, Kafka logging (Part 2) | MEDIUM — "How do you handle async workflows?" |
| **Multi-Tenant Config** | Three-tier config system (Part 1) | MEDIUM — "How do you handle per-customer settings?" |
| **Code Generation** | YAML → Mustache → Haskell (Part 1) | MEDIUM — "How do you keep API contracts consistent?" |
| **Type-Safe API Design** | Servant routes (Part 1) | MEDIUM — "How do you ensure API correctness?" |
| **Crash Recovery** | Processing keys, bracket pattern (Part 3) | HIGH — "How do you handle failures in batch processing?" |
| **Platform Abstraction** | ShopType dispatch (Part 1) | MEDIUM — "How do you support multiple platforms?" |
| **OAuth 2.0 Token Management** | Supermoney Auth (cache + encrypt + TTL) | MEDIUM — "How do you manage third-party auth?" |
| **Data Pipeline** | Cart → Order → Transaction → Supermoney (Part 2) | HIGH — "Walk me through the data flow" |
| **Graceful Shutdown** | MVar + SIGTERM + bracket (Part 3) | LOW — Good to mention in reliability discussions |

---

## 3. Interview Stories (Phase 16)

### Story 1: Supermoney Order Reconciliation Pipeline

**STAR Format for Behavioral Interviews**

**Situation:**
"At Breeze, we had a partnership with Supermoney where we needed to sync all successful payment transactions to their billing system daily. Initially, there was no automated system — someone had to manually export and upload data. With thousands of transactions per day across hundreds of merchants, this was unsustainable."

**Task:**
"I was tasked with designing and implementing an automated, reliable batch sync pipeline. The key requirements were:
1. Process all pending transactions daily, no data loss
2. Handle API failures gracefully (Supermoney had rate limits)
3. Recover from crashes without manual intervention
4. Alert the team on persistent failures"

**Action:**
"I designed a **Master-Worker architecture** using our existing ProcessTracker (distributed task scheduler):

**Master Workflow:**
1. Runs daily via ProcessTracker cascading tasks (self-scheduling)
2. First, it recovers any crashed-in-progress batches from Redis processing keys
3. Queries ALL transactions with `supermoneySyncStatus = PENDING AND status = SUCCESS`
4. Batches them into groups of 100 (for Supermoney's bulk API limit)
5. Pushes each batch to a Redis queue (`RPUSH`)
6. Spawns N worker ProcessTracker tasks (configurable via env var)
7. Creates a cascading task for the next day (self-perpetuating)

**Worker Workflow:**
1. Pops batches from the Redis queue
2. Before processing, stores the batch in a `processing:` Redis key (crash recovery)
3. Fetches full transaction data via a 3-way JOIN (Transaction × Order × Cart)
4. Transforms each record: derives payment mode (COD/PREPAID/PARTIAL), calculates split amounts
5. Calls Supermoney's bulk debit API
6. Updates each transaction's `supermoneySyncStatus` to COMPLETED or FAILED
7. On failure: re-queues with incremented attempt count. After 2 attempts: sends HTML email alert with a CSV attachment of failed transactions

**Crash Recovery:**
The critical innovation was the crash recovery pattern. If a worker dies mid-batch, the `processing:` key survives in Redis (1-hour TTL). Next master run detects orphaned processing keys and re-queues those batches. Zero data loss."

**Result:**
"The pipeline processed thousands of transactions daily without manual intervention. The crash recovery pattern ensured zero data loss even during pod restarts. The email alerting with CSV attachments gave the ops team immediate visibility into failures — they could investigate specific transactions instead of guessing."

**Follow-up Questions They Might Ask:**

| Question | Your Answer |
|----------|-------------|
| "Why Redis queue instead of Kafka?" | "We already had Redis in our infrastructure. The queue didn't need persistence beyond 1 hour (processing TTL). Kafka would've been overkill for a daily batch job." |
| "What if Redis itself crashes?" | "Good question. If Redis crashes between RPUSH and worker processing, the master's next run re-queries the DB (transactions still PENDING), re-batches, and re-queues. The DB is the source of truth, Redis is just the coordination layer." |
| "Why batch size of 100?" | "Supermoney's bulk API had a recommended limit. We could tune this via config. With 100 transactions per batch and 2 workers, we process 200 transactions concurrently." |
| "How do you prevent double-sync?" | "The `supermoneySyncStatus` field acts as an idempotency guard. Once set to COMPLETED, the transaction is excluded from future queries. Even if a batch is processed twice (due to retry), the API call is idempotent because we send the same `partner_order_id`." |

---

### Story 2: Magento Integration (End-to-End Platform)

**STAR Format**

**Situation:**
"When I joined the Breeze project, we supported Shopify and WooCommerce. There was a business need to support Magento — a major e-commerce platform used by enterprise merchants. This wasn't just adding a few API calls; it required building an entire platform integration from scratch."

**Task:**
"I was responsible for the end-to-end Magento integration. This meant implementing every checkout flow for Magento: cart creation, address management, customer management, shipping estimation, coupon handling, order creation, inventory verification, and analytics tracking."

**Action:**
"I built 15 modules under `Services/External/Magento/`:

1. **Session Management** — Magento uses session-based auth (not API keys like Shopify). I built a `MagentoSession` table to persist session tokens mapped to Breeze cart IDs.

2. **Cart Operations** (`Cart.hs`) — Creating carts on Magento via their REST API, mapping Breeze cart items to Magento's cart item format. Magento's API requires separate calls for guest vs authenticated carts.

3. **Order Creation** (`Order.hs`) — The most complex module. Magento requires:
   - Setting the shipping method before creating the order
   - Applying payment method info (COD, prepaid, partial)
   - Handling CPO wallet amounts as discounts
   - Creating invoices for prepaid orders

4. **Coupon System** (`Coupon.hs`) — Magento's coupon API is different from Shopify's. I had to apply coupons, recalculate cart totals including tax adjustments, and build the offer response.

5. **Shipping** (`Shipping.hs`) — Address validation + shipping estimate in one call (Magento combines them). Country and region handling was tricky because Magento uses region IDs, not names.

6. **Analytics** (`Tracker.hs`) — Extracting line items from Magento sessions for Facebook CAPI and GA4 analytics.

Each module implemented the functions required by the platform dispatch layer (`Services/Internal/Platform/`), so the Product layer didn't need to know it was talking to Magento."

**Result:**
"We successfully launched Magento support, enabling enterprise merchants to use Breeze. The integration handled all checkout flows including edge cases like partial payments, COD orders, and custom payment options.

One learning: I initially put some DB queries and business logic directly in the External Magento modules (e.g., `Coupon.hs` contains `modifyOffersAndCart`). This was expedient for the launch but became documented tech debt — it violated our architecture rule that External services should only make API calls. In hindsight, I'd structure the discount calculation in Internal/Offer and pass results to the External module."

**Follow-up Questions:**

| Question | Your Answer |
|----------|-------------|
| "What was the hardest part?" | "Session management. Unlike Shopify (stateless API keys), Magento requires maintaining session state across requests. I designed the `MagentoSession` table to persist session tokens and map them to Breeze cart IDs." |
| "How did you test it?" | "Manual testing against a staging Magento instance. The CATS framework covered contract validation. We didn't have automated integration tests — that's a gap I'd address today." |
| "Why are there architecture violations in the Magento code?" | "Launch pressure. We needed to ship fast. The violations (DB queries in External modules) worked correctly but broke the layered architecture. I'd refactor them today — the discount calculation should move to Internal/Offer." |

---

## 4. Consolidated Quick-Reference (Phase 17)

### One-Minute System Description

> "Vayu is the Haskell backend for Breeze, a one-click checkout SDK embedded on e-commerce websites. It supports 7 e-commerce platforms, 18 payment providers, and handles the complete checkout lifecycle: login, address, offers, payment, and order creation. The architecture is a three-layer monolith (Routes → Product → Services) with 60% auto-generated code from OpenAPI specs. It uses PostgreSQL for ACID data storage, Redis for distributed locking and caching, and a custom ProcessTracker for async workflow management."

### Architecture Cheat Sheet

```
Client (Breeze SDK on merchant site)
  ↓ HTTPS
Warp HTTP Server + Middleware Chain
  ↓
Servant Type-Safe Router (auto-generated API types)
  ↓
Core.hs (6647 lines — route handlers, auth dispatch)
  ↓
Product Layer (57 modules — orchestration, logging, async)
  ↓
Services/Internal (domain logic, Beam SQL, Redis)
Services/External (71+ 3rd-party API integrations)
  ↓
PostgreSQL (75+ tables) + Redis (locks, cache, queues)
```

### Five Things to Memorize

1. **Code generation pipeline**: YAML specs → Mustache → Haskell types + queries + API routes. ~2100 generated files. Never edit generated code.

2. **Three-layer architecture**: Product (orchestration) → Internal (domain logic + DB) → External (API calls only). Known violations in Shopify/Magento/Euler modules.

3. **ProcessTracker**: PostgreSQL-backed distributed task scheduler. Producer (single instance, Redis lock) → Consumer (horizontal, load balanced). 21 active workflows. Exponential backoff. Cascading tasks.

4. **Supermoney Master-Worker**: Master queries DB → batches by 100 → Redis queue → N workers. Crash recovery via processing keys. Two-attempt retry with email alerting.

5. **Three-tier config**: Shop Config (DB) → Global Config (Redis) → Env Config (startup). Non-null wins. 200+ feature flags without deployment.

### Behavioral Talking Points

| Theme | Your Example |
|-------|-------------|
| **Ownership** | "I owned the entire Magento integration end-to-end — from API research to production deployment across 15 modules." |
| **System Design** | "I designed the Supermoney master-worker architecture, including the crash recovery mechanism using Redis processing keys." |
| **Reliability** | "My crash recovery design ensures zero data loss even during pod restarts — orphaned batches are automatically re-queued." |
| **Tech Debt Awareness** | "I recognize that my early Magento code has architecture violations. I documented them and would refactor the DB queries out of External modules." |
| **Scale Thinking** | "The worker count is configurable. With more workers, we process more batches in parallel. The Redis queue acts as a natural backpressure mechanism." |
| **Testing** | "I added Haddock documentation to every function in the Supermoney module and ensured comprehensive error handling with email alerts for visibility." |

### Common Interviewer Traps & How to Handle Them

| Trap | How to Handle |
|------|--------------|
| "Why Haskell? That's unusual." | "It's JusPay's standard stack. The type safety is critical for financial transactions — the compiler catches entire classes of bugs that would be runtime errors in dynamic languages. The tradeoff is hiring, which we mitigate with code generation (60% of code is auto-generated from YAML)." |
| "Why not microservices?" | "For our team size, a modular monolith with clear layer boundaries is more productive. Microservices add network complexity, distributed tracing overhead, and deployment coordination. Our feature flags give us the isolation we need without the operational cost." |
| "What would you do differently?" | Be honest about tech debt (incomplete patterns, SSL verification, no DB transactions), then describe your improvement plan. Show you're self-aware, not defensive. |
| "This sounds over-engineered." | "The code generation pipeline IS unusual, but it eliminates an entire class of errors — type mismatches between API specs, DB schemas, and Haskell code. With 75+ tables and 200+ endpoints, manual sync would be a constant source of bugs." |
