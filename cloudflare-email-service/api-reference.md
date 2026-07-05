# Cloudflare Email Service Reference

This reference summarizes the bundled Cloudflare Email Service docs used to build the skill. If rollout decisions depend on pricing tier, beta status, or current product limits, verify the latest vendor docs before shipping.

## Service Snapshot

- The bundled docs describe Email Service as available on the Workers Paid plan.
- The bundled docs also describe the product as being in private beta at the time of writing.
- Sender domains must use Cloudflare DNS.
- Hosted email templates are not available in the bundled docs snapshot.
- The service covers both outbound email sending and inbound email routing.

## Choosing the API Surface

| Runtime | Preferred API | Notes |
| --- | --- | --- |
| Cloudflare Worker | `send_email` binding | Preferred for outbound email from Worker code |
| Any non-Worker backend | REST API | Requires account ID plus Cloudflare API token |
| Incoming mail into a Worker | `email(message, env, ctx)` handler | Used with Cloudflare Email Routing |
| Existing MIME-based integration | Legacy `EmailMessage` API | Supported, but not the default for new work |

## Worker Binding Setup

Configure outbound email bindings in Wrangler:

```jsonc
{
  "send_email": [
    {
      "name": "EMAIL"
    },
    {
      "name": "RESTRICTED_EMAIL",
      "allowed_sender_addresses": [
        "noreply@yourdomain.com",
        "support@yourdomain.com"
      ]
    }
  ]
}
```

```toml
[[send_email]]
name = "EMAIL"

[[send_email]]
name = "RESTRICTED_EMAIL"
allowed_sender_addresses = ["noreply@yourdomain.com", "support@yourdomain.com"]
```

Use `allowed_sender_addresses` when you want Wrangler to reject unexpected sender identities at the binding layer.

## Worker API

### Methods

- `env.EMAIL.send(message)` sends one email.
- `env.EMAIL.sendBatch(messages)` sends multiple emails in one request.

### Message Shape

```ts
interface EmailMessage {
  to: string | string[]; // Max 50 recipients
  from: string | { email: string; name: string };
  subject: string;
  html?: string;
  text?: string;
  cc?: string | string[];
  bcc?: string | string[];
  replyTo?: string | { email: string; name: string };
  attachments?: Attachment[];
  headers?: Record<string, string>;
}
```

### Attachments

```ts
interface Attachment {
  content: string; // Base64 encoded
  filename: string;
  type: string;
  disposition: "attachment" | "inline";
  contentId?: string;
}
```

Use `disposition: "inline"` plus `contentId` for images referenced by `cid:` in HTML. Keep total content size within the documented limits.

### Response Semantics

- `send()` returns `{ success: true, messageId }` when Cloudflare accepts the email.
- `send()` throws `EmailSendError` for single-send failures.
- `sendBatch()` returns a result array containing success or error objects per email.
- `sendBatch()` still throws for binding-level failures such as an oversized batch.

## REST API

### Endpoint

```text
POST https://api.cloudflare.com/client/v4/accounts/{account_id}/email-service/send
```

### Authentication

```text
Authorization: Bearer <API_TOKEN>
Content-Type: application/json
```

Use the same payload shape as the Worker API. The main difference is transport and authentication.

### Minimal Example

```bash
curl "https://api.cloudflare.com/client/v4/accounts/${ACCOUNT_ID}/email-service/send" \
  --header "Authorization: Bearer ${API_TOKEN}" \
  --header "Content-Type: application/json" \
  --data '{
    "to": "user@example.com",
    "from": "welcome@yourdomain.com",
    "subject": "Welcome",
    "html": "<h1>Welcome</h1>",
    "text": "Welcome"
  }'
```

## Limits and Practical Constraints

- Combined `to` recipients are limited to 50.
- `E_CONTENT_TOO_LARGE` indicates total content above 25 MB.
- Custom header count and size are limited. The bundled docs call out:
  - max 20 whitelisted non-`X-` custom headers
  - max 16 KB total headers payload
  - max 2,048 bytes per header value
  - max 100 bytes per header name
- Do not use custom headers for fields that already have first-class API properties.

## Error Codes Worth Handling Explicitly

| Code | Meaning | Typical response |
| --- | --- | --- |
| `E_VALIDATION_ERROR` | Invalid payload | Fix request construction |
| `E_FIELD_MISSING` | Required field missing | Fix request construction |
| `E_TOO_MANY_RECIPIENTS` | Recipient limit exceeded | Split send or trim recipients |
| `E_SENDER_NOT_VERIFIED` | Sender not verified | Fix domain onboarding or sender identity |
| `E_SENDER_DOMAIN_NOT_AVAILABLE` | Domain not available for sending | Fix service/domain setup |
| `E_RATE_LIMIT_EXCEEDED` | Hourly rate limit reached | Retry later with backoff |
| `E_DAILY_LIMIT_EXCEEDED` | Daily quota reached | Delay until quota resets |
| `E_MONTHLY_LIMIT_EXCEEDED` | Monthly quota reached | Delay or upgrade service plan |
| `E_DELIVERY_FAILED` | Delivery attempt failed | Treat as delivery failure, not payload failure |
| `E_BINDING_UNAVAILABLE` | Worker binding missing | Fix Wrangler config or environment |
| `E_HEADER_NOT_ALLOWED` | Header not allowed | Remove or replace with supported API field |
| `E_HEADER_USE_API_FIELD` | Header must be set via API field | Move value into `from`, `replyTo`, etc. |

## Inbound Email Routing

The bundled overview docs show Email Routing delivering incoming mail to a Worker's `email()` handler. Within that handler you can inspect the message, forward it, or send an automated reply through the same `env.EMAIL` binding.

Treat routing as a separate concern from outbound sending:

- outbound sending uses `env.EMAIL.send()` or the REST endpoint
- inbound handling uses the Worker `email()` handler plus routing rules

## Legacy `EmailMessage`

Cloudflare still supports the older raw MIME `EmailMessage` flow:

```ts
import { EmailMessage } from "cloudflare:email";

await env.EMAIL.send(new EmailMessage(from, to, rawMimeMessage));
```

Use this only when an existing pipeline already generates MIME. Prefer the structured JSON-like message shape for new integrations.
