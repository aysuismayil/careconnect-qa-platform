# Evidence Notes — BUG-003 (Payment double charge on retry)

Referenced by [`bug-reports/BUG-003-payment-double-charge-retry.md`](../../bug-reports/BUG-003-payment-double-charge-retry.md).

- **`network-har-timeout-retry.har`** — HAR export from Chrome DevTools capturing both the original timed-out request and the subsequent manual retry, with auth tokens, cookies, and any PII stripped before saving (HAR files can contain full request/response bodies and headers, so sanitizing before export is a required step, not optional).
- **`db-payments-table-duplicate.png`** — synthetic recreation of two `payments` rows for a single booking attempt, captured from the SQL query result described in [`database-validation/sql-validation-queries.md`](../../database-validation/sql-validation-queries.md), Query 2.
- **`postman-repro-idempotency-check.json`** — Postman export of two `POST /payments/confirm` calls sent without any idempotency key, both returning `200` with distinct `chargeId` values — the evidence that pinned this down to a missing-idempotency-key root cause rather than a UI double-click issue.
