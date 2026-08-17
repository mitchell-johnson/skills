# Email Deliverability & Best Practices

For full details, see the [deliverability docs](https://developers.cloudflare.com/email-service/concepts/deliverability/) and [email authentication docs](https://developers.cloudflare.com/email-service/concepts/email-authentication/). All the monitoring endpoints below can be called via the [REST API](rest-api.md), [Wrangler CLI, or the Cloudflare MCP server](cli-and-mcp.md).

## What Cloudflare Handles

When you onboard a domain, Cloudflare auto-configures:

- **SPF** — TXT records authorizing Cloudflare's sending infrastructure
- **DKIM** — Records for cryptographic signing of outbound emails
- **IP reputation** — Managed sending infrastructure optimized for deliverability
- **Soft bounce retries** — Automatic exponential backoff for temporary failures
- **Suppression lists** — Hard-bounced addresses automatically blocked
- **Feedback loops** — ISP complaint signals processed and acted on
- **DMARC** — A DMARC TXT record is added automatically during domain onboarding

Cloudflare recommends starting DMARC at `p=none` (monitor only) — e.g. `v=DMARC1; p=none; rua=mailto:dmarc-reports@yourdomain.com` — then escalating to `p=quarantine` and finally `p=reject` once you have reviewed the aggregate reports and confirmed your legitimate mail passes.

## Bounce Handling

**Hard bounces** — permanent failures (address doesn't exist, domain doesn't exist). Never retried. Address auto-added to suppression list. Sending to suppressed address returns `E_RECIPIENT_SUPPRESSED`.

**Soft bounces** — temporary failures (mailbox full, server down, greylisting). Cloudflare auto-retries with exponential backoff.

## Suppression Lists

**Account list** (your account) — spam complaints from recipients. Cloudflare integrates with Postmasters to auto-suppress. You can manually add/remove addresses in the Dashboard.

See the [suppressions docs](https://developers.cloudflare.com/email-service/concepts/suppressions/) for details.

## Your Responsibilities

### Content
- Include both HTML and plain text versions
- Use a recognizable sender name: `{ email: "noreply@app.com", name: "My App" }`
- Write honest subject lines — avoid ALL CAPS, excessive punctuation
- Include `List-Unsubscribe` headers for recurring emails
- Use full URLs from your domain — avoid URL shorteners

### List Quality
- Validate email addresses before sending
- Implement double opt-in for subscriptions
- Honor unsubscribe requests promptly

### Transactional Only
Email Service is for **transactional email** (triggered by user actions: signups, password resets, order confirmations). Marketing/bulk campaigns are not permitted — use a dedicated marketing platform.

## Monitoring Deliverability

### Event Subscriptions

Since July 2026, Email Service can publish email lifecycle events to a [Cloudflare Queue](https://developers.cloudflare.com/queues/) via [event subscriptions](https://developers.cloudflare.com/email-service/platform/event-subscriptions/) — the modern alternative to polling analytics.

Events cover the full delivery lifecycle — for example `cf.email.sending.message.delivered`, plus deferred, bounce, and complaint events — and carry SMTP status detail so you can act on the exact failure reason.

Prefer this for anything reactive (suppressing addresses, alerting, updating your own delivery dashboard); use GraphQL analytics for historical aggregates.

### Dashboard

Per-domain and account-wide analytics are available in the Cloudflare dashboard:

1. Log in to the [Cloudflare dashboard](https://dash.cloudflare.com) and select your account.
2. Go to **Compute** > **Email Service**.
3. Select a domain or view account-wide metrics.
4. Select the **Analytics** tab.

### Send Response

Every send (REST API or Workers binding) returns immediate delivery feedback. Check the response to track per-send outcomes:

```json
{
  "result": {
    "delivered": ["user@example.com"],
    "permanent_bounces": ["bad@nonexistent.com"],
    "queued": ["slow@recipient.com"]
  }
}
```

Log these to build your own delivery rate metrics.

### Message Status Lookup

Look up a single sent message by the `messageId` returned from the binding or REST send:

```bash
curl "https://api.cloudflare.com/client/v4/accounts/{account_id}/email/sending/messages/{message_id}" \
  --header "Authorization: Bearer <API_TOKEN>"
```

### Sending Limits

Check your account's daily sending quota:

```bash
curl "https://api.cloudflare.com/client/v4/accounts/{account_id}/email/sending/limits" \
  --header "Authorization: Bearer <API_TOKEN>"
```

Returns:

```json
{
  "result": {
    "quota": { "value": 5000, "unit": "day" }
  }
}
```

Workers Paid includes 3,000 emails/month, then $0.35 per 1,000. Sends to verified destination addresses are free on all plans and don't count toward quota, and rejected or suppressed sends don't consume quota either. Email Routing is free and unlimited. See the [pricing summary in SKILL.md](../SKILL.md#pricing--limits) and the [pricing docs](https://developers.cloudflare.com/email-service/platform/pricing/).

### Suppression List

Addresses that hard-bounced or received spam complaints are auto-suppressed. You can query and manage suppressions via the API.

**List suppressions (account-wide):**

```bash
curl "https://api.cloudflare.com/client/v4/accounts/{account_id}/email/sending/suppression?page=1&per_page=100&order=created_at&direction=desc" \
  --header "Authorization: Bearer <API_TOKEN>"
```

Returns:

```json
{
  "page": 1,
  "per_page": 100,
  "total": 2,
  "result": [
    {
      "id": "396a5436-d4b0-42a6-b3fc-48e8fa522321",
      "email": "bounced@example.com",
      "reason": "hard_bounce",
      "created_at": "2026-03-15T10:00:00Z",
      "expires_at": null
    }
  ]
}
```

Query params: `page`, `per_page` (max 1000), `order` (`email` | `expires_at` | `created_at`), `direction` (`asc` | `desc`).

**Manually suppress an address:**

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/{account_id}/email/sending/suppression" \
  --header "Authorization: Bearer <API_TOKEN>" \
  --header "Content-Type: application/json" \
  --data '{ "email": "user@example.com", "expires_at": "2026-06-01T00:00:00Z" }'
```

`expires_at` is optional — omit for permanent suppression.

**Remove a suppression:**

```bash
curl -X DELETE "https://api.cloudflare.com/client/v4/accounts/{account_id}/email/sending/suppression/{suppression_id}" \
  --header "Authorization: Bearer <API_TOKEN>"
```

Zone-level suppressions are also available at `/zones/{zone_id}/email/sending/suppression` with the same interface.

**Newer plural API:** `/accounts/{account_id}/email/sending/suppressions` sits alongside the singular endpoints above and adds:

- **Cursor pagination** instead of `page`/`per_page`
- Filters: `email`, `search`, `reason`
- `note` and `read_only` fields on each suppression
- `PATCH` support for updating an existing suppression
- `POST /suppressions/bulk` for importing many addresses at once

Prefer the plural endpoints for new integrations, especially when paging through large lists.

### GraphQL Analytics API

Email Service exposes two zone-level datasets via the [GraphQL Analytics API](https://developers.cloudflare.com/analytics/graphql-api/). You can explore the schema interactively at [graphql.cloudflare.com/explorer](https://graphql.cloudflare.com/explorer). Metrics are retained for 31 days.

| Dataset | Description |
|---------|-------------|
| `emailSendingAdaptiveGroups` | Aggregated counts grouped by dimensions (status, date, domain, auth results, etc.) |
| `emailSendingAdaptive` | Individual email events with full detail (from, to, subject, messageId, errors, etc.) |

These are **zone-level** datasets — query under `viewer > zones`, not `accounts`.

**Aggregated dimensions** (`emailSendingAdaptiveGroups`):

| Dimension | Type | Description |
|-----------|------|-------------|
| `date` | Date | Day-level grouping |
| `datetime` | Time | Exact timestamp (also: `datetimeMinute`, `datetimeFiveMinutes`, `datetimeFifteenMinutes`, `datetimeHour`) |
| `status` | string | Delivery status |
| `eventType` | string | Event type |
| `sendingDomain` | string | The sending domain |
| `envelopeTo` | string | Recipient address |
| `errorCause` | string | Error cause for failed sends |
| `arc`, `dkim`, `dmarc`, `spf` | string | Email authentication results |
| `isSpam`, `isNDR`, `isLastEvent` | uint8 | Boolean flags |
| `spamScore`, `spamThreshold` | uint32 | Spam scoring |

**Individual event fields** (`emailSendingAdaptive`) additionally include: `from`, `to`, `subject`, `messageId`, `sessionId`, `errorDetail`.

**Email counts by status and date:**

```graphql
query EmailSendingByStatus(
  $zoneTag: string!
  $start: Date!
  $end: Date!
) {
  viewer {
    zones(filter: { zoneTag: $zoneTag }) {
      emailSendingAdaptiveGroups(
        filter: { date_geq: $start, date_leq: $end }
        limit: 10000
        orderBy: [date_DESC]
      ) {
        count
        dimensions {
          date
          status
        }
      }
    }
  }
}
```

**Filter by status (e.g. only failures):**

```graphql
query EmailFailures(
  $zoneTag: string!
  $start: Date!
  $end: Date!
) {
  viewer {
    zones(filter: { zoneTag: $zoneTag }) {
      emailSendingAdaptiveGroups(
        filter: { date_geq: $start, date_leq: $end, status: "deliveryFailed" }
        limit: 10000
        orderBy: [date_DESC]
      ) {
        count
        dimensions {
          date
          errorCause
          sendingDomain
        }
      }
    }
  }
}
```

**Individual email events (troubleshooting):**

```graphql
query RecentEmailEvents(
  $zoneTag: string!
  $start: Time!
  $end: Time!
) {
  viewer {
    zones(filter: { zoneTag: $zoneTag }) {
      emailSendingAdaptive(
        filter: { datetime_geq: $start, datetime_leq: $end }
        limit: 50
        orderBy: [datetime_DESC]
      ) {
        datetime
        from
        to
        subject
        status
        eventType
        sendingDomain
        messageId
        errorCause
        errorDetail
        dkim
        dmarc
        spf
        isSpam
      }
    }
  }
}
```

Note: `emailSendingAdaptive` filters use `datetime_geq`/`datetime_leq` (Time type, e.g. `"2026-04-01T00:00:00Z"`), while `emailSendingAdaptiveGroups` uses `date_geq`/`date_leq` (Date type, e.g. `"2026-04-01"`).

**curl example:**

```bash
curl "https://api.cloudflare.com/client/v4/graphql" \
  --header "Authorization: Bearer <API_TOKEN>" \
  --header "Content-Type: application/json" \
  --data '{
    "query": "query($zoneTag:string!,$start:Date!,$end:Date!){viewer{zones(filter:{zoneTag:$zoneTag}){emailSendingAdaptiveGroups(filter:{date_geq:$start,date_leq:$end},limit:10000,orderBy:[date_DESC]){count,dimensions{date,status}}}}}",
    "variables": {
      "zoneTag": "<ZONE_ID>",
      "start": "2026-03-15",
      "end": "2026-04-15"
    }
  }'
```

## Metrics to Watch

| Metric | Target | If Out of Range |
|--------|--------|-----------------|
| Delivery rate | > 95% | Check for invalid addresses; verify DNS records |
| Hard bounce rate | < 2% | Clean your email list |
| Complaint rate | < 0.1% | Make unsubscribe easier; stop unwanted emails |
