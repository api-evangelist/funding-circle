---
name: Submit and track a Funding Circle loan application
description: Authenticate, submit a term business loan application, poll for the decision, and handle document requests using the Funding Circle Introducer API.
api: openapi/funding-circle-introducer-openapi.yml
operations:
  - createLoanApplication
  - getLoanApplicationStatus
  - getLoanApplication
  - getLoanApplicationDecision
  - getDocumentUploadUrl
  - submitRequiredAction
---

# Submit and track a Funding Circle loan application

Use the Funding Circle Introducer API to submit a UK small-business term loan
application and follow it through to a credit decision.

## Authenticate
1. Base64-encode `client_id:client_secret` and POST it to the token endpoint
   (sandbox: `https://auth.sandbox.api.fundingcircle.co.uk/oauth2/token`) using
   the OAuth 2.0 client-credentials grant.
2. Use the returned `access_token` (valid 24 hours) as
   `Authorization: Bearer {access_token}` on every request except the presigned
   document-upload PUT.

## Submit the application
3. Call **createLoanApplication** (`POST /loan_application`) with
   `Content-Type: application/json`. Provide `business_reference`, `ebo_list`,
   `customer_info`, `business_info`, `loan_info`, and `loan_purpose`. Include an
   optional `callback_url` to be notified asynchronously. Store the returned
   `loan_application_uuid`.

## Track the decision
4. Poll **getLoanApplicationStatus**
   (`GET /loan_application_status/{loan_application_uuid}`) — HTTP 202 means
   processing is ongoing, HTTP 303 means it is complete. Prefer the
   `callback_url` webhook where possible and use polling as a fallback.
5. When complete, call **getLoanApplication**
   (`GET /loan_application/{loan_application_uuid}`). The `status` is
   `in_progress`, `action_required`, or `decision_made`.
6. On `decision_made`, call **getLoanApplicationDecision**
   (`GET /loan_application/{loan_application_uuid}/decision`). The decision
   `status` is `provisional_offer`, `offered`, or `rejected`.

## Handle a document request
7. On `action_required`, call **getDocumentUploadUrl**
   (`GET /loan_application/{loan_application_uuid}/document_upload_url`) to get a
   presigned S3 URL (expires in 30 seconds). PUT the PDF (max 100 MB) to that URL
   **without** the bearer token.
8. Call **submitRequiredAction**
   (`PATCH /loan_application/{loan_application_uuid}/submit_required_action`) to
   resume processing, then return to step 4.

## Rules
- Errors follow RFC 7807 (`application/problem+json`); read `x-amzn-trace-id`
  for support. See `errors/funding-circle-problem-types.yml`.
- No idempotency key is documented — do not blindly retry
  `createLoanApplication`; check status first to avoid duplicate applications.
- Test in sandbox with the documented `companies_house_id` and `business_name`
  values in `sandbox/funding-circle-sandbox.yml`.
