# Idempotency API Engine & Docs-as-Code Specification

A production-grade, modular OpenAPI 3.1 specification and technical documentation framework designed for distributed payment systems. It defines safe retry semantics using UUIDv4 idempotency keys, HMAC-SHA256 webhook verification, and automated Spectral CI contract enforcement.

---

## The Problem Solved

In high-concurrency payment integrations, network dropouts often prevent client applications from receiving transaction receipts. Blind retries can lead to double billing and ledger inconsistencies.

This engine enforces:
- **Idempotency Guarantees:** Replaying identical requests with the same `Idempotency-Key` returns the cached original result without re-executing charges.
- **Payload Integrity:** Guarding against accidental key collisions or parameter tampering via semantic checks.
- **Cryptographic Webhooks:** Validating delivery authenticity using HMAC-SHA256 and preventing replay attacks with Unix epoch timestamp verification.

---

## State Machine & Behavior Matrix

The `/charges` endpoint processes idempotent calls according to this deterministic state table:

| Scenario | HTTP Status | Behavior |
| :--- | :--- | :--- |
| **Initial Request** | `201 Created` | Payment processes normally. Idempotency key is registered. |
| **Exact Retry** | `200 OK` | Original response returned from cache. Balance untouched. |
| **Concurrent Request** | `409 Conflict` | Identical key is actively processing in-flight. Safe to retry with backoff. |
| **Payload Mismatch** | `422 Unprocessable` | Key was previously executed with a different amount or currency. |

---

## Architecture & Repository Structure

The contract is structured using modular OpenAPI 3.1 `$ref` components to prevent monolithic drift:

```text
project-1-idempotency-api-engine/
├── .github/workflows/
│   └── spectral-lint.yml        # Automated CI workflow validating OpenAPI specs
├── docs/
│   ├── guides/
│   │   ├── idempotency-keys.md  # Deep dive into key lifecycle, headers, and retries
│   │   └── verifying-webhooks.md# HMAC-SHA256 validation & replay attack prevention
│   └── reference/
│       └── errors.md            # Uniform error envelope and remediation dictionary
├── openapi/
│   ├── openapi.yaml             # Root contract aggregating components via $ref
│   └── components/
│       ├── parameters/          # Reusable header parameters (Idempotency-Key)
│       ├── responses/           # Standard 409 Conflict and 422 Unprocessable schemas
│       └── schemas/             # ChargeRequest and ChargeResponse definitions
└── .spectral.yaml               # Strict linting ruleset enforcing OpenAPI 3.1 standards
```

**Local Verification & Quickstart**
1. Install Dependencies
```bash
npm install
```

2. Lint Contract with Spectral
Ensure the spec complies with OpenAPI 3.1 rules and contains zero broken references:
```bash
npm run lint
```

3. Start Mock Server (Prism)
Spin up a local mock server using your OpenAPI contract:
```bash
npx @stoplight/prism-cli mock openapi/openapi.yaml
```
4. Execute a Test Charge
In a separate terminal, dispatch an idempotent payment request:
```bash
curl -X POST http://127.0.0.1:4010/charges \
  -H "Authorization: Bearer test_key_12345" \
  -H "Idempotency-Key: 9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 4999,
    "currency": "usd",
    "customer_id": "cust_98765"
  }'
```
Returns:

```json
{
  "id": "ch_3MtwxPEby7qXAgqq1T62Zq1",
  "status": "succeeded",
  "amount": 1000,
  "currency": "usd"
}
```
**Continuous Integration (CI)**:
Every commit and pull request to main runs an automated GitHub Action executing Spectral linting. This prevents syntax regressions, broken $ref pointers, and non-standard HTTP responses from reaching production.
