# Optional clerk modes

The mailroom and the clerk are different systems.

The mailroom receives, validates, stores, and exposes receipts. A clerk interprets those receipts and creates downstream work. Turning off a clerk must never stop inbound mail or erase a receipt.

## Mode 1: no clerk

Use this for the smallest safe deployment.

- Store receipts and expose them through the protected read API.
- Do not publish adapter events when no consumer exists.
- Let a human or later automation review pending receipts.
- Show the pending count and oldest pending receipt in private operations tooling.

This mode needs no Cursor automation, hosted agent, model API, or webhook. It does not classify mail, write files, open pull requests, or create tasks.

## Mode 2: Cloudflare-native clerk

Use this when the user wants an always-on clerk without an external agent harness.

Start with deterministic routing in a separate Worker:

- map the recipient lane directly;
- apply explicit sender, subject, and content rules;
- produce a normalized intake record;
- leave ambiguous messages for human review;
- keep all side effects idempotent by receipt/event ID.

If semantic classification is genuinely needed, Workers AI may classify into a fixed schema. Treat the model output as an untrusted suggestion, validate it against the lane catalog, and require human review for ambiguous or high-impact decisions.

Creating GitHub files or pull requests is a separate adapter capability. It requires a scoped GitHub App or other approved credential stored as a Cloudflare secret. Before enabling it:

- name the exact repository and allowed paths;
- grant the least privilege needed;
- refuse deletes and unrelated edits;
- derive a stable branch/file key from the receipt ID;
- check for an existing result before retrying;
- open a pull request only after a new file was written;
- never merge automatically unless separately approved.

This mode depends on Cloudflare and any chosen destination API, but not on Cursor, Codex, Claude, Grok, or another agent runtime.

## Mode 3: hosted-agent clerk

Use this when a hosted agent needs to interpret the receipt or work inside its own repository tools.

- Deliver only the signed, versioned event payload the adapter needs.
- Do not give the clerk a broad receipt-list token when the event already contains the required data.
- Restrict repository writes to a named path and pull-request workflow.
- Require the clerk to ignore malformed source/event values.
- Keep email sending, Email Routing, DNS, secrets, and unrelated repositories outside its authority.
- Stamp generated artifacts with the real product and environment when provenance is required.

The adapter must be replaceable. Its downtime causes delayed triage, not lost mail.

## Failure behavior

| Component off | What still works | What waits |
| --- | --- | --- |
| Hosted agent clerk | Inbound routing and receipt storage | Classification and downstream work |
| Cloudflare-native clerk | Inbound routing and receipt storage | Rule-based downstream work |
| Queue consumer | Inbound routing, receipt storage, and outbox | Adapter delivery until recovery |
| Ingest Worker or Email Routing | Nothing new can arrive | Sender retries or rejects according to SMTP behavior |

Monitor these states separately. A green clerk run is not proof that mail storage is healthy, and a stored receipt is not proof that downstream work was created.

## Choosing a mode

Recommend no clerk for the first end-to-end test. Add the Cloudflare-native clerk when deterministic rules cover the job. Choose a hosted-agent clerk only when its judgment or repository tooling adds real value.

The build brief must say what happens when the chosen clerk is off, how pending work is visible, and how another clerk can replay the same event without duplicating work.
