# One-question setup interview

Use this as a state machine, not a questionnaire. Maintain one setup record and ask only the next unresolved decision.

## Conversation contract

- Ask one question per turn and end the turn after it.
- Keep the question short enough to answer without technical knowledge.
- Explain only the fact and tradeoff needed for that decision.
- Recommend a default, but do not treat silence as consent.
- Accept a plain-language answer and translate it into technical configuration yourself.
- Record answers without personalizing the reusable source or committing private values.
- Skip decisions the user already answered, including answers volunteered early.
- Use live discovery instead of asking the user to look up account, plan, zone, DNS, or deployed-resource facts.
- When several blockers exist, resolve only the first in this order: account identity, DNS/mail safety, plan purchase, destructive action, deployment approval.

## Setup record

Keep these fields during the interview:

```yaml
mode: build | audit | migrate
cloudflare_account: unresolved
zone: unresolved
workers_plan_observed: unresolved
feature_fit_checked_at: unresolved
email_subdomain: unresolved
existing_apex_mail: unresolved
receive_policy: receive-only
front_door: frontdesk
lanes: []
unknown_recipient_policy: reject
sender_policy: approved-senders
body_policy: text-preview
attachment_policy: metadata-only
retention_days: 90
failure_retention_days: 30
raw_message_storage: false
delivery_guarantee: durable-at-least-once
outbox_reconcile_interval_minutes: 5
pending_alert_after_minutes: 15
read_access: access-or-bearer-token
adapters: []
repository: unresolved
environment: test-then-production
deployment_approved: false
```

Do not commit the live setup record until a repository is chosen. Never write secrets into it.

## Phase 0: understand the request

Infer `build`, `audit`, or `migrate` from the request. Ask only if it is genuinely ambiguous.

Default goal: receive and preserve messages first; route work second.

If outbound sending appears important, ask this early:

> I recommend starting receive-only so a setup mistake cannot send email in your name. Should this first version receive only?

Treat sending as a separate feature with separate threat modeling, deliverability checks, limits, and approval.

## Phase 1: inspect Cloudflare before asking

Use the connected Cloudflare surface to discover:

- authenticated account identity and available accounts;
- zones and whether Cloudflare is authoritative DNS;
- current Workers plan;
- Email Service / Email Routing status;
- existing Email Workers, Durable Objects, Queues, R2 buckets, Access apps, and likely name conflicts;
- apex and candidate-subdomain MX, SPF, DKIM, and DMARC records.

Ask only what discovery cannot decide.

If there are multiple accounts:

> I found more than one Cloudflare account. Which account should own the mailroom?

If there are multiple eligible zones:

> Which of these domains should own the mailroom?

If there is no active Cloudflare connection:

> Can you connect the Cloudflare account that owns the domain now?

Do not ask for an API token in chat.

## Phase 2: choose a safe address boundary

Recommend a dedicated subdomain, especially when the apex already handles human email.

Ask:

> I recommend `[suggested-subdomain].[zone]` so this cannot disturb your normal email. Do you want that subdomain?

Use `mailroom`, `agents`, or `bots` as suggestions, not assumptions. Check DNS availability before suggesting one.

If the user chooses the apex and existing MX records are present, stop. Explain which provider appears to handle current mail and ask one question about whether they want a subdomain instead. Never replace those MX records merely because the user said to use the domain.

## Phase 3: define the front door and lanes

Recommend one public front door plus explicit specialist lanes. This keeps the public address stable while internal roles evolve.

Ask for the front door first:

> What should the main intake address be called? I recommend `frontdesk@...`.

Then build lanes one at a time:

> What is the first specialist lane this mailroom should have?

After each answer, paraphrase the lane's purpose and ask whether to add one more lane. That is still one question. Use short, role-based slugs such as `research`, `creative`, `operations`, or `test`; do not encode a vendor or model name unless the user insists.

Always create a `test` lane that cannot trigger production work.

Then ask:

> I recommend rejecting unknown addresses so typos are visible. Should unknown recipients be rejected?

Default rejection uses explicit address rules with catch-all disabled. If the user chooses quarantine, explain that Cloudflare must deliberately route catch-all mail to the Worker. Prove from live configuration that catch-all can be scoped only to the selected dedicated subdomain; capture the apex catch-all before-state and never enable or alter it. If scope cannot be verified unambiguously, quarantine is not an available choice and unknown addresses must be rejected. Store quarantine mail in an isolated lane, do not notify production adapters, apply stricter size/rate limits, and include the spam/cost tradeoff in the build brief.

Normalize address casing and decide whether `lane+tag@...` maps to `lane`. Default: allow plus-tags and preserve the tag as metadata.

## Phase 4: decide who can send

Default: accept only approved senders while recording authentication results and applying size/rate controls. For a public intake address, allow internet mail only after the user chooses it, and route untrusted content to human review or a no-side-effect classification adapter first.

Ask:

> Should these addresses accept mail from anyone, or only approved senders?

If approved senders are chosen, collect one sender or domain per turn. Use SMTP envelope values for enforcement; display headers are untrusted.

## Phase 5: decide what survives

Resolve these fields in order, showing only the current field to the user: receipt retention, body retention, attachment retention, then raw-message retention. Never display the four prompts together.

- For receipt retention, recommend 90 days for a first version.
- For body retention, recommend a bounded plain-text preview rather than the full body.
- For attachments, recommend metadata-only rather than storing bytes.
- For the original raw email, recommend no storage unless legal, audit, or archival needs justify it.

Ask about only the current field, then stop. Explain privacy and cost in one sentence. Set explicit byte limits, MIME allowlists, retention, and deletion behavior for anything stored in R2.

Default durable failure records to 30 days or the receipt-retention period, whichever is shorter. They follow the same body-redaction policy as Queue events, not the richer receipt. A legal hold is a separate explicit decision with scope, cost, access, and release criteria.

## Phase 6: choose downstream consumers

Explain that the mailroom works without an agent connection. Then ask:

> Where should a new stored message be announced first?

Valid answers include manual review with no clerk, a Cloudflare-native rules clerk, a generic signed webhook, a Cloudflare pull consumer, an agent adapter, or a workflow tool. Configure each integration as a separate adapter. Never place vendor-specific SDK calls in ingest or receipt storage. Read [clerk-options.md](clerk-options.md) before recommending among these modes.

Any adapter that invokes an agent must wrap message content as untrusted quoted data, exclude raw HTML and attachment bytes by default, prevent email text from changing tools or policy, and require an explicit allowlist or human approval before irreversible external actions.

For every adapter, determine:

- endpoint or consumer type;
- minimum event fields;
- authentication method;
- timeout and retry behavior;
- idempotency behavior;
- whether it may fetch full content;
- what a permanent failure means.

Resolve these one question at a time only when the answer cannot be safely defaulted.

## Phase 7: choose access and ownership

Ask where the source should live:

> Which Git repository should own the mailroom source and its non-secret manifest?

Recommend an existing user-owned repository when appropriate. Otherwise create a small standalone repository only after explicit approval.

Ask how people should inspect receipts:

> Should receipt access use Cloudflare Access, a secret API token, or both?

Recommend Cloudflare Access for humans and a scoped bearer token for machine adapters. Never expose a public list endpoint.

## Phase 8: confirm the plan fit

Use current official Cloudflare sources and live account data to compare the design with the observed plan. Report verified facts, expected low-volume cost, quota risks, and what could not be read. Do not upgrade or purchase anything automatically.

If the current plan is insufficient, ask one question offering the smallest safe alternatives: reduce the design, use an outbox-only fallback, or approve a named plan change. Never hide a plan requirement inside deployment approval.

## Phase 9: action-time approval

Present the exact build brief required by the main skill. Then ask only:

> Should I apply this exact build brief now?

On approval, execute the listed work and verification. If the work reveals a new DNS conflict, plan purchase, destructive change, or broader scope, stop and ask one new question before that action.

## Completion record

At completion, replace unresolved fields with observed values and add:

```yaml
deployed_at: ISO-8601 timestamp
worker_name: non-secret name
queue_name: non-secret name or null
dead_letter_queue_name: non-secret name or null
r2_bucket_name: non-secret name or null
durable_object_namespace: non-secret binding name
schema_version: 1
event_contract_version: 1
failure_retention_days: configured value
outbox_reconcile_interval_minutes: configured value
pending_alert_after_minutes: configured value
source_commit: git commit SHA
last_verified_test_receipt_id: opaque non-secret ID
```

Write this non-secret record to `mailroom-manifest.json` and document where secrets are managed without exposing their values.
