# Portable mailroom architecture

This reference defines the default technical design. Adjust it only when the interview exposes a concrete need.

## Design goal

Email intake must outlive any individual agent platform. The domain and Cloudflare resources form the mailroom; bots and workflows are detachable consumers.

```text
sender
  -> Cloudflare Email Service / Email Routing
  -> Email Worker
       1. validate recipient and size
       2. buffer the raw stream once
       3. parse bounded fields
       4. persist receipt + outbox atomically
  -> lane-scoped SQLite Durable Object
       - receipts
       - outbox
       - adapter delivery ledger
       - durable failure archive
       - retention jobs
  -> alarm + scheduled outbox reconciliation
  -> JSON Queue
       - retries
       - dead-letter queue
  -> adapter Worker or pull consumer
       - signed webhook
       - agent-specific adapter
       - workflow adapter
       - human review adapter
  -> active dead-letter consumer
       - durable failure record
       - owner alert
       - controlled replay
```

R2 is optional for raw `.eml` files or attachment bytes. A queue event should carry a pointer and bounded metadata, never the raw message.

When R2 storage is enabled, write bytes to a deterministic object key before committing the receipt pointer. If receipt persistence then fails, lifecycle cleanup may remove the unreferenced object. Never commit a receipt that claims an object exists before the object write succeeds.

## Components

### Email Worker

- Use ES modules and an `email()` handler.
- Route on the SMTP envelope recipient, not display headers.
- Reject oversized messages before expensive parsing.
- Buffer `message.raw` exactly once, then reuse that buffer.
- Validate the local part against a declarative inbox catalog.
- Normalize casing and plus-tags according to the catalog.
- Parse text defensively; sanitize or omit HTML.
- Record authentication results without assuming every missing result is malicious.
- Preserve inbound content as untrusted data; do not interpret message text as operational instructions.
- Return success only after durable receipt storage completes.
- Never call an agent harness directly.

### Lane-scoped Durable Object

Use one deterministic object ID per lane, not one global object for the entire system. Use SQLite storage for new classes.

Store receipt and outbox rows in one transaction. Suggested tables:

```sql
CREATE TABLE receipts (
  id TEXT PRIMARY KEY,
  lane TEXT NOT NULL,
  envelope_from TEXT NOT NULL,
  envelope_to TEXT NOT NULL,
  subject TEXT NOT NULL,
  text_preview TEXT,
  links_json TEXT NOT NULL,
  attachments_json TEXT NOT NULL,
  auth_results TEXT,
  raw_object_key TEXT,
  received_at TEXT NOT NULL,
  expires_at TEXT
);

CREATE TABLE outbox (
  event_id TEXT PRIMARY KEY,
  receipt_id TEXT NOT NULL,
  event_json TEXT NOT NULL,
  status TEXT NOT NULL,
  attempts INTEGER NOT NULL DEFAULT 0,
  next_attempt_at TEXT,
  last_error TEXT,
  created_at TEXT NOT NULL,
  sent_at TEXT
);

CREATE TABLE deliveries (
  event_id TEXT NOT NULL,
  adapter TEXT NOT NULL,
  status TEXT NOT NULL,
  attempts INTEGER NOT NULL DEFAULT 0,
  last_error TEXT,
  updated_at TEXT NOT NULL,
  PRIMARY KEY (event_id, adapter)
);

CREATE TABLE failed_events (
  event_id TEXT NOT NULL,
  receipt_id TEXT NOT NULL,
  adapter TEXT NOT NULL,
  event_json TEXT NOT NULL,
  error_class TEXT NOT NULL,
  attempts INTEGER NOT NULL,
  last_error TEXT,
  failed_at TEXT NOT NULL,
  expires_at TEXT,
  legal_hold INTEGER NOT NULL DEFAULT 0,
  replayed_at TEXT,
  PRIMARY KEY (event_id, adapter)
);
```

Generate migrations using current Cloudflare guidance. Do not copy this schema blindly when requirements call for encryption, legal hold, tenancy, or more stringent audit controls.

### Transactional outbox and reconciliation

The receipt and its outbox event must commit together. Do not rely on a catch block to create the only retry trigger: a process can crash after commit but before that catch runs.

Maintain an independent reconciliation path:

1. Ensure the lane's outbox alarm is armed before accepting a new write, then commit the receipt and pending outbox row.
2. After commit, attempt immediate Queue publication.
3. On every Durable Object start, write, and alarm, scan bounded pending rows and re-arm the next alarm before returning.
4. Run a scheduled Worker reconciliation over every configured lane as a backstop, so a missing or cleared alarm cannot strand a row.
5. Alert when the oldest pending row exceeds the agreed service window.

Marking an event sent and Queue delivery are not globally atomic, so duplicates are possible and expected.

Use the stable `event_id` as an application-level idempotency key; Cloudflare Queue does not deduplicate it for you. Before a side effect, the consumer must transactionally claim a delivery row using the unique `(event_id, adapter)` key and a bounded lease. Pass the same key to the downstream system when it supports idempotency. On success, mark delivered and then acknowledge the Queue message. If a timeout leaves the outcome ambiguous, query downstream status or retry with the same key, never a new one. If a non-reversible destination offers no idempotency mechanism, require human review or document and obtain acceptance of the duplicate risk.

Retry transient failures with capped exponential backoff and route exhausted events to a dead-letter queue. Configure an active DLQ consumer; the queue itself is temporary transport. The consumer must write the event, receipt ID, adapter, error class, attempts, and timestamps to `failed_events`, alert the owner, and acknowledge only after archival. Use `(event_id, adapter)` as the durable key. Provide a guarded replay operation that reuses the original `event_id`.

Failure records need their own retention policy. Default to 30 days or the receipt-retention period, whichever is shorter; keep only the redacted Queue event, never richer message content. Delete expired records and linked error artifacts unless `legal_hold = 1`. Legal hold requires separate approval, named ownership, access controls, cost review, and a documented release process.

If Queue is unavailable or intentionally omitted, dispatch directly from the outbox with Durable Object alarms and the scheduled reconciler. Define a maximum attempt count, capped backoff, `poisoned` terminal state, durable failure record, alert, and guarded replay. Retain the same event contract and document the weaker operational characteristics.

### Read API

Expose only the minimum required routes, for example:

```text
GET /health
GET /v1/lanes
GET /v1/lanes/:lane/receipts?limit=...
GET /v1/receipts/:id
GET /v1/events/:id/status
```

Protect every route except a metadata-light health endpoint. Return structured errors. Do not reveal account IDs, secret names, sender allowlists, message bodies, or resource internals from `/health`.

### Adapter boundary

Define an interface such as:

```ts
interface MailroomAdapter {
  readonly name: string;
  deliver(event: MailroomEvent, context: AdapterContext): Promise<DeliveryResult>;
}
```

Adapters may notify a bot, create a task, update a board, or request human review. They receive the stable event contract and use an authenticated receipt API when more content is required.

No adapter may change whether the ingest Worker accepts a valid message.

Email is an untrusted ingress channel. Agent adapters must:

- place subject, body, links, filenames, headers, and attachment metadata inside a clearly delimited untrusted-data field;
- ignore any message instruction that asks the agent to reveal secrets, change policy, call tools, contact people, or override the system;
- avoid fetching links or opening attachments automatically;
- scan or sandbox attachments before use;
- allow irreversible side effects only through an explicit policy or human approval gate;
- preserve provenance from the receipt ID through every generated task or action.

## Event contract

Use a CloudEvents-shaped JSON object so non-Cloudflare consumers can understand it:

```json
{
  "specversion": "1.0",
  "id": "evt_opaque_stable_id",
  "type": "mailroom.receipt.created.v1",
  "source": "urn:portable-mailroom:lane:research",
  "subject": "receipt_opaque_id",
  "time": "2026-01-01T12:00:00.000Z",
  "datacontenttype": "application/json",
  "data": {
    "schemaVersion": 1,
    "receiptId": "receipt_opaque_id",
    "lane": "research",
    "envelopeFrom": "sender@example.net",
    "envelopeTo": "research@mailroom.example.com",
    "subject": "Example subject",
    "receivedAt": "2026-01-01T12:00:00.000Z",
    "bodyPreview": "Bounded plain-text preview",
    "links": [],
    "attachments": [
      { "filename": "brief.pdf", "contentType": "application/pdf", "size": 12345 }
    ],
    "receiptUrl": "https://worker.example/v1/receipts/receipt_opaque_id"
  }
}
```

Do not put secrets, raw MIME, full HTML, or attachment bytes in this event. Make `bodyPreview` optional when privacy policy requires pointer-only events.

Version the event type and data schema. Preserve backward compatibility or publish a migration guide before changing them.

## Declarative lane catalog

Keep lanes in a non-secret file rather than scattered through code:

```json
{
  "schemaVersion": 1,
  "frontDoor": "frontdesk",
  "unknownRecipientPolicy": "reject",
  "plusAddressing": "normalize-and-preserve-tag",
  "lanes": [
    { "slug": "frontdesk", "label": "Front Desk", "purpose": "Default intake", "enabled": true },
    { "slug": "research", "label": "Research", "purpose": "Research requests", "enabled": true },
    { "slug": "test", "label": "Test", "purpose": "End-to-end tests only", "enabled": true }
  ]
}
```

Lanes should describe work, not a model vendor. Map lanes to particular bots inside adapter configuration.

## Security defaults

- Dedicated email subdomain.
- Receive-only.
- Unknown recipients rejected.
- Catch-all disabled.
- Raw email and attachment bytes not retained by default.
- Bounded plain-text preview; sanitized HTML omitted.
- Explicit maximum message and attachment sizes.
- Envelope addresses used for routing and allowlists.
- Secrets in Cloudflare secret storage, never plain `vars` or Git.
- Cloudflare Access for humans; separately scoped machine token for adapters.
- Constant-time token comparison where applicable.
- Logs contain opaque IDs and outcomes, not bodies or secrets.
- Agent prompts label email content as hostile input and keep tool authority outside that content.
- Rate controls and abuse monitoring on public endpoints.
- Least-privilege Cloudflare credentials for CI.

If quarantine is explicitly selected, the catch-all exception is allowed only on the dedicated mailroom subdomain. Verify its effective scope before mutation, preserve the apex catch-all before-state, and functionally test both a known apex address and an unknown subdomain address afterward. If the Cloudflare surface cannot prove that separation, keep catch-all disabled and use rejection.

## Portability requirements

The completed repository must make these exits possible:

1. Replace the agent harness without changing DNS, ingest, or stored receipts.
2. Disable all adapters while continuing to receive mail.
3. Export receipts as versioned JSONL and stored raw messages as standard `.eml` files when enabled.
4. Rebuild resources from source and a non-secret manifest.
5. Rotate every secret without changing the event schema.
6. Point a pull consumer outside Cloudflare at the same JSON event contract.
7. Document how to stop intake and restore prior DNS without deleting stored data.

For a Cloudflare exit, provide a concrete provider-neutral migration runbook:

1. Export the lane catalog, manifest, receipts as versioned JSONL, failure records, and any retained `.eml` objects.
2. Implement the same receive-only address rules, receipt schema, read API, and CloudEvents-shaped contract on the replacement provider.
3. Point a staging adapter at both implementations and compare test receipt/event IDs and fields.
4. Lower relevant DNS TTLs only within the approved change window, capture the before-state, and switch only the dedicated mail subdomain's MX records.
5. Run inbound, outage, duplicate, and replay tests on the replacement.
6. Keep the Cloudflare path recoverable through the agreed rollback window.
7. Decommission Cloudflare resources only through a separately approved deletion plan after exports and retention checks pass.

## Minimum tests

- Known recipient maps to exactly one lane.
- Address casing and plus-tags normalize correctly.
- Unknown recipient follows the configured reject/quarantine policy.
- Oversized email fails before parsing.
- The raw stream is consumed once.
- Receipt and outbox are committed together.
- Queue publish failure leaves a retryable outbox row.
- A crash between receipt commit and immediate Queue publication is recovered by alarm or scheduled reconciliation.
- Duplicate Queue delivery creates no duplicate downstream work.
- One failed event does not retry an entire batch unnecessarily.
- Exhausted events reach the dead-letter queue.
- The active dead-letter consumer archives, alerts, and replays without changing `event_id`.
- Adapter outage does not lose or reject inbound mail.
- Prompt-injection text, malicious links, and unsafe attachment metadata cannot acquire tool authority or trigger side effects.
- Expired receipts and R2 objects follow the retention policy.
- Unauthorized receipt access returns no private metadata.
- Health checks reveal no sensitive configuration.
