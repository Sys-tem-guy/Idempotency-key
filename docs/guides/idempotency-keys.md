# Handling Payment Idempotency
When integrating payment APIs, network connections can drop after the server processes the transaction, preventing the client from receiving confirmation. Additionally, users may submit the checkout form multiple times during latency spikes.

This API uses **Idempotency Keys** to guarantee that sending identical requests produces the exact same result without duplicate charges.

---

## How it works
1. The client generates a unique **UUIDv4** string.
2. The client attaches this string in the `Idempotency-Key` header of a `POST /charges` request.
3. The server checks if this key is processed within the last 24 hours:
- **First request**: The server processes the payment and caches the reponse.
- **Identical retry**: The server returns the original cached response immediately.

---

## Behaviour Matrix:

| Scenario | HTTP Status | Behavior |
| :--- | :--- | :--- |
| **New Transaction** | `201 Created` | Payment executes normally. Key is registered. |
| **Exact Retry** | `200 OK` | Cached response returned. No funds are deducted. |
| **Concurrent Request** | `409 Conflict` | A request with this key is already processing. Wait and retry. |
| **Payload Mismatch** | `422 Unprocessable` | Key was reused with different parameters (e.g., amount or currency). |

---

## Quickstart Request

Generate a UUIDv4 and include it in the `Idempotency-key` header:

 ```bash
curl -X POST (https://api.example.com/charges) \
  -H "Authorization: Bearer test_key_12345" \
  -H "Idempotency-key: 9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 4999,
    "currency": "usd",
    "customer_id": "cust_98765"
  }'
```

## Expected Output
```bash
{"id":"ch_3MtwxPEby7qXAgqq1T62Zq1","status":"succeeded","amount":1000,"currency":"usd"}
```