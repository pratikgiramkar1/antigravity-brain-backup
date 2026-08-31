# Part 4 — Security, Observability, Testing & Critical Review

> Covers **Phase 8** (Security), **Phase 9** (Observability), **Phase 10** (Testing), **Phase 11** (Critical Review), **Phase 12** (Improvement Plan), and **Phase 13** ("If I Built It Today")

---

## 1. Security (Phase 8)

### 1.1 Authentication — Two Parallel Systems

> Source: [19-authentication.md](file:///Users/pratik.giramkar/Breeze/vayu/plan/19-authentication.md)

| System | Purpose | Token Type | Verification |
|--------|---------|-----------|-------------|
| **Customer Auth** | End-user checkout sessions | JWT (HMAC-SHA256) | Verify signature + expiry + extract claims |
| **S2S Auth** | Server-to-server API calls | JWT (Bearer) or Basic | DB lookup → check access list + route permissions |

**Customer JWT Claims:**
```json
{
  "iss": "www.breeze.in",
  "iat": 1692700000,
  "exp": 1695292000,
  "name": "Pratik",
  "emailAddress": "pratik@example.com",
  "countryCode": "+91",
  "phoneNumber": "9876543210",
  "customerId": "cust_abc123",
  "scope": "checkout"
}
```

**Auth Validation in Route Handlers:**
```haskell
-- Core.hs handler pattern
cartService reqBody sessionId shopUrl auth deviceId = do
  env ← overrideSessionEnv sessionId shopUrl deviceId
  withFlowHandler env $ do
    -- Some endpoints validate auth, some don't
    mToken ← getBearerTokenData auth  -- Verify JWT, extract Token
    Product.Cart.createCart reqBody mToken ...
```

**Three auth patterns in Core.hs:**

| Pattern | Example Endpoints | Auth Check |
|---------|-------------------|-----------|
| **Public** (no auth) | `POST /cart`, `GET /heartbeat` | None |
| **Customer only** | `POST /start-payment`, `PUT /offer` | `getBearerTokenData` → validate JWT |
| **S2S only** | `POST /global-config`, `DELETE /cpo` | `validateS2SAuth` → DB key lookup |
| **Dual** (either) | `GET /order/{id}`, `POST /session/start` | Try customer first, fallback to S2S |

### 1.2 Secrets Management

| Secret Type | Storage | Access |
|------------|---------|--------|
| **DB credentials** | Environment variables | `Config.hs` reads at startup |
| **Supermoney tokens** | Redis (encrypted + Base64) | Encrypt via `KeyStore`, TTL-based expiry |
| **Platform API keys** | `shop_integration` table (encrypted JSON) | Decrypted at runtime via `KeyStore` |
| **S2S API keys** | `key_store` table | `KeyStore.encrypt/decrypt` with key identifier |
| **JWT signing key** | `TOKEN_SIGN_KEY` env var | Read once at startup |
| **Notification credentials** | `message_credentials` table (encrypted) | Decrypted per-request |

**Encryption pattern** (KeyStore):
```haskell
-- Services/Internal/KeyStore/Main.hs
encrypt :: KeyIdentifierEnum → Text → Flow Text
decrypt :: KeyIdentifierEnum → Text → Flow Text
-- Uses identifier to select the encryption key from key_store table
```

### 1.3 Security Middleware

**Response middleware** ([Response.hs](file:///Users/pratik.giramkar/Breeze/vayu/src/Vayu/Middlewares/Response.hs)):
- `X-Frame-Options: DENY`
- `Content-Security-Policy: default-src 'self'`
- `X-Content-Type-Options: nosniff`
- `Strict-Transport-Security: max-age=31536000`

**Request middleware**:
- Raw query string and headers injected for downstream access
- Webhook empty body protection (prevents parse failures)

### 1.4 Log Masking

```haskell
-- App.hs:66-70
LogMaskingConfig
  { maskKeys = GenAccessor.maskConfig   -- Auto-generated blacklist
  , maskText = "XXXXXXXXX"
  , keyType  = BlackListKey             -- Mask these keys
  }
```

The `maskConfig` is auto-generated from YAML schemas — any field marked as sensitive is automatically masked in logs.

### 1.5 Security Weaknesses (Honest Assessment)

| Issue | Severity | Details |
|-------|----------|---------|
| **No request signing** | MEDIUM | SDK→backend calls rely on CORS + JWT only. No HMAC request signing. |
| **SSL verification disabled** | HIGH | `TLSSettingsSimple True False False` — first `True` disables certificate verification. Confirmed in [App.hs:118-119](file:///Users/pratik.giramkar/Breeze/vayu/src/Vayu/App.hs#L118-L119). |
| **JWT key rotation** | UNKNOWN | No evidence of key rotation mechanism for `TOKEN_SIGN_KEY`. |
| **Rate limiting scope** | MEDIUM | OTP rate limiting exists, but no general API rate limiting (DDoS protection likely at infra/CDN level). |
| **No CSRF protection** | LOW | SDK is embedded via JS (no cookies), so CSRF is less relevant. |

---

## 2. Observability (Phase 9)

### 2.1 Three-Level Structured Logging

> Source: [CLAUDE.md](file:///Users/pratik.giramkar/Breeze/vayu/CLAUDE.md), [plan/01-architecture.md](file:///Users/pratik.giramkar/Breeze/vayu/plan/01-architecture.md)

| Level | Function | Where Used | What It Logs |
|-------|----------|-----------|-------------|
| **Product API** | `withProductAPILogging` | Product layer entry points | Method, path, payload, response, latency |
| **Product Helper** | `withFunctionLogging` | Product helper functions | Function name, args, result |
| **Service** | `logFunctionCalled` + `logFunctionCallResult` | Services/Internal, Services/External | Function entry/exit with args |

**Logging pattern example:**
```haskell
-- Product layer
getWalletBalance partnerMerchantId = do
  startTime ← logProductAPIRequest tag GET "/supermoney/wallet/balance" payload
  result ← Service.getWalletBalance partnerMerchantId
  case result of
    Left err  → logAndThrowErr500 tag err
    Right resp → logProductAPIResponse tag startTime resp

-- Service layer
getWalletBalance pmId = do
  logFunctionCalled tag [("partnerMerchantId", pmId)]
  result ← callAPI ...
  logFunctionCallResult tag result
  return result
```

### 2.2 Log Format

```
-- Development (INTEG):
customFlowFormatter → human-readable, colored output

-- Production (SANDBOX/PROD):
defaultFlowFormatter → structured JSON for log aggregation
```

### 2.3 Tracing

| Header | Purpose |
|--------|---------|
| `X-Session-Id` | Session-level tracing (ties all requests in one checkout) |
| `x-request-id` | Request-level tracing (unique per HTTP request) |
| `X-Device-Id` | Device-level tracing |

These are stored in `Env` and threaded through every `Flow` action.

### 2.4 Metrics (Prometheus)

```haskell
-- App.hs:211-215
if isPrometheusEnabled
  then do
    register ghcMetrics          -- GHC runtime metrics
    prometheus def baseApp       -- WAI prometheus middleware
```

Enabled via `ENABLE_PROMETHEUS_METRICS=true`. Exposes standard WAI metrics (request count, latency histograms, response codes) + GHC runtime metrics (heap size, GC stats).

### 2.5 Kafka Logging

```haskell
-- Async event logging pipeline
bracket
  (KafkaProducer.initKafka env)        -- Initialize producer
  (\env' → closeKafkaRuntime ...)      -- Cleanup on shutdown
  (\env' → runSettings settings app)   -- Serve with Kafka in scope
```

Events are published to Kafka topics for downstream analytics and auditing.

### 2.6 Telemetry Batching

```haskell
-- App.hs:188-189
when isBatchingEnabled $
  forkIO $ forever $
    threadDelay (flushIntervalSeconds * 1000000) >> flushTelemetry env
```

Telemetry events are batched in memory and flushed periodically (configurable interval). On shutdown, remaining events are drained.

---

## 3. Testing (Phase 10)

### 3.1 Unit Tests (60 Haskell spec files)

Location: `test/` directory

| Category | Test Files | Examples |
|----------|-----------|---------|
| **Order** | 7 | `OrderStatusUpdateSpec`, `ShopifyOrderSpec`, `NeedToProcessOrderSpec` |
| **Cart** | 5 | `CartUtilsSpec`, `CartLockSpec`, `CartSignedCheckoutSpec` |
| **Offer** | 4 | `OfferProviderSpec`, `LineItemOffersSpec`, `NewCustomerOffersSpec` |
| **Payment** | 5 | `EulerUtilsSpec`, `ShopifyPaymentsRefundSpec`, `PaymentUtilsCurrencySpec` |
| **Shipping** | 3 | `ShippingSpec`, `ClickPostServiceabilitySpec`, `LeonardoEDDSpec` |
| **Abandonment** | 3 | `AbandonmentSpec`, `AbandonmentPermalinkSpec`, `CartRecreateAbandonmentSpec` |
| **Analytics** | 3 | `ExternalTrackerPurchaseEventSpec`, `ShopifyAnalyticsSpec`, `WebEngageSpec` |
| **Identity** | 2 | `PhoneNumberSpec`, `CustomerLocationSpec` |
| **Other** | 28 | Address, Campaign, Subscription, Wallet, Supermoney, etc. |

### 3.2 CATS Functional Testing

> Source: [README.md](file:///Users/pratik.giramkar/Breeze/vayu/README.md), `doc/cats/`

CATS (Contract API Testing Suite) — fuzzes the OpenAPI spec to test API contracts:

```bash
# Run CATS tests
pnpm run merge:openapi:functional
java -jar cats.jar --contract merged_api.yaml --server http://localhost:9100
```

Custom test cases in `doc/cats/functionalFuzzer.yaml` with request/response assertions.

### 3.3 Sanity Tests

Location: `tests/sanity/` — TypeScript tests for Shopify, WooCommerce, and GCP integrations.

### 3.4 Testing Gaps

| Gap | Risk | Evidence |
|-----|------|---------|
| **No integration test framework** | Medium | No Docker Compose or test containers for PostgreSQL/Redis |
| **No load testing** | Medium | No k6/Gatling/Locust scripts in repo |
| **No contract tests for external APIs** | High | External API changes detected only at runtime |
| **Limited ProcessTracker tests** | High | No tests for the distributed scheduling logic |
| **No Magento-specific tests** | Medium | `tests/sanity/` has Shopify and WooCommerce but not Magento |

---

## 4. Critical Review (Phase 11)

### 4.1 Architecture Strengths

1. **Code generation pipeline** — Eliminates manual type synchronization across 75+ tables. Schema changes propagate automatically.
2. **Type-safe API routes** — Servant catches routing errors at compile time. Adding a new endpoint is a YAML change + regeneration.
3. **ProcessTracker** — Battle-tested distributed task scheduler without external dependencies (Kafka/RabbitMQ).
4. **Platform abstraction** — Adding a new e-commerce platform requires implementing ~8 functions, not changing core logic.
5. **Three-tier config** — Feature flags at shop level without deployments.
6. **Crash recovery in master-worker** — Processing keys ensure no data loss on pod crashes.

### 4.2 Architecture Weaknesses

| Issue | Severity | Details |
|-------|----------|---------|
| **Monolithic Core.hs** | HIGH | 6647 lines, all route handlers in one file. Adding a route requires touching this file. |
| **Product/Offer.hs** | HIGH | ~38K tokens. Mixes Shopify-specific, Euler, and platform-generic logic. |
| **No DB transactions** | HIGH | Multi-table writes (cart + items + offers) are not atomic. |
| **Unbounded ProcessTracker** | MEDIUM | No cleanup of terminal tasks. Table grows forever. |
| **`-fno-warn-incomplete-patterns`** | HIGH | Used in 10+ files. Suppresses compiler warnings that catch missing enum cases → runtime crash risk. |
| **Architecture violations** | HIGH | 5+ External service files contain DB queries and business logic (see Part 1). |
| **SSL verification disabled** | HIGH | `TLSSettingsSimple True False False` in production HTTP manager. |
| **`fromJust` usage** | HIGH | Found in `Product/Offer.hs` — runtime crash on `Nothing`. |
| **No read replicas** | MEDIUM | Single PostgreSQL instance for all reads and writes. |
| **Producer polling gap** | LOW | 10s sleep between cycles = up to 10s delay for new tasks. |

### 4.3 Code Quality Signals

| Signal | Status |
|--------|--------|
| Consistent naming conventions | ✅ Yes — modules follow `Product/X/Main.hs`, `Services/Internal/X/Main.hs` pattern |
| Documentation in code | ⚠️ Mixed — Supermoney has excellent Haddock docs, older code has minimal comments |
| Generated code segregation | ✅ Yes — strict `src/Vayu/Generated/` boundary |
| Error handling consistency | ⚠️ Mixed — some use `Either`, some throw exceptions, some use `fromJust` |
| Logging consistency | ⚠️ Mixed — newer code uses `withProductAPILogging`, older code uses manual `logInfo` calls |

---

## 5. Improvement Roadmap (Phase 12)

### Priority 1: Safety & Correctness

| # | Improvement | Impact | Effort |
|---|-------------|--------|--------|
| 1 | **Remove all `-fno-warn-incomplete-patterns`** | Prevents runtime crashes when new enum values are added | Medium |
| 2 | **Replace `fromJust` with safe alternatives** | Prevents production crashes on unexpected `Nothing` | Low |
| 3 | **Enable SSL verification** | Prevents MITM attacks on external API calls | Low |
| 4 | **Wrap multi-table writes in DB transactions** | Prevents orphaned records on crashes | High |

### Priority 2: Scalability

| # | Improvement | Impact | Effort |
|---|-------------|--------|--------|
| 5 | **Add ProcessTracker table cleanup** | Prevents unbounded table growth | Low |
| 6 | **Read replicas for PostgreSQL** | Reduce read load on primary | High (infra) |
| 7 | **Event-driven PT trigger** | Replace 10s polling with pub/sub notification | Medium |
| 8 | **Split Core.hs into domain modules** | Improve maintainability and build parallelism | Medium |

### Priority 3: Architecture Debt

| # | Improvement | Impact | Effort |
|---|-------------|--------|--------|
| 9 | **Refactor External services** | Move DB queries out of Shopify/Magento/Euler external modules | High |
| 10 | **Split Product/Offer.hs** | Break 38K-token file into focused modules | Medium |
| 11 | **Add integration test framework** | Catch DB/Redis issues before production | High |
| 12 | **Implement contract testing** | Detect external API changes early | Medium |

---

## 6. "If I Built It Today" (Phase 13)

### What I'd Keep

1. **Code generation from OpenAPI specs** — This is the single best architectural decision. I'd keep it and extend it.
2. **Type-safe SQL (Beam)** — Catching SQL errors at compile time is invaluable for financial data.
3. **ProcessTracker concept** — Database-backed task scheduling with retry is pragmatic. I might use a more standard tool (Temporal, pg-boss) but the concept is right.
4. **Three-tier config** — Shop-level overrides without deployments is essential for multi-tenant systems.
5. **Platform abstraction via ADT dispatch** — Clean, compiler-checked exhaustiveness.

### What I'd Change

| Area | Current | Proposed |
|------|---------|----------|
| **DB Transactions** | Individual queries per table | `withTransaction` wrapping related writes |
| **Task Scheduling** | Custom ProcessTracker | Temporal.io for workflow orchestration (typed sagas, visibility UI, built-in retry) |
| **Service Mesh** | Direct HTTP calls | gRPC with protobuf for internal service calls, REST for external |
| **Observability** | Custom structured logging | OpenTelemetry (traces + metrics + logs unified) |
| **Testing** | Manual unit tests | Property-based testing (QuickCheck) for price calculations, contract tests for external APIs |
| **Deployment** | Monolith with feature flags | Domain-bounded services (Order, Payment, Notification as separate services) — BUT only after the team grows |
| **Message Queue** | Redis List (RPUSH/LPOP) | Kafka or SQS for durable queuing (Redis queues are not durable across restarts) |
| **Secret Management** | Env vars + DB encryption | HashiCorp Vault or AWS Secrets Manager |

### What I'd NOT Change for a Team of This Size

1. **Keep the monolith** for now — microservices add network complexity. A modular monolith with clear boundaries is better for small teams.
2. **Keep Haskell** — the type safety pays off enormously in financial systems. The hiring cost is real but manageable with the EulerHS framework lowering the barrier.
3. **Keep the code generation pipeline** — it's unusual but it works extremely well at this scale.
