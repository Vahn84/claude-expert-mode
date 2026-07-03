---
name: secure-by-default
description: Use when WRITING or MODIFYING code where untrusted input reaches a dangerous sink — file uploads, SQL/queries, shell/exec, template/HTML rendering, file paths, deserialization, outbound HTTP/redirects. Fires at implementation time, not review time. The real security boundary is the default, never a footnote. For auth/session/CSRF/password surfaces, see `endpoint-auth`; for the review-time counterpart, see `security-review`.
---

# The Footnote Is the Control

Measured failure mode: when reviewing a PR, vulnerability recall is excellent. When *writing* code unprompted, the instinct is to reach for the plausible-looking control and relegate the actual boundary to a "before production, you'd want to…" note. Example caught in the handover: an avatar upload validated by `file.mimetype` (client-supplied, trivially spoofed) with "or re-encode through sharp" mentioned as an afterthought — so a polyglot with `Content-Type: image/png` ships. **The footnote was the real control. Do the footnote by default.**

Rule: for each surface below, the *right column is what you write the first time*, not what you suggest folding in later. If you catch yourself writing "for real safety you'd want X" — X is the implementation, not the addendum.

| Surface (untrusted input → sink) | The plausible control (insufficient) | The actual boundary (write this by default) |
|---|---|---|
| **File upload** | Extension / `mimetype` allowlist | Re-encode/parse through a real library (`sharp` for images), server-derived filename, size + count caps, store outside webroot or object storage |
| **SQL / queries** | Escaping, string checks | Parameterized queries / bound params — always, no exceptions for "internal" values |
| **Shell / exec** | Sanitizing the string | `execFile`/`spawn` with an arg array, no shell; allowlist the command and flags |
| **HTML / templates** | Stripping `<script>` | Context-aware auto-escaping output encoding; treat all interpolated data as hostile; CSP as defense-in-depth |
| **File paths** | Prefix-concatenating user input | `path.basename` + a fixed root option that rejects `..`; never interpolate user input into a path |
| **Deserialization** | Trusting the payload shape | Schema-validate before use; never deserialize to arbitrary types; reject unknown fields |
| **Redirects / SSRF** | Checking the string starts with your domain | Allowlist of exact hosts; resolve and re-check; block internal IP ranges for outbound fetch |

## The gate

Before finishing code on any surface above, write one line naming the boundary you used from the right column. If the line reads like a deferral ("would want to", "in prod you'd add", "for real safety"), stop — move it into the code. And make assumptions **visible, not silent**: if the secure version depends on `requireAuth` existing, say "this is unsafe to ship until `requireAuth` is wired" rather than assuming it silently — then run `endpoint-auth` for that half.

## Why this exists

Handover finding: the knowledge is fully present — cold Opus *named* the correct fix every time. The gap is that at implementation time the correct fix arrives as a suggestion, not as the code. This skill collapses that gap: the boundary is the default. Split from the identity/session surfaces (`endpoint-auth`) so each gate stays scannable.
