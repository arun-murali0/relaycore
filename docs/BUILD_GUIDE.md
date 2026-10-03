# RelayCore - Build Guide

## 1. Architecture

```
                 ┌────────────────────────── API process (src/server.ts) ──────────────────────────┐
Internal         │  authenticateTenant → rateLimit(Redis) → route → service                         │
Service ──POST──▶│                                          │                                       │
 /api/v1/events  └──────────────────────────────────────────┼───────────────────────────────────────┘
                                                            │ 1. save Event        (MongoDB)
                                                            │ 2. match Endpoints   (MongoDB)
                                                            │ 3. save Delivery=PENDING per endpoint
                                                            │ 4. queue.add(deliveryId)  (BullMQ/Redis)
                                                            ▼
                                                    ┌──────────────┐
                                                    │ Redis queue  │
                                                    └──────┬───────┘
                 ┌───────────────── Worker process (src/worker.ts) ─────────────────────────┐
                 │  load Delivery+Endpoint+Event                                            │
                 │  circuit OPEN? ──yes──▶ delay job, don't call                            │
                 │  sign payload (HMAC) → POST to customer URL (timeout)                    │
                 │     ├─ 2xx            → DELIVERED, reset circuit                         │
                 │     ├─ retryable fail → RETRYING, backoff, failures++ (maybe open circuit)│
                 │     ├─ 4xx            → FAILED (no retry)                                │
                 │     └─ attempts used up → FAILED = dead letter                           │
                 └──────────────────────────────────────────────────────────────────────────┘
```

Two processes, same codebase: **API** (fast, never calls customers) and **Worker** (slow, does the HTTP calls).
That split is the core design idea. You can scale workers independently.

Rule that applies to every phase: **every DB query includes `tenantId`**.

## 2. Folder map

| Path | Role |
|---|---|
| `src/config` | env parsing/validation (zod) |
| `src/db` | Mongo + Redis connections |
| `src/models` | Mongoose schemas: Tenant, Endpoint, Event, Delivery |
| `src/middleware` | auth, rate limit, error handler |
| `src/routes` | thin HTTP layer: validate input, call service, return JSON |
| `src/services` | business logic (event fan-out, delivery attempt) |
| `src/queue` | BullMQ queue definition |
| `src/workers` | BullMQ consumer |
| `src/lib` | pure logic: signature, retry rules, circuit breaker (unit-test these first) |
| `tests` | vitest |

## 3. Phases

Do them in order. Each ends with a "done when" check. Commit after each.

### Phase 0 - Setup (30 min) - DONE in scaffold
`cp .env.example .env && npm i && npm run infra:up && npm run dev:api`
**Done when:** `GET /health` returns `{mongo:true, redis:true}` and `npm test` passes.
Learn: why API and worker are separate processes.

### Phase 1 - Tenants + API-key auth
- Add a script `scripts/createTenant.ts`: generate random key (`crypto.randomBytes(24).toString("hex")`), print it once, store only its SHA-256 hash.
- Implement `middleware/auth.ts`: read Bearer token, hash, find Tenant, set `req.tenant`. 401 otherwise.
- Add a type augmentation so `req.tenant` is typed.
**Done when:** `/api/v1/*` returns 401 without a key, 200 with one.
Learn: why you hash API keys; tenant identity flowing through a request.

### Phase 2 - Endpoint CRUD
- `POST/GET/PATCH/DELETE /api/v1/endpoints`, always filtered by `tenantId`.
- On create: validate URL with zod (https only), generate `secret` (`whsec_` + random), return it once.
- Block private/localhost URLs (basic SSRF guard).
**Done when:** Tenant A cannot read, update or delete Tenant B's endpoint (write a test for this).
Learn: tenant isolation in queries, SSRF.

### Phase 3 - Event ingestion + fan-out
- `POST /api/v1/events` body `{type, payload}`, optional `Idempotency-Key` header.
- `eventService.create`: save Event → find active endpoints whose `eventTypes` match (`*` or exact) → create a Delivery(PENDING) per endpoint → `deliveryQueue.add("deliver", {deliveryId}, {jobId: deliveryId})`.
- Return `202 Accepted` with the event id.
**Done when:** posting an event creates N Delivery docs and N jobs in Redis. Posting twice with the same idempotency key creates one event.
Learn: async acceptance (202), fan-out, idempotency.

### Phase 4 - Worker: deliver + sign + record
- `deliveryWorker`: BullMQ `Worker` calls `deliveryService.attempt(deliveryId)`.
- attempt(): set PROCESSING, build body `JSON.stringify({id, type, createdAt, data})`, header `X-Relay-Signature: sign(secret, body)`, `fetch` with `AbortSignal.timeout(WEBHOOK_TIMEOUT_MS)`.
- Record `attempts++`, `lastResponse {statusCode, durationMs, error}`. 2xx → DELIVERED.
- Enable the import in `src/worker.ts`.
Test it with a tiny local receiver (or webhook.site) that calls `verify()` from `lib/signature.ts`.
**Done when:** an event reaches a receiver and the signature verifies there.
Learn: HMAC signing, timeouts, why 2xx = success.

### Phase 5 - Retries + failure classification
- In attempt(): if failure and `isRetryable(status)` and `attempts < maxAttempts` → status RETRYING, set `nextRetryAt`, re-enqueue with `delay: backoffMs(attempts)`.
- Non-retryable (400/401/403/404) or attempts exhausted → FAILED.
- Add jitter (±20%) to the delay so retries don't all fire together.
**Done when:** a receiver that returns 503 three times then 200 results in DELIVERED with attempts=4; a receiver returning 404 fails immediately.
Learn: exponential backoff, jitter, retryable vs permanent errors.

### Phase 6 - Rate limiting
- `middleware/rateLimit.ts`: key `rl:<tenantId>:<epochMinute>`, `INCR`, `EXPIRE 60` on first hit; over limit → 429 with `Retry-After`.
- Limit comes from `tenant.rateLimitPerMin`.
**Done when:** the 501st request in a minute gets 429 and a different tenant is unaffected.
Learn: fixed window vs sliding window, Redis atomic counters.

### Phase 7 - Circuit breaker
- State lives on `endpoint.circuit`. Implement `lib/circuitBreaker.ts` as pure functions: `onSuccess`, `onFailure`, `canAttempt`.
- CLOSED → OPEN after `FAILURE_THRESHOLD` consecutive failures. OPEN → HALF_OPEN after `COOLDOWN_MS` (allow one probe). Probe success → CLOSED, failure → OPEN.
- In worker: if not `canAttempt`, re-queue the job with a delay and do **not** count an attempt.
**Done when:** a dead endpoint stops receiving calls after 5 failures and recovers automatically when it comes back.
Learn: state machines, protecting your own workers.

### Phase 8 - Delivery visibility + manual retry (dead letters)
- `GET /api/v1/deliveries?status=FAILED&eventId=...` (paginated, tenant-scoped).
- `GET /api/v1/deliveries/:id`.
- `POST /api/v1/deliveries/:id/retry`: only for FAILED; reset attempts, status PENDING, re-enqueue.
**Done when:** you can find a failed delivery, fix the endpoint, retry it and see it DELIVERED.
Learn: designing for debuggability.

### Phase 9 - Hardening
- Graceful shutdown (SIGTERM: stop accepting, `worker.close()`, close DB).
- Integration tests (supertest + a fake receiver).
- Structured logs with `tenantId`, `deliveryId`.
- Metrics: queue depth, delivery latency, failure rate.
- Secret rotation for endpoints, replay endpoint for an event, Dockerfile for api + worker.

## 4. Suggested weekly pace
Weekend 1: phases 0-3 · Weekend 2: phases 4-5 · Weekend 3: phases 6-7 · Weekend 4: phases 8-9.

## 5. Gotchas to remember
- BullMQ needs `maxRetriesPerRequest: null` on the Redis connection (already set).
- Use `jobId: deliveryId` so enqueueing twice can't create duplicate jobs.
- Webhooks are **at-least-once**: receivers must be idempotent. Include the event `id` in the body.
- Never log secrets or full payloads.
- Sign the exact raw string you send, not a re-serialized object.
