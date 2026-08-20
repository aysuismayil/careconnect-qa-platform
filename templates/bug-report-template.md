# Bug Report Template

This is the exact structure I use for every bug report (mirrors how I file bugs in Jira). Every bug in [`bug-reports/`](../bug-reports/) follows this format.

---

**Title/Summary:**
One line, specific, includes the feature area and the failure. Format: `[Area] Short description of what's wrong`

**Short Description:**
1–3 sentences summarizing the bug and its impact in plain language.

**Preconditions:**
State the system, account, and data conditions needed before starting the steps.

**Steps to Reproduce:**
Numbered, specific, minimal steps. Should be reproducible by anyone with no extra context.

**Actual Result:**
What actually happened, factually and precisely.

**Expected Result:**
What should have happened, ideally tied back to acceptance criteria or spec.

**Test Data:**
Exact sample data used (accounts, values, IDs) so the bug is reproducible — always synthetic/sanitized.

**Environment/Configuration:**
Build/version, environment (staging/prod), browser/OS/device, feature flags if relevant.

**Evidence/Attachments:**
Screenshots, screen recordings, HAR files, console logs, API responses, or SQL query results. Reference file names/paths, described but sanitized.

**Severity:**
Technical impact of the bug (Critical / High / Medium / Low) — see scale below.

**Priority:**
Business urgency to fix (P0 / P1 / P2 / P3) — see scale below.

**Repro Rate:**
How consistently the bug reproduces (e.g., "5/5", "intermittent, ~30%").

**Workaround:**
Is there a way for the user or support team to avoid/mitigate the issue right now? If none, state "None identified."

**Investigation Notes:**
Any root-cause investigation I did myself — DevTools findings, API request/response details, SQL results, hypotheses — to help the developer triage faster.

---

## Severity scale

| Severity | Meaning |
|---|---|
| **Critical** | System crash, data loss/corruption, security issue, or complete blocker of a core flow (booking, payment, login) with no workaround |
| **High** | Major functionality broken or significantly degraded; workaround may exist but is painful |
| **Medium** | Feature partially broken or behaves incorrectly in non-critical ways; workaround exists |
| **Low** | Cosmetic, minor UX inconsistency, or edge case with negligible business impact |

## Priority scale

| Priority | Meaning |
|---|---|
| **P0** | Fix immediately / blocks release |
| **P1** | Fix before this release ships |
| **P2** | Fix in the next 1–2 sprints |
| **P3** | Backlog, fix when convenient |

Severity and priority are independent — a **Low severity, P0 priority** bug can exist (e.g., a typo on a legal/compliance page that must ship immediately), and a **Critical severity, P2 priority** bug can exist (e.g., a crash in a rarely-used admin tool with a workaround and no user-facing impact this sprint).
