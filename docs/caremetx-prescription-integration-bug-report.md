# Caremetx Prescription Integration - Webhook Contract Mismatch

## Summary
The endpoint `POST /api/v1/caremetx/prescription-integration` rejects the raw MedVantx webhook payload that is documented in the integration references. The response indicates that the P3-specific `Patient`, `Physician`, and `Prescription` sections are required at the root of the request body. Unless Caremetx performs an additional transformation, the data push they receive from MedVantx cannot be forwarded to P3 without a 400 validation error.

## Steps to Reproduce
1. Set the `Authorization` header to `ApiKey {valid key}`.
2. Send the MedVantx sample webhook JSON (subscription, transactionId, webhookBody with nested patient/prescriber data) as the request body.
3. Inspect the response.

## Actual Result
- HTTP 400 response with RFC 7807 payload.
- `errors` contains entries for `Patient`, `Physician`, and `Prescription` stating that each field is required.

## Expected Result
- Either the endpoint should accept the documented MedVantx webhook payload directly, or integration documentation should state that Caremetx must transform the webhook into the P3 contract before invoking the endpoint.

## Impact
High. Without clarity or transformation, the direct webhook payload causes ingestion to fail, blocking the Caremetx-to-P3 eRx integration.

## Recommendation
Align the endpoint contract and integration documentation by either:
- Updating the endpoint to accept the MedVantx webhook payload (and internally map to P3 structures), **or**
- Explicitly documenting that Caremetx must enrich and reshape the MedVantx webhook into the P3 contract prior to submission, including example payloads and validation requirements.
