# BUG-004 — Session/auth token remains valid on the backend after user logs out

**Title/Summary:**
[Authentication/Security] API access token captured before logout continues to authenticate successfully against protected endpoints after the user has logged out, instead of being invalidated server-side.

**Short Description:**
Logging out is expected to invalidate the session on the server, not just clear local client storage. Testing shows that a bearer token captured before logout can still be replayed against protected API endpoints (e.g., fetching booking history) *after* logout, meaning the server is not actually revoking the token — only the client-side UI is "logged out."

**Preconditions:**
- Staging environment.
- Customer test account, logged in via web.
- Ability to inspect and capture the auth/bearer token via Chrome DevTools (Application tab → Local Storage/Cookies, or Network tab on an authenticated request).
- Postman (or curl) available to replay captured requests independently of the browser session.

**Steps to Reproduce:**
1. Log in as `cust-session-qa@example-mail.com` on staging web.
2. Open DevTools → Network tab, perform any authenticated action (e.g., load "My Bookings"), and capture the `Authorization: Bearer <token>` header from that request.
3. In Postman, save this token and issue a `GET` request to the same protected endpoint (e.g., `GET /api/v1/bookings/me`) using the captured token — confirm it returns `200 OK` with booking data (sanity check that the token is valid pre-logout).
4. Back in the browser, click "Log Out."
5. Confirm the UI redirects to the login page as expected.
6. Immediately re-send the exact same Postman request from step 3, using the same captured token, without modification.

**Actual Result:**
The replayed request in step 6 still returns `200 OK` with the customer's booking data — the token remains valid and usable for at least several minutes after logout (tested up to 15 minutes post-logout, still valid at that point; did not test longer windows).

**Expected Result:**
Per standard session-security expectations, logging out should invalidate the token/session server-side immediately. A replayed request using a post-logout token should return `401 Unauthorized`.

**Test Data:**
- Account: `cust-session-qa@example-mail.com`
- Captured token: a JWT-style bearer token (value not reproduced here — see [Evidence README](../evidence/README.md) for why raw tokens are never included in this repo, even sanitized/expired ones)
- Endpoint tested: `GET /api/v1/bookings/me` (also reproduced against `GET /api/v1/profile/me`)

**Environment/Configuration:**
- Environment: Staging
- Build: `auth-service v2.0.4` (sample version label)
- Browser: Chrome 126
- Client: Postman v10 for replay verification (isolates this as a backend issue, not a browser cache artifact)

**Evidence/Attachments:**
- `evidence/bug-004-session-token/postman-pre-logout-200.png` — synthetic recreation of the pre-logout request succeeding
- `evidence/bug-004-session-token/postman-post-logout-still-200.png` — synthetic recreation of the same request still succeeding after logout (token value redacted/blurred)
- `evidence/bug-004-session-token/logout-network-call.png` — DevTools capture of the logout API call itself, showing a `200 OK` client-side response with no indication of what server-side invalidation (if any) occurred

**Severity:** Critical — this is a security/session-integrity issue. A stolen or captured token (e.g., via a compromised device, shared computer, or browser extension) remains usable well beyond when the user believes they've logged out.

**Priority:** P1 — not blocking this specific release's feature scope, but should be fixed urgently in the next sprint given the security implications; escalated to the security-conscious lead engineer for prioritization input alongside PM.

**Repro Rate:** 5/5 (100%) — consistently reproducible, and the token was still valid when re-tested up to 15 minutes after logout (the maximum window tested).

**Workaround:** None from the end-user side. Recommended interim mitigation: reduce the access token's natural expiry (TTL) so any leaked/replayed token has a smaller exposure window while the proper server-side revocation fix is developed.

**Investigation Notes:**
- Confirmed via DevTools → Application tab that the client-side logout correctly clears local storage/cookies — so the bug is not "logout button doesn't work," it's specifically that the *server* doesn't revoke the token, meaning anything holding a copy of the old token independently of the browser (like my Postman capture) is unaffected.
- Inspected the logout network request/response in DevTools: the logout call returns `200 OK` but the response body gives no indication whether server-side token revocation actually happened — raised this as a secondary observation, since even the API contract doesn't make revocation observable/verifiable.
- Hypothesis handed to backend dev: this looks like the API is using stateless JWTs with no server-side revocation list/blocklist and no short-lived token + refresh-token rotation pattern — the "logout" only ever was a client-side action. Recommended either (a) a server-side token blocklist checked on each request, or (b) moving to short-lived access tokens (e.g., 5–15 min) with revocable refresh tokens, so logout can meaningfully revoke the refresh token even if a short-lived access token has to naturally expire.
- Recommended this become a permanent regression test case (see [TC-AUTH-021](../test-cases/01-authentication-test-cases.md)) since session security is easy to silently break during unrelated refactors.
- Flagged to PM as a candidate for security review/pen-test scope given the potential real-world impact if this reached production undetected.
