# Caremetx Prescription Integration Test Plan

This manual test plan covers functional validation of the `POST /api/v1/caremetx/prescription-integration` endpoint that receives eRx payloads from Caremetx. The goal is to verify that the endpoint satisfies the business and technical requirements defined for the Idorsia program and provides predictable error handling for invalid requests.

## 1. Test Environment & Prerequisites

- Deployed P3 environment with the Caremetx prescription integration endpoint enabled.
- Valid API key provisioned for the Caremetx integration client.
- Access to underlying P3 data sources (database or API) to confirm entity creation/updates for patients, physicians, facilities, cases, and prescriptions.
- Ability to reset or seed test data so that scenarios can be executed deterministically (e.g., clearing previously created patients with specific identifiers).
- Postman (or similar REST client) configured with the accompanying automated tests collection and environment variables:
  - `base_url`
  - `apiKey`
  - `invalidApiKey`

## 2. Functional Test Scenarios

The scenarios below align with the stated business logic, validation rules, and negative cases. They are organized into logical groupings so teams can plan execution waves while ensuring full coverage.

### 2.1 Core Happy Path & Idempotency

| # | Scenario | Preconditions | Steps | Expected Results |
|---|----------|---------------|-------|------------------|
| 1 | Successful prescription ingestion | No conflicting patient/physician/facility exists for the provided identifiers | Send valid payload with populated patient, physician, facility, and prescription objects | HTTP 201 (or 200/204 per implementation); JSON envelope returns `value`, `code`, `success`, `message`; `value` contains `prescriptionId`, `caseId`, `success`, `message`; patient, physician, facility, case, and prescription are created in P3; external reference IDs stored; entities associated correctly |
| 2 | Patient update on existing match | Patient exists with same first name, last name, and DOB for target client/program | Send payload with updated patient demographic info | HTTP 200/204; response envelope matches schema with `success=true`; existing patient updated with new values; `ExternalReferenceId` matches incoming patient ID; no duplicate patient created |
| 3 | Physician update on existing NPI | Physician exists with same NPI | Send payload with changed physician details | HTTP 200/204; physician information updated; no duplicate physician created |
| 4 | Facility deduplication | Facility exists with matching Address1, State, Zip | Send payload referencing existing facility but updated secondary fields | HTTP 200/204; facility updated in place; association with physician maintained |
| 5 | Case creation for new patient | Patient does not exist yet | Send payload that creates new patient | New case created for patient; case linked to prescription; workflow triggered according to Idorsia rules |
| 6 | Case remains unchanged for existing patient | Patient already exists with same identifiers | Re-send payload with unchanged patient | No duplicate case created; existing case remains associated to prescription |
| 7 | Prescription creation for each call | Existing case (new or existing patient) | Send payload | New prescription record created and associated with case; NDC mapped to product |
| 8 | Duplicate prescription handling | Send same payload twice | First call creates entities; second call updates existing records without creating duplicates; follow-up response envelope includes message such as "Prescription already exists" |

### 2.2 Validation & Error Handling

| # | Scenario | Preconditions | Steps | Expected Results |
|---|----------|---------------|-------|------------------|
| 9 | Missing required field (patient last name) | None | Remove patient last name and submit | HTTP 400 with RFC 7807 payload; `errors['Patient.LastName']` contains "The LastName field is required." |
| 10 | Missing required sections entirely | None | Drop the `patient` object from the payload | HTTP 400 with RFC 7807 payload; errors identify `Patient`, `Physician`, and `Prescription` as required |
| 11 | Invalid field format (phone number) | None | Provide invalid patient phone (e.g., alphabetic) | HTTP 400 with error referencing phone format requirement |
| 12 | Invalid enumeration/identifier | None | Use unsupported `state` value or non-numeric `programId` | HTTP 400 with descriptive error |
| 13 | Boundary testing for max field lengths | None | Send longest supported values (names, addresses, sig text) | HTTP 200/204 for valid lengths; HTTP 400 when length exceeds configured maximum |
| 14 | Payload schema validation | None | Send payload with unexpected data type (e.g., `patientId` as string) | HTTP 400 with RFC 7807 payload listing offending fields in `errors` |
| 15 | Empty optional arrays | None | Provide empty `addresses` / `phones` arrays | HTTP 200/204; request succeeds and downstream data not overwritten incorrectly |
| 16 | Null optional values | None | Set optional string fields to `null` | HTTP 200/204 if allowed; otherwise HTTP 400 with descriptive validation error |
| 17 | Raw MedVantx webhook payload | None | Submit raw MedVantx webhook sample without P3 contract wrapper | HTTP 400 with RFC 7807 payload; `errors` includes keys for `Patient`, `Physician`, and `Prescription` explaining missing sections |

### 2.3 Authorization & Transport

| # | Scenario | Preconditions | Steps | Expected Results |
|---|----------|---------------|-------|------------------|
| 18 | Unauthorized request (no API key) | None | Omit `Authorization` header | HTTP 401 with consistent error payload |
| 19 | Unauthorized request (invalid API key) | None | Provide malformed or expired API key | HTTP 401 with consistent error payload |
| 20 | Unauthorized request (empty API key value) | None | Provide header `Authorization: ApiKey` with blank value | HTTP 401 with consistent error payload |
| 21 | Incorrect auth scheme | None | Send header `Authorization: Bearer {{apiKey}}` | HTTP 401 with consistent error payload |
| 22 | Missing or incorrect `Content-Type` header | None | Send request without `Content-Type` or with unsupported type | HTTP 415 or 400 depending on platform standard |

### 2.4 Integration & Downstream Effects

| # | Scenario | Preconditions | Steps | Expected Results |
|---|----------|---------------|-------|------------------|
| 23 | Facility association to physician | Physician exists without facility | Send payload including new facility details | Facility created and linked to physician |
| 24 | Multiple physician addresses | Physician requires more than one address | Include multiple addresses | All addresses persisted or appropriate warnings returned; duplicates not created |
| 25 | Multiple physician phones | Provide more than one phone entry | Determine how endpoint handles additional phones (ignored, stored, or mapped) and confirm consistent behavior |
| 26 | Product NDC mapping failure | Provide NDC not in product catalog | Endpoint either rejects request with descriptive error or creates prescription with default/unknown product per requirements |
| 27 | Workflow trigger verification | Platform configured with Idorsia workflow | After successful ingestion, confirm workflow/task creation in P3 |
| 28 | Rollback on partial failure | Force downstream error after patient creation (e.g., database constraint for prescription) | Confirm transaction either rolls back prior entities or leaves system in consistent state with clear error response |
| 29 | Audit trail and logging | None | Process success and failure requests | Audit records/logs capture transactionId, patientId, and traceId for troubleshooting |

### 2.5 Resilience, Performance, and Security Hardening

| # | Scenario | Preconditions | Steps | Expected Results |
|---|----------|---------------|-------|------------------|
| 30 | High-volume ingestion | Load test with expected peak volume | Endpoint maintains performance SLAs; no throttling or data loss |
| 31 | Concurrent duplicate submissions | Send identical payloads simultaneously | Only one prescription created; others treated as updates |
| 32 | Large payload handling | Include max-length arrays for addresses/phones/sig text | Request processed successfully within size limits |
| 33 | Rate limiting / throttling | Exceed documented request rate (if applicable) | API returns throttling response (429) with retry headers |
| 34 | Security scanning | Run automated security tests (e.g., fuzzing, injection attempts in text fields) | Endpoint rejects malicious input and logs appropriately |
| 35 | Traceability of failures | Trigger validation and server errors | Trace IDs correlate with logs/monitoring for diagnostics |
| 36 | Disaster recovery / retry behavior | Simulate temporary downstream outage | Endpoint returns retriable error; replays succeed once systems restored |

## 3. Data Validation Checklist

For scenarios that create or update entities, verify within P3 that:

- Patient demographics, contact information, language, and `ExternalReferenceId` match the payload.
- Physician record is linked to the facility; facility details match the payload or are updated appropriately.
- Facility deduplication uses Address1 + State + 5-digit Zip as keys; associations to physicians persist.
- Case is created only when a new patient is ingested and the expected workflow is triggered.
- Prescription is created (or updated per design) and linked to the correct case; NDC is mapped to the product catalog.
- No duplicate records are created for patient, physician, or facility when identifiers match existing data.

## 4. Error Handling Expectations

- Validation errors return HTTP 400 with RFC 7807 `application/problem+json` payloads that include `type`, `title`, `status`, `traceId`, and an `errors` object describing field-level issues (e.g., `Patient.LastName`, `Patient`, `Physician`).
- Authentication failures return HTTP 401 with a consistent error schema.
- Responses include useful diagnostic context (e.g., `traceId`, `timestamp`) when provided by platform standards.
- Successful responses return HTTP 200, 201, or 204 depending on create vs. update behavior. When a body is returned it uses the envelope `{ value: { prescriptionId, caseId, success, message }, code, success, message }` and `success` remains `true`.

## 5. Regression Considerations

- Re-test the collection whenever validation rules or downstream entity creation logic changes.
- Include smoke tests for the endpoint in CI/CD pipelines using the Postman collection with environment-specific variables.
- Monitor logs or application telemetry during execution to confirm workflow triggers and to capture unexpected errors.

