# Test Plan & Test Strategy — CareConnect Marketplace

**Document owner:** QA Engineer (Manual)
**Scope:** Web (responsive) + Mobile (iOS/Android) caregiving marketplace
**Status:** Sample/portfolio version, sanitized from real project work

---

## 1. Objective

Define what gets tested, how, by whom, and to what standard before each CareConnect release, so the team ships booking and payment flows with confidence and catches regressions before customers do.

## 2. In scope

| Area | Included |
|---|---|
| Authentication | Sign up, login, logout, password reset, session handling, role-based access (caregiver vs. customer) |
| Caregiver search & filtering | Location/radius search, filters (service type, price, rating, availability, gender preference), sorting, pagination |
| Caregiver profiles | Profile creation/editing, photo upload, certifications, service listings, availability calendar |
| Booking | Availability checks, booking creation, rescheduling, cancellation, booking status lifecycle |
| Payments | Payment method entry (via PCI-compliant vendor iframe, never touched directly), checkout, receipts, refunds, failed-payment handling |
| Mobile | Responsive web on mobile browsers, native app parity (iOS/Android) for the above flows |
| Notifications | Booking confirmations, reminders, payment receipts (email/push) |
| Accessibility | Keyboard navigation, screen reader support, color contrast on key flows |

## 3. Out of scope (owned elsewhere)

- Automated regression suite (owned by SDET team) — this repo documents **manual** QA ownership; I collaborated with the SDET on coverage decisions but did not write the automation framework itself.
- Infrastructure/load/performance testing (owned by a dedicated performance engineer).
- Internal admin/back-office tooling (separate QA owner).
- Payment processor's own PCI-scoped systems (we test our integration surface, not the processor's internals).

## 4. Test levels and approach

| Level | Approach | Who |
|---|---|---|
| Requirements review | Review user stories/AC in refinement before dev starts; raise gaps and edge cases | QA + PM + Dev |
| Functional (manual) | Structured test cases executed against staging before each release | QA (me) |
| Exploratory | Time-boxed, charter-based sessions targeting risk areas (booking, payments) | QA (me) |
| API | Direct endpoint testing via Postman, independent of UI, for contract and edge-case coverage | QA (me) |
| Database validation | Read-only SQL checks confirming backend state matches UI actions | QA (me) |
| Accessibility | Manual checks (keyboard, screen reader, contrast) on customer-facing flows | QA (me) |
| Smoke | Critical-path check immediately after each deploy to staging/production | QA (me) |
| Regression | Full pass across all major flows before a release, focused pass after any bug fix | QA (me) |
| Automated regression | Maintained separately by SDET; I flagged candidate cases for automation | SDET |

## 5. Test environments

| Environment | Purpose | Data |
|---|---|---|
| Local/Dev | Developer smoke checks | Synthetic seed data |
| Staging | Primary QA environment — full manual, API, exploratory, and DB testing | Anonymized/synthetic data, sandboxed payment processor (test mode) |
| Production (post-release) | Smoke test only, no destructive testing | Real data (read-only observation only) |

All payment testing is done against the payment provider's **sandbox/test mode** — no real card data or real charges are ever used in QA.

## 6. Entry criteria (before QA starts a feature)

- User story has clear, testable acceptance criteria (see [Requirements Review](02-requirements-and-acceptance-criteria-review.md)).
- Feature is deployed to staging and passes a basic developer smoke check.
- Test data / seed accounts are available in staging.
- Any third-party sandbox (payments, maps/geolocation) is configured and reachable.

## 7. Exit criteria (before a release ships)

- All P0/P1 test cases for the release scope pass.
- No open Critical or High severity bugs without an accepted risk sign-off from PM/Eng lead.
- Smoke suite passes 100% on the release candidate build.
- Regression suite passes on affected areas; any known issues are documented with workaround/risk notes.
- Accessibility spot-check passes on new/changed customer-facing screens.
- Release Readiness / Test Summary report is written and shared (see [`03-release-readiness-and-test-summary.md`](03-release-readiness-and-test-summary.md)).

## 8. Risk-based prioritization

I prioritize testing effort using business impact × likelihood of breakage:

| Priority | Example areas | Why |
|---|---|---|
| **P0 — highest risk** | Payments, booking creation/cancellation, authentication | Direct revenue and trust impact; failures here are business-critical |
| **P1** | Search/filtering, availability accuracy, profile data integrity | Core discovery flow — broken search directly reduces bookings |
| **P2** | Notifications, non-critical profile fields, sorting edge cases | Important but has workarounds or lower blast radius |
| **P3** | Cosmetic/UI polish issues that don't block a flow | Low urgency, fixed opportunistically |

This is the same priority scale I use when triaging bugs (see [bug report template](../templates/bug-report-template.md)).

## 9. Roles and responsibilities

| Role | Responsibility |
|---|---|
| QA Engineer (me) | Test planning, manual test execution, exploratory testing, API/DB validation, accessibility checks, bug reporting, release sign-off recommendation |
| Developers | Fix bugs, support root-cause investigation, unit/integration test coverage |
| SDET | Own and maintain the automated regression suite |
| Product Manager | Prioritize bug fixes, accept/reject risk on known issues before release |
| Designer | Confirm UI/UX intent when behavior is ambiguous |

## 10. Test deliverables

- Test cases (this repo: [`test-cases/`](../test-cases/))
- Bug reports (this repo: [`bug-reports/`](../bug-reports/))
- Exploratory session charters/notes (this repo: [`exploratory-testing/`](../exploratory-testing/))
- API test collection (this repo: [`api-testing/`](../api-testing/))
- Smoke/regression checklists (this repo: [`smoke-and-regression/`](../smoke-and-regression/))
- Release Readiness / Test Summary report per release (this repo: [`docs/03-release-readiness-and-test-summary.md`](03-release-readiness-and-test-summary.md))

## 11. Assumptions and constraints

- Staging closely mirrors production configuration but runs on synthetic data.
- Third-party integrations (maps/geocoding, payment gateway, push notifications) are tested against their official sandbox environments.
- Mobile testing covers the two most recent major OS versions for iOS and Android, plus a representative device/screen-size matrix (see [`test-cases/06-mobile-test-cases.md`](../test-cases/06-mobile-test-cases.md)).
