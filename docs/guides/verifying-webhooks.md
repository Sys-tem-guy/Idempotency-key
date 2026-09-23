# Verifying Webhook Signatures

Webhook endpoints are exposed, publicly accessible HTTP URLs. Without cryptographic verification, anyone can send spoofed `POST` payloads pretending a payment succeeded, resulting in unauthorized order fulfillment.

This API signs every outgoing webhook notification with an HMAC-SHA256 signature using a shared secret known only to your server and the gateway.

---

## Signature Headers

Every incoming webhook event includes two verification headers:

| Header | Description |
| :--- | :--- |
| `X-Signature-Timestamp` | Unix epoch timestamp (in seconds) indicating when the event was dispatched. Used to mitigate **replay attacks**. |
| `X-Signature-SHA256` | The HMAC-SHA256 hex digest generated from the timestamp, a dot delimiter, and the raw payload body. |

> **Replay Attack Defense:** Your server must reject any webhook where `current_timestamp - X-Signature-Timestamp > 300` (older than 5 minutes).

---

## Verification Algorithm

Follow these 4 steps to authenticate incoming requests:

1. **Capture Raw Body:** Extract the raw request payload buffer before running any JSON parsing middleware. Changing whitespace, line breaks, or key ordering will cause the hash check to fail.
2. **Read Headers:** Extract `X-Signature-Timestamp` and `X-Signature-SHA256`.
3. **Compute Digest:** Compute the HMAC-SHA256 hash using your secret signing key:
   $$\text{expected\_signature} = \text{HMAC-SHA256}(\text{key} = \text{webhook\_secret}, \text{data} = \text{timestamp} + \text{"."} + \text{raw\_body})$$
4. **Constant-Time Comparison:** Compare the computed signature against the header value using a constant-time comparison utility (such as `crypto.timingSafeEqual`) to prevent side-channel timing attacks.

---

## Implementation Example (Node.js)

```javascript
const crypto = require('crypto');

function verifyWebhook(rawBody, headers, webhookSecret) {
  const timestamp = headers['x-signature-timestamp'];
  const receivedSignature = headers['x-signature-sha256'];

  // 1. Verify timestamp is within tolerance (5 minutes)
  const currentTime = Math.floor(Date.now() / 1000);
  if (Math.abs(currentTime - parseInt(timestamp, 10)) > 300) {
    throw new Error('Webhook timestamp outside acceptable tolerance.');
  }

  // 2. Compute expected HMAC-SHA256 signature
  const payloadToSign = `${timestamp}.${rawBody}`;
  const computedSignature = crypto
    .createHmac('sha256', webhookSecret)
    .update(payloadToSign, 'utf8')
    .digest('hex');

  // 3. Constant-time comparison to prevent timing attacks
  const signatureBuffer = Buffer.from(receivedSignature, 'utf8');
  const computedBuffer = Buffer.from(computedSignature, 'utf8');

  if (
    signatureBuffer.length !== computedBuffer.length ||
    !crypto.timingSafeEqual(signatureBuffer, computedBuffer)
  ) {
    throw new Error('Invalid webhook signature.');
  }

  return true;
}
```

## Return Receipt (200 OK)
Once verified and enqueued for processing, return an HTTP 200 OK status immediately. If your server returns a 4xx or 5xx code, or times out, the gateway will retry delivery with exponential backoff.