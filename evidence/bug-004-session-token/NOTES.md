# Evidence Notes — BUG-004 (Session token valid after logout)

Referenced by [`bug-reports/BUG-004-session-token-not-invalidated-on-logout.md`](../../bug-reports/BUG-004-session-token-not-invalidated-on-logout.md).

- **`postman-pre-logout-200.png`** — synthetic recreation of the Postman request to `GET /bookings/me` succeeding with `200 OK` before logout, using a captured bearer token, to establish the baseline.
- **`postman-post-logout-still-200.png`** — the same request replayed after logout, still returning `200 OK` — the core evidence of the bug. Token value fully redacted/blurred in the screenshot, consistent with the sanitization rules in the [Evidence README](../README.md) (raw tokens, even expired or test ones, are never included in this repo).
- **`logout-network-call.png`** — DevTools capture of the logout API call itself, included to show the client-side call succeeded normally, ruling out "the logout button is broken" as an explanation and isolating the issue to server-side token handling.
