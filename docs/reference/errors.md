## Standard Error Envelope:
```json
{
  "error": {
    "type": "invalid_request_error",
    "code": "parameter_missing",
    "message": "The amount parameter is required.",
    "param": "amount"
  }
}
```
## HTTP Status Code Summary
| Status Code | Meaning | Description |
| :--- | :--- | :--- | 
| 400 | Bad request | Malformed syntax or missing required fields. |
| 401 | Unauthorized | Missing or invalid Bearer token. |
| 409 | Conflict | A request with the same idempotency key is actively processing. |
| 422 | Unprocessable Entity | Request payload is well-formed syntax, but cannot be processed semantically (e.g., payload mismatch or invalid currency). |
| 500/503 | Internal Error | Safe to retry with exponential backoff. |

## Error Code dictionary & Remediation:

| Code | HTTP | Cause | Developer Action |
| :--- | :--- | :--- | :--- |
| idempotency_conflict | 409 | Same key submitted while first transaction is in flight | Wait 2-5 seconds with exponential backoff; check transaction satus with `GET /charges/{id}`. |
| idempotency_mismatch | 422 | Key was previously used with a different payload (e.g., changed amount). | Generate a fresh UUIDv4 for a distinct transaction; do not reuse old keys across different charges. |
| invalid_currency | 422 | Currency code not supported. | Supply a valid ISO-4217 three-letter currency code (e.g.,usd,eur). |
| rate_limit_exceeded | 429 | Request volume exceeded quota. | Inspect `Retry-After` header and throttle queue workers. |
