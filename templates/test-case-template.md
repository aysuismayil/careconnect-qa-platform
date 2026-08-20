# Test Case Template

Format used across all files in [`test-cases/`](../test-cases/).

| Field | Description |
|---|---|
| **Test Case ID** | Unique ID, e.g. `TC-AUTH-001` |
| **Title** | Short description of what's being verified |
| **Feature Area** | e.g. Authentication, Search, Booking |
| **Preconditions** | State required before test starts |
| **Test Data** | Specific input values (synthetic) |
| **Steps** | Numbered actions |
| **Expected Result** | What should happen |
| **Priority** | P0/P1/P2/P3 (business urgency, see [bug template](bug-report-template.md)) |
| **Type** | Positive / Negative / Boundary / Edge |
| **Platform** | Web / iOS / Android / All |

Test cases are grouped by feature area into tables for readability, with the same columns.
