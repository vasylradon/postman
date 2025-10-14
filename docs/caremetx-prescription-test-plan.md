# Caremetx Prescription Integration Test Plan Tracker

The detailed, execution-ready manual test plan for `POST /api/v1/caremetx/prescription-integration` is now maintained in the spreadsheet-friendly comma-separated file [`caremetx-prescription-test-plan.csv`](caremetx-prescription-test-plan.csv). Import the CSV into Excel, Google Sheets, or LibreOffice Calc to mark completion status, record evidence links, and assign owners during each test cycle while keeping the artefact diff-friendly in source control.

## Spreadsheet Overview

| Column | Purpose |
| --- | --- |
| `#` | Stable scenario identifier for traceability with requirements and bug reports. |
| `Scenario` | Short description of the test objective. |
| `Preconditions` | Data setup or environment assumptions that must be satisfied before execution. |
| `Steps` | High-level execution steps (Postman request, data seeding, downstream checks, etc.). |
| `Expected Results` | Observable outcomes required to pass the scenario (HTTP status, side effects, logging, workflow triggers). |
| `Status` | Free-form execution flag (e.g., Pass/Fail/Blocked) for each cycle. |
| `Notes` | Space for evidence links, log references, ticket numbers, or remediation details. |

The tracker contains 36 scenarios grouped across functional validation, data integrity, authorization, downstream workflow, and resilience/performance coverage. Scenarios 1–15 and 23–24 align exactly with the additional cases requested (e.g., successful ingestion, deduplication, authorization failures, and schema validation). Additional rows capture the remaining behaviors documented in prior iterations such as transaction retry handling, Boomi handshake, audit logging, and performance/regression considerations.

### Scenario Index

| # | Scenario |
| --- | --- |
| 1 | Successful new prescription ingestion |
| 2 | Patient update on existing match |
| 3 | Physician update on existing NPI |
| 4 | Facility deduplication |
| 5 | Case creation for new patient |
| 6 | Prescription creation for each call |
| 7 | Duplicate prescription handling |
| 8 | Missing required field (patient last name) |
| 9 | Invalid field format (phone number) |
| 10 | Invalid enumeration/identifier |
| 11 | Unauthorized request (no API key) |
| 12 | Unauthorized request (invalid API key) |
| 13 | Unauthorized request (empty API key value) |
| 14 | Payload schema validation |
| 15 | Graceful handling of empty optional sections |
| 16 | Missing top-level object validation |
| 17 | Client/program mismatch |
| 18 | Unsupported HTTP method |
| 19 | Unsupported media type |
| 20 | Large payload resilience |
| 21 | Retry/idempotency via transactionId |
| 22 | Partial downstream failure handling |
| 23 | Facility association to physician |
| 24 | Multiple physician addresses |
| 25 | Multiple physician phone numbers |
| 26 | Patient contact deduplication |
| 27 | Date normalization |
| 28 | Past prescription dates |
| 29 | Future-dated prescription checks |
| 30 | NDC to product mapping failure |
| 31 | Audit log creation |
| 32 | Security logging for failures |
| 33 | Performance baseline |
| 34 | Swagger contract alignment |
| 35 | Case workflow automation |
| 36 | Boomi integration handshake |

## How to Use

1. Open or import the CSV in Excel, LibreOffice Calc, or Google Sheets and enable filters/frozen header rows as desired for execution tracking.
2. Filter or sort by `Status` to manage daily execution and hand-offs between QA and engineering.
3. Capture evidence links or query references in the `Notes` column for traceability to logs, database extracts, or screenshots.
4. Update the tracker whenever endpoint behavior, validation rules, or downstream workflows change. Commit the revised CSV back to the repository to keep the canonical source synchronized.

For quick reference while reviewing defects or preparing summary reports, continue to use the accompanying documentation and Postman collection in this repository. Any future changes to the spreadsheet structure should preserve the existing scenario numbering to avoid breaking links from other artefacts.
