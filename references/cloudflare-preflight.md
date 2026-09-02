# Cloudflare preflight and safety gates

Use this checklist for discovery, planning, deployment, and audit. Prefer the Cloudflare plugin; use the authenticated dashboard or Wrangler only where the plugin lacks a needed control.

## Refresh current facts

Before recommending architecture, plan, price, limits, commands, or configuration, retrieve current official material for:

- Email Service / Email Routing setup and subdomains;
- Email Workers and the `email()` handler;
- Workers plans, limits, and pricing;
- SQLite-backed Durable Objects;
- Queues delivery guarantees, retention, dead-letter queues, and pricing;
- R2 pricing and lifecycle rules when storing bytes;
- Cloudflare Access when protecting human inspection;
- current Wrangler configuration schema and generated Worker types.

Start with official Cloudflare documentation:

- <https://developers.cloudflare.com/email-service/get-started/route-emails/>
- <https://developers.cloudflare.com/email-service/configuration/subdomains/>
- <https://developers.cloudflare.com/email-service/api/route-emails/email-handler/>
- <https://developers.cloudflare.com/durable-objects/>
- <https://developers.cloudflare.com/queues/>
- <https://developers.cloudflare.com/queues/reference/delivery-guarantees/>
- <https://developers.cloudflare.com/queues/configuration/dead-letter-queues/>
- <https://developers.cloudflare.com/email-service/configuration/email-routing-addresses/>
- <https://developers.cloudflare.com/r2/>
- <https://developers.cloudflare.com/cloudflare-one/>
- <https://developers.cloudflare.com/workers/wrangler/>

These links are starting points, not cached truth. Record the page dates or retrieval date in the build brief.

## Read-only account discovery

Inspect before asking the user:

- connected account identity and account ID;
- candidate zone names and zone IDs;
- authoritative nameserver status;
- observed Workers plan and billing status;
- existing Worker names and routes;
- existing Durable Object namespaces, Queues, dead-letter queues, R2 buckets, and Access applications;
- existing Worker schedules, dead-letter consumers, and alerting destinations;
- Email Routing status for the apex and any configured subdomains;
- current routing rules and catch-all behavior;
- DNS records relevant to email: MX, SPF, DKIM, DMARC, and verification records;
- resource names that would collide with the proposed project.

Do not expose account IDs or sensitive configuration in public artifacts. Use them only where Cloudflare configuration requires them.

If the plugin cannot read the plan, say it is unverified and use the authenticated billing/Workers overview. Never infer a paid plan merely because a paid-compatible resource exists.

## DNS safety gate

Treat the apex domain's MX records as a protected production system.

Before enabling Email Routing:

1. List current apex MX records and identify the likely mail provider.
2. List MX and TXT records on the proposed subdomain.
3. Confirm the proposed subdomain is not already used for mail sending, routing, support, newsletters, or another service.
4. Save the exact before-state and rollback records in the build brief.
5. Prefer Cloudflare's supported subdomain onboarding flow.

Stop if:

- enabling routing would replace or conflict with existing human-mail MX records;
- Cloudflare is not authoritative for the selected zone and the required supported setup has not been established;
- the candidate subdomain already has an unexplained mail role;
- the user selected the wrong Cloudflare account or zone;
- the DNS before-state cannot be captured.

Do not "fix" unrelated DNS.

## Plan and cost gate

Use live plan details and current official pricing. Estimate the smallest realistic volume using:

- inbound messages per day and peak burst;
- average stored receipt size;
- raw email and attachment retention;
- Durable Object requests, rows, and stored bytes;
- Queue operations, retries, dead-letter traffic, and retention needs;
- durable failure-archive storage and replay volume;
- scheduled reconciliation frequency and the pending-row alert window;
- Worker requests and CPU;
- R2 operations and storage;
- Access seats or other separately billed features, if applicable.

Distinguish verified price from estimate. State the assumptions. If the free plan fits a test but not the desired retention or failure window, explain that directly. Never change a plan, add a paid product, or accept a billing prompt without a separate explicit approval.

## Resource naming

Use stable, vendor-neutral names. Suggested pattern:

```text
<project>-ingest-<env>
<project>-events-<env>
<project>-events-dlq-<env>
<project>-attachments-<env>
MAILROOM_LANES
MAILROOM_EVENTS
MAILROOM_ATTACHMENTS
```

The DLQ name is not an archive guarantee. Configure an active consumer that copies failures into durable storage before acknowledging them, then alert and support replay.

Do not put a person's name, email address, bot vendor, secret, or account ID in a reusable template.

## Secret handling

Use Cloudflare secret storage for values such as:

- receipt API bearer token;
- webhook signing secret;
- adapter credentials;
- encryption key, if the design requires application-level encryption.

Use plain configuration only for non-secret values such as environment name, retention days, and public base URL.

Create `.dev.vars.example` with names and comments but no values. Ensure `.dev.vars`, local state, logs, and downloaded messages are ignored by Git.

Do not print secret values to verify them. Verify behavior instead.

## Deployment gate

Before mutation, the build brief must name every proposed action, including:

- files to create or edit;
- Workers, Durable Object migrations, Queues, DLQ, active DLQ consumer, scheduled reconciler, durable failure archive, R2, and Access resources;
- DNS and Email Routing records or rules;
- secrets to create by name only;
- test and production environments;
- any recurring cost;
- rollback steps.

After exact approval:

1. Build and test locally.
2. Validate current Wrangler configuration and types.
3. Run a dry deployment.
4. Create isolated test resources.
5. Deploy and test the `test` lane.
6. Interrupt immediate outbox publication and prove scheduled reconciliation recovers it.
7. Force an event through retry exhaustion and prove the DLQ consumer archives and alerts before acknowledgment.
8. If quarantine was selected, prove the catch-all applies only to the dedicated subdomain, send a known-address apex test and an unknown-address subdomain test, and compare the apex routing before-state. Do not activate quarantine if this proof is unavailable.
9. Inspect logs and stored state without exposing message content.
10. Enable production routing only after the test path passes.
11. Capture resulting non-secret resource IDs/names, reconciliation interval, pending-row alert window, and failure-record retention in the manifest.
12. Re-read DNS and routing rules to confirm no unrelated change.

## Rollback

Rollback must be possible without deleting stored receipts:

- disable or remove only the new routing rule;
- restore the captured DNS before-state if the deployment changed it;
- leave Worker resources paused or undeployed as appropriate;
- preserve receipts, outbox, and the durable failure archive until the user approves deletion; do not rely on queue retention;
- revoke adapter credentials and receipt-read tokens;
- document what remains billable after intake stops.

Deletion is a separate destructive action. Never include it implicitly in rollback.
