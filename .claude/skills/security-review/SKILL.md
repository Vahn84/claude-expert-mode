---
name: security-review
description: Use when reviewing code, a diff, or a PR for security — or when asked to audit/analyze a handler, endpoint, or module. A systematic pass so recall doesn't depend on which vulnerabilities happen to catch the eye. Report everything, including low-severity and uncertain findings; filter downstream, not at the finding stage. The review-time counterpart to `secure-by-default` + `endpoint-auth`.
---

# Systematic Security Review Pass

Vulnerability recall here is already strong. This skill exists so the pass is *consistent and complete* rather than dependent on what jumps out — and so nothing is silently dropped for being "probably fine" or "low severity". Report every finding with a severity and a concrete fix; let a downstream decision filter, never self-censor at the finding stage (self-filtering is how measured recall silently drops).

## The pass — walk every input to every sink

For each handler/function, trace **every source of untrusted input** (path params, query string, body, headers, cookies, uploaded files, external API responses, DB values written by other tenants) to **every dangerous sink** (SQL, shell, filesystem, template/HTML, HTTP client, deserializer, redirect). A finding is any untrusted source reaching a sink without the real boundary from `secure-by-default` / `endpoint-auth`.

Run these seven, explicitly — tick each or note "N/A because":

1. **Identity** — is the caller authenticated with a *verified* (not decoded) token, unexpired, right issuer/audience, in a header not the URL? And on state-changing requests, is there CSRF protection?
2. **Authorization (object-level)** — does this user own/may-access *this specific resource id*, on reads, writes, lists, and every download/export? IDOR is the most-missed critical. Watch mass-assignment too — can the body set `role`/`account_id`?
3. **Injection** — SQL/NoSQL, command, LDAP, template. Parameterized? Shell-free? Any string-built query is a finding.
4. **Untrusted content & files** — path traversal on read *and* write; user-influenced filenames; stored XSS / SVG / polyglot; is uploaded content re-encoded and served from a non-executable location?
5. **Secrets & data exposure** — secrets in code/logs/errors/URLs; verbose errors leaking internals; PII in logs; passwords stored with a fast hash or plaintext.
6. **Abuse & isolation** — size/rate/concurrency/pagination caps (unbounded work per request is a DoS finding); tenant isolation in key schemes, caches, and queues; auth-endpoint lockout + non-enumerating errors.
7. **Config, deps & error handling** — known-vuln dependencies, TLS on internal connections, services network-exposed that shouldn't be, unhandled promise rejections (hang/DoS), swallowed errors hiding auth failures.

## Output shape

Rank by severity (Critical → Low). For each: one-line mechanism, concrete fix, and — if unsure — say so and flag it anyway ("not sure the driver supports `$1` placeholders — confirm"). A "block this PR / rewrite required" verdict is valid and sometimes correct; say it when items are independently exploitable.

## Tooling note

Automated SAST (semgrep, gitleaks for secrets, trivy for deps) catches a portion — run it as a floor, not a ceiling. The authz/IDOR and business-logic findings above are exactly what scanners miss; those need this manual pass.

## Why this exists

Handover finding: given a PR, recall was full-marks (12 findings including the "not sure" flags). This codifies that pass so it's systematic every time and independent of what catches the eye — the analysis counterpart to the `secure-by-default` + `endpoint-auth` implementation gates. Consolidated from ten items to seven (the review dimensions grouped, none dropped) to stay under the skim limit.
