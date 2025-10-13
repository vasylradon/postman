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

The scenarios below align with the stated business logic, validation rules, and negative cases.

| # | Scenario | Preconditions | Steps | Expected Results |
|---|----------|---------------|-------|------------------|
| 1 | Successful prescription ingestion | No conflicting patient/physician/facility exists for the provided identifiers | Send valid payload with populated patient, physician, facility, and prescription objects | HTTP 201 (or 200/204 per implementation); response conforms to contract; patient, physician, facility, case, and prescription are created in P3; external reference IDs stored; entities associated correctly |
| 2 | Patient update on existing match | Patient exists with same first name, last name, and DOB for target client/program | Send payload with updated patient demographic info | HTTP 200/204; existing patient updated with new values; `ExternalReferenceId` matches incoming patient ID; no duplicate patient created |
| 3 | Physician update on existing NPI | Physician exists with same NPI | Send payload with changed physician details | HTTP 200/204; physician information updated; no duplicate physician created |
| 4 | Facility deduplication | Facility exists with matching Address1, State, Zip | Send payload referencing existing facility but updated secondary fields | HTTP 200/204; facility updated in place; association with physician maintained |
| 5 | Case creation for new patient | Patient does not exist yet | Send payload that creates new patient | New case created for patient; case linked to prescription; workflow triggered according to Idorsia rules |
| 6 | Prescription creation for each call | Existing case (new or existing patient) | Send payload | New prescription record created and associated with case; NDC mapped to product |
| 7 | Duplicate prescription handling | Send same payload twice | First call creates entities; second call updates existing records without creating duplicates; prescription behavior confirmed per design (new or updated) |
| 8 | Missing required field (patient last name) | None | Remove patient last name and submit | HTTP 400 with descriptive validation error referencing missing field |
| 9 | Invalid field format (phone number) | None | Provide invalid patient phone (e.g., alphabetic) | HTTP 400 with error referencing phone format requirement |
| 10 | Invalid enumeration/identifier | None | Use unsupported `state` value or non-numeric `programId` | HTTP 400 with descriptive error |
| 11 | Unauthorized request (no API key) | None | Omit `Authorization` header | HTTP 401 with consistent error payload |
| 12 | Unauthorized request (invalid API key) | None | Provide malformed or expired API key | HTTP 401 with consistent error payload |
| 13 | Unauthorized request (empty API key value) | None | Provide header `Authorization: ApiKey` with blank value | HTTP 401 with consistent error payload |
| 14 | Payload schema validation | None | Send payload with unexpected data type (e.g., `patientId` as string) | HTTP 400 with descriptive validation error |
| 15 | Graceful handling of empty optional sections | None | Omit optional objects (e.g., `physician.addresses`) while meeting minimum requirements | HTTP 200/204; request succeeds and optional data remains untouched |
| 16 | Response contract compliance | Execute scenarios that return both success and error | Inspect responses to confirm JSON structure, error messaging, and headers comply with Swagger definition |

## 3. Data Validation Checklist

For scenarios that create or update entities, verify within P3 that:

- Patient demographics, contact information, language, and `ExternalReferenceId` match the payload.
- Physician record is linked to the facility; facility details match the payload or are updated appropriately.
- Facility deduplication uses Address1 + State + 5-digit Zip as keys; associations to physicians persist.
- Case is created only when a new patient is ingested and the expected workflow is triggered.
- Prescription is created (or updated per design) and linked to the correct case; NDC is mapped to the product catalog.
- No duplicate records are created for patient, physician, or facility when identifiers match existing data.

## 4. Error Handling Expectations

- Validation errors return HTTP 400 and include field-level messages indicating which requirement failed (missing, format, range, enumeration, etc.).
- Authentication failures return HTTP 401 with a consistent error schema.
- Responses include useful diagnostic context (e.g., `traceId`, `timestamp`) when provided by platform standards.
- Successful responses return HTTP 200, 201, or 204 depending on create vs. update behavior and include body content matching the API specification when applicable.

## 5. Regression Considerations

- Re-test the collection whenever validation rules or downstream entity creation logic changes.
- Include smoke tests for the endpoint in CI/CD pipelines using the Postman collection with environment-specific variables.
- Monitor logs or application telemetry during execution to confirm workflow triggers and to capture unexpected errors.

