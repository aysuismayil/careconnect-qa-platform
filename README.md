# CareConnect QA Platform

A portfolio repository documenting my QA work on **CareConnect**, a caregiving services marketplace (Web + Mobile) that connects caregivers (elder care, childcare, pet care, and companion care providers) with customers who search for, book, and pay for care services.

> **Note on this repository:** CareConnect is a sanitized, fictionalized name I use to present my real QA experience without exposing confidential company information, customer data, or proprietary systems. All test data, bug reports, screenshots, and evidence in this repo are **synthetic examples I recreated from memory** to demonstrate my QA process, tooling, and thinking — not exported artifacts from any employer's systems. No real user data, internal URLs, or production evidence is included anywhere in this repo.

I did not build or develop this application. My role was **Manual QA Engineer** — I owned quality for the caregiver and customer-facing flows described below, working closely with developers, product managers, and designers.

---

## About the product (for context)

CareConnect is a two-sided marketplace:

- **Caregivers** create a profile (bio, certifications, services offered, hourly rates, availability calendar) and get discovered by customers.
- **Customers** search for caregivers using filters (location/radius, service type, price range, rating, availability), view profiles, book a service for a specific date/time, and pay through the platform.

Core flows I tested: account registration & login, caregiver profile creation/editing, caregiver search & filtering, availability & booking, in-app payments, notifications, and the responsive/mobile experience (mobile web + native app parity).

---

## My role and contributions

I was the primary Manual QA Engineer embedded with a cross-functional squad (2 backend devs, 2 frontend devs, 1 PM, 1 designer). My day-to-day responsibilities included:

- **Requirements review** — reviewing user stories and acceptance criteria before development started, flagging gaps, ambiguities, and missing edge cases in refinement/grooming sessions.
- **Test planning & strategy** — writing test plans per release/feature, deciding what to cover manually vs. what belonged in automation (owned by a separate SDET), and defining entry/exit criteria.
- **Manual functional testing** — writing and executing structured test cases across web and mobile for authentication, search & filtering, profiles, booking, and payments.
- **Exploratory testing** — running time-boxed, charter-based exploratory sessions to find issues structured test cases miss, especially around booking edge cases and payment flows.
- **API testing** — using Postman to test and validate backend endpoints directly (independent of the UI), including negative/edge cases and contract checks.
- **Database validation** — writing read-only SQL queries against a staging database to confirm that UI actions produced the correct backend state (bookings, payment records, status transitions).
- **Accessibility testing** — manual keyboard navigation, screen reader spot checks, and WCAG 2.1 AA color-contrast checks on key customer-facing flows.
- **Bug reporting & triage** — writing detailed, reproducible bug reports in Jira, working with developers to reproduce and investigate root cause (including frontend/backend/API investigations using browser DevTools and Postman), and re-testing fixes.
- **Smoke & regression testing** — owning the pre-release smoke suite and regression pass before each production deploy, and signing off on release readiness.
- **Release sign-off** — writing test summary reports and communicating go/no-go recommendations to the team before releases.

---

## Repository structure

```
careconnect-qa-platform/
├── docs/                        Test strategy, requirements review, release readiness
├── test-cases/                  Manual test cases by feature area (web + mobile)
├── bug-reports/                 Detailed bug reports (Jira-style, sanitized)
├── exploratory-testing/         Session charters and exploratory testing notes
├── api-testing/                 Postman collection + API-level test cases
├── database-validation/         SQL validation queries used during testing
├── accessibility-testing/       Manual accessibility test cases and findings
├── smoke-and-regression/        Smoke suite and regression suite
├── evidence/                    Structure for evidence per bug (sanitized/synthetic)
└── templates/                   Reusable bug report and test case templates
```

### Where to start

| I want to see... | Go to |
|---|---|
| How I plan and scope testing | [`docs/01-test-plan-and-strategy.md`](docs/01-test-plan-and-strategy.md) |
| How I review requirements before dev starts | [`docs/02-requirements-and-acceptance-criteria-review.md`](docs/02-requirements-and-acceptance-criteria-review.md) |
| My manual test cases | [`test-cases/`](test-cases/) |
| My strongest bug reports | [`bug-reports/`](bug-reports/) |
| How I test outside of scripted cases | [`exploratory-testing/`](exploratory-testing/) |
| How I test APIs directly | [`api-testing/`](api-testing/) |
| How I validate data at the DB level | [`database-validation/`](database-validation/) |
| How I check accessibility | [`accessibility-testing/`](accessibility-testing/) |
| How I run smoke/regression before a release | [`smoke-and-regression/`](smoke-and-regression/) |
| My release sign-off process | [`docs/03-release-readiness-and-test-summary.md`](docs/03-release-readiness-and-test-summary.md) |

---

## Tools I used on this project

- **Test management / bug tracking:** Jira, Confluence
- **API testing:** Postman
- **Database validation:** SQL (PostgreSQL) via read-only staging access, DBeaver
- **Browser tooling:** Chrome DevTools (Network, Console, Application tabs) for frontend/backend investigation
- **Mobile testing:** iOS/Android physical devices + simulators, BrowserStack for cross-device coverage
- **Accessibility:** axe DevTools, VoiceOver (iOS), TalkBack (Android), keyboard-only navigation
- **Version control / docs:** Git, Confluence, Google Docs/Sheets

---

## A note on data and evidence

Everything in this repository — user names, emails, booking IDs, amounts, SQL results, screenshots described in evidence folders — is **synthetic sample data** built to illustrate real testing scenarios and real bugs I found, without exposing my employer's confidential information. See [`evidence/README.md`](evidence/README.md) for details on how evidence is organized and sanitized.
