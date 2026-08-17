# CLI, MCP, and Project Setup

Manage Cloudflare Email Service from the command line and coding agents.

For full CLI reference, run `npx wrangler email --help`. For Dashboard setup, see the [getting started docs](https://developers.cloudflare.com/email-service/get-started/).

## Wrangler Email Commands

```
wrangler email routing
├── enable/disable   <domain>          # Toggle email routing
├── dns get          <domain>          # Show required DNS records
├── rules list/create/update/delete    # Manage routing rules
└── addresses list/create/delete       # Destination addresses (account-scoped)

wrangler email sending
├── enable/disable   <domain>          # Toggle email sending
├── dns get          <domain>          # Show sending DNS records (SPF, DKIM)
├── send             --from --to ...   # Send an email (builder flags)
└── send-raw         --from --to ...   # Send a raw MIME email
```

## Domain Setup

### Via Dashboard

1. Navigate to **Compute** > **Email Service** > **Email Sending** (or **Email Routing**)
2. Select **Onboard Domain** > choose domain > **Add records and onboard**

Onboarding auto-adds three records on the `cf-bounce` subdomain — an **MX** record (bounce routing), an **SPF** TXT record, and a **DKIM** TXT record (there is no CNAME) — plus a **DMARC** TXT record on `_dmarc.yourdomain.com`. DNS usually propagates within 5-15 minutes.

Before any domain is onboarded you can only send to [verified destination addresses](https://developers.cloudflare.com/email-service/configuration/domains/); once a domain is onboarded you can send to anyone.

### Via CLI

```bash
npx wrangler email sending enable yourdomain.com
npx wrangler email sending dns get yourdomain.com   # Verify records
```

### Sending Subdomains

You can also onboard a **subdomain** for sending (e.g. `mail.yourdomain.com`). Sending subdomains are zone-scoped and have their own REST endpoints:

| Endpoint | Purpose |
|----------|---------|
| `/zones/{zone_id}/email/sending/subdomains` | List, create, and delete sending subdomains |
| `/zones/{zone_id}/email/sending/subdomains/{subdomain}/dns` | Required DNS records |
| `/zones/{zone_id}/email/sending/subdomains/{subdomain}/dns/status` | DNS verification status |
| `/zones/{zone_id}/email/sending/subdomains/{subdomain}/reputation/complaints` | Complaint rate for the subdomain |

`npx wrangler email sending list` lists the onboarded sending domains and subdomains. See the [subdomains docs](https://developers.cloudflare.com/email-service/configuration/subdomains/).

## Local Development

Add `"remote": true` to send real emails during `wrangler dev`:

```jsonc
{ "send_email": [{ "name": "EMAIL", "remote": true }] }
```

```bash
npx wrangler dev
```

Emails are actually sent — use test addresses you control. `"remote": true` only affects `wrangler dev`, so it is fine to leave in place when you deploy.

## Cloudflare MCP Server

If you have the [Cloudflare MCP server](https://github.com/cloudflare/mcp) (`https://mcp.cloudflare.com/mcp`) configured, you can manage Email Service through its three tools: `search` (find API endpoints), `execute` (call them), and `docs` (search the Cloudflare documentation).

Use `search` to find email sending endpoints:

```javascript
// search tool — find all email sending API endpoints
async () => {
  const results = [];
  for (const [path, methods] of Object.entries(spec.paths)) {
    if (path.includes('email/sending')) {
      for (const [method, op] of Object.entries(methods)) {
        results.push({ method: method.toUpperCase(), path, summary: op.summary });
      }
    }
  }
  return results;
}
```

Then use `execute` to call them — for example, checking sending limits or sending an email:

```javascript
// execute tool — check sending quota
async () => {
  return cloudflare.request({
    method: "GET",
    path: `/accounts/${accountId}/email/sending/limits`
  });
}

// execute tool — send an email
async () => {
  return cloudflare.request({
    method: "POST",
    path: `/accounts/${accountId}/email/sending/send`,
    body: {
      to: "user@example.com",
      from: { address: "notifications@yourdomain.com", name: "My App" },
      subject: "Deployment Complete",
      html: "<h1>Deployed!</h1>",
      text: "Deployed!"
    }
  });
}
```

GraphQL analytics queries also work through `execute` — see [deliverability.md](deliverability.md#graphql-analytics-api) for query examples. Note that email analytics are **zone-level** datasets (`emailSendingAdaptiveGroups`, `emailSendingAdaptive`) queried under `viewer > zones`, and require the **Analytics Read** token permission.

## Sending from CLI / Agents

```bash
npx wrangler email sending send \
  --from "agent@yourdomain.com" \
  --to "developer@company.com" \
  --subject "Deployment Complete" \
  --text "Your Worker was deployed successfully."
```

Or via REST API:

```bash
curl "https://api.cloudflare.com/client/v4/accounts/${CLOUDFLARE_ACCOUNT_ID}/email/sending/send" \
  --header "Authorization: Bearer ${CLOUDFLARE_API_TOKEN}" \
  --header "Content-Type: application/json" \
  --data '{
    "to": "developer@company.com",
    "from": {"address": "agent@yourdomain.com", "name": "Build Agent"},
    "subject": "Deployment Complete",
    "text": "Your Worker was deployed successfully."
  }'
```
