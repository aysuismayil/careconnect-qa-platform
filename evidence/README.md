# Evidence Structure

This folder documents **how** I organize and attach evidence to bug reports — it is not a dump of real production screenshots or exported files.

## Why evidence in this repo is described, not attached as binary files

Real bug evidence (screenshots, HAR files, session recordings, database exports) almost always contains information that shouldn't leave a company's systems — internal URLs, real or realistic-looking user data, internal tooling UI, API keys, or infrastructure details. Rather than fabricate fake screenshots that could be mistaken for real product UI, or strip a real one down until it's misleading, I've documented **what evidence I collected and why** in each bug report's "Evidence/Attachments" section, with descriptive filenames and folders, instead of including binary files here.

This mirrors how I actually work: every bug report I file links out to a Jira attachment or a shared evidence folder — the bug report text itself always describes precisely what the evidence shows and why it matters, which is the actual skill being demonstrated (knowing what evidence proves your bug, not just screenshotting something).

## Folder structure

```
evidence/
├── bug-001-search-radius/           Referenced by BUG-001
├── bug-002-double-booking/          Referenced by BUG-002
├── bug-003-payment-double-charge/   Referenced by BUG-003
├── bug-004-session-token/           Referenced by BUG-004
├── bug-005-price-mismatch-api/      Referenced by BUG-005
└── bug-006-mobile-duplicate-charge/ Referenced by BUG-006
```

Each folder contains a short `NOTES.md` describing exactly what evidence would live there and how it was captured — the same information a hiring manager or teammate would want before opening the actual file.

## What I actually capture as evidence, in practice

| Evidence type | When I use it | Tool |
|---|---|---|
| Screenshot | UI-visible bug state, especially with annotations pointing to the issue | Native OS screenshot / browser DevTools |
| Screen recording | Timing-dependent or multi-step bugs (race conditions, animations, intermittent issues) | QuickTime / built-in screen recorder |
| HAR file (network export) | Frontend/backend investigation, request/response inspection | Chrome DevTools → Network → "Save all as HAR" |
| Console log export | JavaScript errors, warnings | Chrome DevTools → Console |
| Postman export (collection run / response) | API-level reproduction, independent of UI | Postman → Export |
| SQL query result | Database-level confirmation of actual state | DBeaver / psql, exported as text/CSV |

## Sanitization rules I follow before attaching any evidence

- Redact or replace real user emails, names, phone numbers, and addresses with synthetic equivalents.
- Redact auth tokens, session cookies, and API keys entirely — never partially shown "just in case," always fully removed.
- Redact internal-only URLs, hostnames, and infrastructure details not meant for external sharing.
- Never attach production customer data, even redacted — staging/synthetic data only.
- Confirm screenshots don't incidentally capture unrelated sensitive info in browser tabs, bookmarks bar, or notifications.
