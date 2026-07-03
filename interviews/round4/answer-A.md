# Plan of Record: Externalizing the Order Management API

## The one decision everything else hangs on

**We do not expose the internal API. We build a dedicated external API tier in front of it.**

Our current API was written for a trusted caller (our own frontend). That means it almost certainly has: authorization that assumes "the UI only asks for its own data" (IDOR waiting to happen), verbose errors that leak internals, unbounded queries, response shapes coupled to our DB schema, and no versioning contract. If we point 30 partners at it directly, every internal refactor becomes a partner-breaking incident and every implicit trust assumption becomes a security hole.

So the deliverable is an **external API tier** — an anti-corruption layer with its own explicitly designed contract, its own DTOs, and its own authz enforcement — that calls internal services. Internal API keeps moving fast; external contract stays stable. This is the load-bearing decision and I'll defend it against pressure to "just add auth to what we have," because that path doesn't survive contact with quarterly-release retailers.

---

## 1. Architecture

```
Partner → API Gateway (edge) → Partner API service (facade) → internal services/DB
             ↑ TLS, authn, rate-limit,        ↑ contract, DTO mapping,
               WAF, request logging             authz scoping, validation
```

- **Edge gateway** (Kong / AWS API Gateway / Envoy — pick per our infra): TLS termination, WAF, authn token validation, global rate limiting, request/response logging, IP allow-listing where partners want it.
- **Partner API service** (new service, owned by us): the public contract. Maps public DTOs ↔ internal models, enforces per-partner authorization on every object, validates all input, produces the stable error model. This is where the real work is.
- Facade calls internal services over our existing internal network. No partner traffic ever reaches an internal service directly.

---

## 2. Contract & versioning

- **OpenAPI 3.1 is the source of truth**, checked into the repo, CI-linted (Spectral). Docs, SDKs, and the mock server all generate from it. Spec review is a required PR gate.
- **URL major versioning: `/v1/`.** Simple, cache-friendly, unambiguous for partners.
- **Additive-only within a major version.** New optional fields and endpoints are fine; removing/renaming fields or tightening validation is a breaking change and requires a new major.
- **Contract test suite in CI** that fails the build on any breaking change to `/v1` (schema diff gate). This is the mechanism that makes "we won't break v1" a fact rather than a promise.
- We deliberately **narrow the surface**: expose only the resources partners need for order management (orders, order items, status, shipments, catalog lookups as required). Every exposed field is a field we're committing to support — default to not exposing.

---

## 3. AuthN / AuthZ / tenant isolation

**AuthN — OAuth2 client-credentials grant (M2M).**
- Each partner gets a `client_id` + `client_secret`, exchanges it for a short-lived (≤1h) JWT access token.
- **Secret rotation with overlap**: two active secrets per client so partners rotate without downtime — essential for the quarterly-release retailers.
- Offer **mTLS as an option** for large retailers whose security teams require it.
- API keys are explicitly rejected as the primary mechanism (weaker, hard to scope/rotate).

**AuthZ + tenant isolation — the highest-risk area.**
- Every access token carries the partner's `tenant_id` and a set of **scopes** (`orders:read`, `orders:write`, etc.). No token is issued with more scope than the deal requires.
- **Every object access is authorized against the token's tenant** in the facade. A partner requesting `GET /v1/orders/{id}` for an order they don't own gets a `404` (not `403` — don't confirm existence).
- **Defense in depth:** the facade scopes the query by tenant, *and* re-checks ownership on the returned object. We assume the internal service will someday return cross-tenant data by mistake and make sure it can't leak.
- I will run a dedicated **IDOR/BOLA audit** on every endpoint before go-live. This is the failure mode most likely to cause a breach and a lost contract.

---

## 4. Traffic management & API conventions

- **Rate limits & quotas per partner**, tiered (default tier + negotiated tiers). Enforced at the gateway. `429` with `Retry-After` and `X-RateLimit-*` headers. Limits are published in the docs.
- **Idempotency**: `Idempotency-Key` header required on all mutating requests (order creation especially). We store key→response for 24h so partner retries don't double-create orders. Non-negotiable for order APIs.
- **Cursor-based pagination** with an enforced max page size. No unbounded list endpoints.
- **Error model**: RFC 9457 (`application/problem+json`) with a **stable, documented error-code enum**. Never leak stack traces, SQL, or internal service names. Validation errors return field-level detail.
- **Consistency**: ISO-8601 UTC timestamps, documented enums, explicit nullability, correlation ID (`X-Request-Id`) echoed on every response and present in our logs.

---

## 5. Security hardening & review

- WAF at the edge; strict input validation (allow-list, size caps, content-type enforcement) in the facade.
- Secrets in our vault, never in code/config; partner credentials hashed at rest.
- Security headers, TLS 1.2+ only, HSTS.
- **Threat model workshop** in week 1 and a **third-party pentest** scheduled to complete before the first partner goes live — book the vendor now, it's the long-lead item.
- Abuse controls: anomaly alerts on per-partner error spikes and volume spikes; ability to kill a partner's access instantly (credential revocation switch).

---

## 6. Environments

- **Dedicated sandbox environment**, isolated from prod, with seeded/synthetic data. Partners integrate and test here first. Startups will live in it day one; retailers need it for their long QA cycles.
- Sandbox mirrors the prod contract exactly and is part of the release pipeline (never drifts).
- Prod access is granted only after a partner passes basic sandbox validation.

---

## 7. Observability, audit & support tooling

- **Per-partner dashboards**: request volume, latency (p50/p95/p99), error rate, rate-limit hits, top endpoints. Alerting per partner.
- **Immutable audit log** of every partner mutation (who, what, when, correlation ID) — needed for disputes and compliance.
- Correlation IDs threaded from edge through facade to internal calls so support can trace a partner's exact request.
- **Internal support console** to look up a partner's recent requests/errors — the support team cannot depend on us grepping logs at 2am.

---

## 8. Documentation & developer experience

DevX quality directly determines integration speed and support load.
- **Auto-generated reference docs** from the OpenAPI spec (Redocly/Stoplight), hosted on a partner developer portal.
- **Quickstart guide**, auth walkthrough, and a **Postman collection**.
- Published: rate limits, error codes, pagination model, idempotency behavior, **changelog**, and the **deprecation/versioning policy**.
- Generate at least one **client SDK** (e.g., TypeScript) from the spec to accelerate the startup partners.
- Self-service or lightly-gated **credential provisioning** flow for sandbox.

---

## 9. Lifecycle & governance

- **Published versioning & deprecation policy**, calibrated to our slowest partners:
  - Additive changes anytime, announced in the changelog.
  - **Minimum 12 months support** per major version after a successor ships; **≥6 months notice** before any endpoint sunset, communicated via changelog, email, and the `Sunset`/`Deprecation` HTTP headers.
- **Breaking-change governance**: any breaking change requires a new major version and sign-off — no exceptions, enforced by the CI schema-diff gate.
- **SLA** (uptime target, support response times, incident comms) — I'll draft the technical terms; **flag to legal/CTO** that partner-facing SLA, API Terms of Service, and data-processing terms need legal ownership before signature. I own the tech, not the contract language, but these must land in parallel.

---

## 10. Explicitly out of scope for v1 (scope discipline)

To hit 6 weeks safely, these are deferred and communicated as roadmap:
- **Webhooks / async event notifications** (partners will want order-status pushes) → Phase 2, with HMAC-signed payloads and retry/DLQ. High demand, but not on the critical path for first go-live.
- GraphQL, bulk/batch endpoints, partner self-serve analytics.
- Additional resources beyond core order management.

Naming these now prevents scope creep from sinking the date.

---

## 11. Six-week execution timeline

| Week | Milestone |
|------|-----------|
| **1** | Threat model + IDOR audit of internal endpoints. Freeze v1 contract (OpenAPI). Book pentest. Stand up gateway + skeleton facade + auth (client-credentials). |
| **2** | Implement core read endpoints with tenant-scoped authz + DTO mapping. Error model, correlation IDs, structured logging. |
| **3** | Mutating endpoints (order create/update) with idempotency + validation. Rate limiting + quotas at edge. |
| **4** | Sandbox environment live with seeded data. Docs portal + Postman + TS SDK generated. Contract-test CI gate + schema-diff gate in place. |
| **5** | Per-partner observability/dashboards/alerts. Pentest executes; remediate findings. Onboard first partner into **sandbox**. Load test to validate rate limits. |
| **6** | Close pentest findings, security sign-off. First partner sandbox validation passes → grant prod credentials → **go-live** with heightened monitoring. |

Critical path & risks: the **IDOR audit** and the **pentest** are the two things that can slip the date if started late — both begin week 1. If the internal API's authorization turns out to be worse than assumed, the facade's defense-in-depth checks are our safety net, but I'll escalate early if remediation threatens the timeline.

---

## 12. What "own it end to end" means operationally

Beyond ship: I own the **on-call rotation** for the partner API, the **partner-facing incident comms**, the **breaking-change approval gate**, the deprecation calendar, and the DevX quality bar. The team executes against this doc; I hold the line on "never break v1 in place" and "no partner request reaches an internal service unmediated" — those two invariants are what keep 30 partners on different release cadences from becoming 30 recurring fire drills.

---

Two things I need from the CTO to lock this: (1) legal engaged this week on ToS/SLA/data terms, and (2) confirmation of the pentest budget so I can book the vendor today. Everything else I can drive.

Want me to expand any section into an implementation-level design doc — the facade/authz layer and the sandbox are the two I'd write up first.