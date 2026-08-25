BUG-004 — User can access the account after logging out

**Title/Summary:** [ACCOUNT] User can access the account after logging out

**Short Description:** Logging out should invalidate the session on the server, not just clear it on the screen. Testing shows a token captured before logout can still be used to call protected API endpoints (like booking history) after the user has logged out. The server isn't actually revoking the token — only the screen looks "logged out."

**Preconditions:**
- Staging environment.
- Customer test account, logged in through the web app.
- Able to capture the auth token via Chrome DevTools (Application tab → Local Storage/Cookies, or Network tab on an authenticated request).
- Postman (or curl) available to replay the request outside the browser session.

**Steps to Reproduce:**
1. Log in as cust-session-qa@example-mail.com on staging web.
2. In DevTools → Network tab, load "My Bookings" and capture the Authorization: Bearer token from that request.
3. In Postman, use that token to call the same endpoint (GET /api/v1/bookings/me) and confirm it returns 200 OK with booking data — this just confirms the token is valid before logout.
4. Back in the browser, click "Log Out."
5. Confirm the UI goes back to the login page.
6. Immediately re-send the same Postman request from step 3, with the same token, unchanged.

**Actual Result:** The replayed request in step 6 still returns 200 OK with the customer's booking data. The token stays valid and usable for at least several minutes after logout (tested up to 15 minutes post-logout, still valid at that point; didn't test past that).

**Expected Result:** Logging out should invalidate the token/session on the server right away. A request using a post-logout token should return 401 Unauthorized.

**Test Data:**
- Account: cust-session-qa@example-mail.com
- Captured token: a bearer token (value not included here — see Evidence README for why raw tokens aren't included in this repo, even expired ones)
- Endpoint tested: GET /api/v1/bookings/me (also reproduced against GET /api/v1/profile/me)

**Environment/Configuration:**
- Environment: Staging
- Browser: Chrome 126
- Client: Postman for replay verification (this rules out the issue being a browser cache artifact)

**Evidence/Attachments:**
- evidence/bug-004-session-token/postman-pre-logout-200.png — the pre-logout request succeeding
- evidence/bug-004-session-token/postman-post-logout-still-200.png — the same request still succeeding after logout (token value redacted/blurred)
- evidence/bug-004-session-token/logout-network-call.png — DevTools capture of the logout API call, a 200 OK with nothing in the response indicating whether server-side invalidation happened

**Severity:** Critical — this is a session-security issue. A captured token (from a compromised device, shared computer, or browser extension) stays usable well beyond when the user thinks they've logged out.

**Priority:** P1 — not blocking this release's feature scope, but should be fixed urgently given the security implications.

**Repro Rate:** 5/5 (100%) — reproduced consistently, and the token was still valid when re-tested up to 15 minutes after logout, which was the longest window I tested.

**Workaround:** None on the end-user side. As an interim step, I suggested shortening the access token's expiry so a leaked/replayed token has a smaller exposure window while the real fix is worked on.

---

**Investigation Notes:**
- Confirmed in DevTools → Application tab that logout does correctly clear local storage/cookies on the client. So this isn't "the logout button doesn't work" — it's specifically that the server doesn't revoke the token, so anything holding a separate copy of it (like my Postman capture) is unaffected.
- The logout response itself (200 OK) gives no indication whether the server actually revoked anything — worth noting as a separate, smaller issue: there's no way to verify revocation from the API response alone.
- Based on this behavior, my guess is the app is using tokens with no server-side revocation check — but I haven't verified this in the code, so flagging it as a hypothesis for backend to confirm. Two directions that would fix it: a server-side check on each request, or short-lived tokens with a separate refresh token that can be revoked.
- Flagged this to the team as worth a permanent regression test case (see TC-AUTH-021), and as a candidate for security review given the potential impact if this reached production.
