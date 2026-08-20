# Test Cases — Authentication

Covers sign up, login, logout, password reset, and session handling for both caregiver and customer accounts. See [template](../templates/test-case-template.md) for column definitions.

## Sign Up

| ID | Title | Preconditions | Test Data | Steps | Expected Result | Priority | Type | Platform |
|---|---|---|---|---|---|---|---|---|
| TC-AUTH-001 | Successful sign up with valid data (customer) | No account exists for the email | email: jane.doe.qa@example-mail.com, password: `Str0ngP@ss!`, role: Customer | 1. Go to Sign Up 2. Fill name, email, password 3. Select "Customer" 4. Submit | Account created, verification email sent, user redirected to "verify your email" screen | P0 | Positive | Web |
| TC-AUTH-002 | Successful sign up with valid data (caregiver) | No account exists for the email | email: maria.qa.tester@example-mail.com, password: `Str0ngP@ss!`, role: Caregiver | Same as above, select "Caregiver" | Account created, redirected to caregiver onboarding (profile setup) instead of customer home | P0 | Positive | Web |
| TC-AUTH-003 | Sign up rejected for already-registered email | An account already exists for the email | Existing email: jane.doe.qa@example-mail.com | 1. Go to Sign Up 2. Enter the existing email 3. Submit | Inline error: "An account with this email already exists. Log in instead?" with link to Login | P1 | Negative | Web/Mobile |
| TC-AUTH-004 | Password strength validation | None | password: `1234` | Enter weak password, attempt submit | Inline error explaining requirement (min 8 chars, 1 number, 1 special char); submit blocked | P1 | Negative | Web/Mobile |
| TC-AUTH-005 | Email format validation | None | email: `not-an-email` | Enter invalid email format, attempt submit | Inline error "Enter a valid email address"; submit blocked | P2 | Negative | Web/Mobile |
| TC-AUTH-006 | Required field validation | None | Leave name blank | Submit form with name field empty | Inline error under name field; submit blocked; no partial account created | P2 | Negative | Web/Mobile |
| TC-AUTH-007 | Terms of Service checkbox required | None | Valid data, ToS checkbox unchecked | Fill all fields correctly, leave ToS unchecked, submit | Submit blocked with message to accept ToS | P2 | Negative | Web |

## Login

| ID | Title | Preconditions | Test Data | Steps | Expected Result | Priority | Type | Platform |
|---|---|---|---|---|---|---|---|---|
| TC-AUTH-010 | Successful login with valid credentials | Verified account exists | email: jane.doe.qa@example-mail.com / correct password | Enter valid credentials, submit | Redirected to home/dashboard for correct role; session cookie/token set | P0 | Positive | Web/Mobile |
| TC-AUTH-011 | Login rejected with wrong password | Verified account exists | Correct email, wrong password | Enter valid email + wrong password, submit | Generic error "Incorrect email or password" (does not reveal which field is wrong); no lockout on first attempt | P0 | Negative | Web/Mobile |
| TC-AUTH-012 | Account lockout after repeated failed attempts | Verified account exists | Correct email, wrong password ×5 | Attempt login with wrong password 5 times in a row | Account temporarily locked (15 min) with message; further attempts show lockout message even with correct password until cooldown expires | P1 | Negative/Security | Web/Mobile |
| TC-AUTH-013 | Login blocked for unverified email | Account created, email not yet verified | Unverified account credentials | Attempt login | Message: "Please verify your email to continue" with resend-verification option; not logged in | P1 | Negative | Web/Mobile |
| TC-AUTH-014 | "Remember me" persists session across app restarts | Valid account | — | Login with "Remember me" checked, fully close and reopen app/browser | User remains logged in | P2 | Positive | Web/Mobile |
| TC-AUTH-015 | Login as caregiver vs. customer routes to correct home screen | Both account types exist | Caregiver + customer accounts | Log in as each role separately | Caregiver → caregiver dashboard (bookings, availability); Customer → search/home screen | P0 | Positive | Web/Mobile |

## Logout & Session Handling

| ID | Title | Preconditions | Test Data | Steps | Expected Result | Priority | Type | Platform |
|---|---|---|---|---|---|---|---|---|
| TC-AUTH-020 | Successful logout | Logged in | — | Tap/click Logout | Session terminated, redirected to login/landing page, protected pages no longer accessible without re-login | P0 | Positive | Web/Mobile |
| TC-AUTH-021 | Session token invalidated server-side on logout | Logged in, session token captured (e.g., via DevTools) | Captured bearer token | 1. Log out 2. Replay a previously-captured authenticated API request using the old token (e.g., via Postman) | API should reject the request with 401 Unauthorized — token must not still work after logout | P0 | Security/Negative | Web (API) |
| TC-AUTH-022 | Expired session redirects to login, not a broken state | Logged in, session artificially expired (staging config) | — | Let session expire, then interact with the app | User is redirected to login with a clear "session expired" message; no silent failures or broken UI state | P1 | Negative | Web/Mobile |
| TC-AUTH-023 | Logout on one device does not affect other active sessions (unless "log out everywhere" is used) | Logged in on two devices | — | Log out on Device A only | Device B remains logged in and functional | P2 | Positive | Web/Mobile |

## Password Reset

| ID | Title | Preconditions | Test Data | Steps | Expected Result | Priority | Type | Platform |
|---|---|---|---|---|---|---|---|---|
| TC-AUTH-030 | Request password reset for valid account | Account exists | jane.doe.qa@example-mail.com | Enter email on "Forgot password", submit | Generic confirmation shown ("If that email exists, a reset link was sent") regardless of whether account exists (prevents email enumeration); reset email sent if account exists | P0 | Positive/Security | Web/Mobile |
| TC-AUTH-031 | Password reset link expires after set time window | Reset link generated >1 hour ago (or per policy) | Expired reset token | Click expired reset link | Message: "This link has expired, request a new one"; cannot set new password with expired token | P1 | Negative/Security | Web |
| TC-AUTH-032 | Password reset link is single-use | Valid, unused reset link | Reset token | 1. Use link to reset password successfully 2. Reuse the same link again | Second attempt fails with "This link is no longer valid" | P1 | Negative/Security | Web |
| TC-AUTH-033 | New password must meet strength policy on reset | Valid reset link | Weak new password: `abc123` | Attempt to set weak password via reset flow | Inline validation error; reset blocked until policy met | P2 | Negative | Web |

## Role-Based Access

| ID | Title | Preconditions | Test Data | Steps | Expected Result | Priority | Type | Platform |
|---|---|---|---|---|---|---|---|---|
| TC-AUTH-040 | Customer cannot access caregiver-only screens (e.g., availability calendar editor) via direct URL | Logged in as customer | Direct URL to caregiver dashboard route | Navigate directly to a caregiver-only URL while logged in as a customer | Access denied / redirected, not shown caregiver data or controls | P0 | Security/Negative | Web |
| TC-AUTH-041 | Caregiver cannot access another caregiver's private dashboard data via ID manipulation | Two caregiver accounts | Caregiver A logged in, Caregiver B's internal ID | Modify a dashboard URL/API call to reference Caregiver B's ID while logged in as Caregiver A | Request rejected (403) — no data leakage of Caregiver B's private info | P0 | Security/Negative | Web (API) |
