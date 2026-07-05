---
name: cloudflare-email-service
description: Use when building or migrating email sending with Cloudflare Email Service, especially when choosing between Workers bindings and the REST API, wiring Wrangler `send_email` bindings, handling attachments or batches, or mapping Cloudflare-specific limits and error codes.
---

# Cloudflare Email Service

## Overview

Use this skill to choose the right Cloudflare Email Service API surface, shape valid payloads, and avoid common integration mistakes. Prefer the native Worker binding whenever code already runs inside Cloudflare Workers.

## Quick Reference

| Situation | Use | Why |
| --- | --- | --- |
| Code runs inside a Cloudflare Worker | `send_email` binding plus `env.EMAIL.send()` or `env.EMAIL.sendBatch()` | Native path, no manual bearer token handling |
| Code runs outside Workers | REST API | Works from any backend with account ID and API token |
| Need inbound handling | Worker `email()` handler | Email Routing delivers incoming messages to Workers |
| Need hosted templates | App-side rendering | The bundled docs snapshot says templates are not available |
| Need raw MIME control | Legacy `EmailMessage` API | Keep this for existing MIME pipelines only |

## Workflow

1. Confirm prerequisites before touching code.
   The bundled docs snapshot says Email Service is on the Workers Paid plan, requires Cloudflare DNS for the sender domain, and requires verified senders or onboarded domains.
2. Choose the transport based on runtime, not convenience.
   Inside Workers, use the binding.
   Outside Workers, use the REST endpoint.
   Do not call the REST API from a Worker unless a hard runtime constraint prevents bindings.
3. Build the payload with API fields instead of controlled headers.
   Set `to`, `from`, and `subject` explicitly.
   Add `html`, `text`, `replyTo`, `cc`, `bcc`, `attachments`, and `headers` only when needed.
   Keep total recipients within the documented 50-recipient limit.
4. Treat attachments and batches as structured API features.
   Base64-encode every attachment body.
   Use `disposition: "inline"` plus `contentId` for CID images.
   Expect `sendBatch()` to produce mixed per-message results.
5. Normalize Cloudflare errors at the boundary.
   Map validation and sender-verification problems to non-retryable failures.
   Map rate-limit or internal-service failures to retry or backoff behavior.
   Distinguish configuration errors from delivery failures.

## Default Pattern

Wrap Cloudflare Email Service behind one app-level helper so logging, retries, and provider migration stay in one place:

```ts
interface Env {
  EMAIL: EmailBinding;
}

export async function sendWelcomeEmail(env: Env, recipient: string) {
  return env.EMAIL.send({
    to: recipient,
    from: { email: "welcome@yourdomain.com", name: "Acme" },
    subject: "Welcome to Acme",
    html: "<h1>Welcome</h1><p>Thanks for signing up.</p>",
    text: "Welcome. Thanks for signing up.",
  });
}
```

Prefer the structured `send()` message shape for new work. Only fall back to the legacy `EmailMessage` MIME API when integrating with an existing raw-email pipeline.

## Common Mistakes

- Using the REST API from a Worker even though a binding is available.
- Sending from an address whose domain is not verified or onboarded.
- Setting `From`, `Reply-To`, or similar controlled values inside `headers` instead of dedicated API fields.
- Forgetting that `attachments[].content` must be base64.
- Treating `sendBatch()` as all-or-nothing instead of checking each result.
- Assuming Cloudflare hosts email templates for you.
- Ignoring the docs snapshot when plan limits, quotas, or beta access matter.

## Red Flags

- "I will just fetch the REST endpoint from this Worker."
- "The sender address probably works if the payload validates."
- "I can stuff `From` into custom headers."
- "A successful batch call means every email was accepted."
- "Templates must exist somewhere in the dashboard."

If any of those thoughts appear, stop and re-check the API surface and the reference file.

## Resources

Read `api-reference.md` for binding configuration, REST auth, payload fields, documented limits, and Cloudflare error codes.
