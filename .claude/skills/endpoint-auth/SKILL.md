---
name: endpoint-auth
description: Use when WRITING or MODIFYING an authenticated HTTP endpoint, a login/session/password flow, or any handler that acts on a user-scoped resource. The identity, session, and request-integrity half of secure implementation — the surfaces that a naive handler gets wrong by default. Sibling to `secure-by-default` (injection/execution surfaces).
---

# Endpoint Auth: Identity Is Not a Footnote Either

The same footnote-instead-of-code failure that `secure-by-default` fixes for injection surfaces applies to identity and session surfaces — and these are the ones a "quick" handler silently assumes away ("`requireAuth` sets `req.user`", "the user can only see their own stuff"). Each row's right column is what you write the first time.

| Surface | The plausible control (insufficient) | The actual boundary (write this by default) |
|---|---|---|
| **Authentication** | `jwt.decode` / trusting the payload / token in the URL | `jwt.verify` with secret + expiry + issuer/audience; token in the `Authorization` header (query strings leak into logs/proxies/Referer) |
| **Authorization (object-level / IDOR)** | Authenticated == allowed | On every access — including reads, list, and the download/export endpoint everyone forgets — check that *this* user may touch *this* resource id. Derive ownership server-side (`req.user.id`), never from a client-supplied param |
| **CSRF** | "They're logged in, so it's them" | State-changing requests need an anti-CSRF token or `SameSite` cookies + origin check. A logged-in session is not intent |
| **Mass-assignment / over-posting** | Binding the request body straight to the model | Allowlist bindable fields explicitly. Never let the client set `role`, `is_admin`, `account_id`, `verified` by putting them in the JSON body |
| **Password & secret storage** | Fast hash (md5/sha), or "we'll encrypt it later", or a key in the code | `bcrypt`/`argon2`/`scrypt` with a per-user salt for passwords; secrets from env or a secret manager, never committed. Plaintext or a fast hash is a breach waiting to be dumped |
| **Auth-endpoint abuse** | Trusting the endpoint won't be hammered | Rate-limit / lockout on login, password-reset, and token endpoints; return generic messages ("invalid credentials") so the response can't enumerate valid users |

## The gate

Before finishing an authenticated endpoint, write one line per applicable surface naming the boundary — and state the auth dependency out loud: *"unsafe to ship until `requireAuth` verifies the token and sets `req.user.id`"* beats assuming it silently. If the endpoint takes untrusted input into a sink as well (it usually does), run `secure-by-default` alongside this.

## Why this exists

Review finding: the original `secure-by-default` table carried auth surfaces but a mechanical audit found CSRF, mass-assignment, and password-hashing **absent entirely** — three of the most common web-app vulnerabilities, with zero coverage. Splitting identity/session into its own gate both fixes that hole and keeps each security skill under the "≤7 items or it gets skimmed" limit that the whole handover is built on.
