# Portable Agent Mailroom

Give AI agents inboxes on a domain you own without tying the mail to one model, bot, or agent platform.

This repository contains a reusable `SKILL.md` that guides an agent through the setup one question at a time. It checks the user's live Cloudflare account and current plan before proposing changes.

## What it builds

```text
email on your domain
  -> Cloudflare Email Routing
  -> receive-only Email Worker
  -> durable receipt and outbox
  -> retryable queue
  -> replaceable adapter
  -> any agent or human workflow
```

The receipt is stored before an agent is notified. If the agent, webhook, or platform is down, the mail still lands.

## Does this require a cloud agent?

No. The mailroom works without Cursor or another continuously running agent. It will still receive, validate, and store messages.

The guided setup proves that no-clerk path first. It does not make a hosted agent part of the foundation.

A clerk is needed only for last-mile work such as interpreting a message, writing an intake file, opening a pull request, or creating a task. The skill offers three modes:

1. No clerk yet. Review stored receipts manually.
2. A Cloudflare-native rules clerk, with optional Workers AI classification.
3. A replaceable hosted-agent clerk such as Cursor, Codex, Claude, or Grok.

See [`references/clerk-options.md`](references/clerk-options.md) for the exact boundary and failure behavior.

## Start with your agent

Open [`COPY_TO_AGENT.md`](COPY_TO_AGENT.md), copy the instruction block, and paste it into an agent with access to your Cloudflare account and a working folder.

If the agent supports reusable skills, it should install this repository as `portable-agent-mailroom`. If it does not, it can follow the files directly as operating instructions.

## Defaults

- Receive-only. No replies or automatic sending.
- A dedicated email subdomain, protecting existing human mail.
- Approved senders unless the user chooses public intake.
- Explicit inbox lanes and no silent catch-all.
- SQLite-backed Durable Objects for receipts and the outbox.
- Cloudflare Queues for retryable delivery when the live plan supports it.
- A durable failure archive instead of relying on temporary queue retention.
- Agent-specific integrations behind replaceable adapters.
- Email bodies, links, filenames, and attachments treated as untrusted input.

## Files

- [`SKILL.md`](SKILL.md): main operating instructions
- [`references/interview.md`](references/interview.md): one-question setup flow
- [`references/architecture.md`](references/architecture.md): portable system design and event contract
- [`references/cloudflare-preflight.md`](references/cloudflare-preflight.md): account, plan, DNS, deployment, and rollback checks
- [`references/clerk-options.md`](references/clerk-options.md): no-clerk, Cloudflare-native, and hosted-agent modes

## What this is not

This is not Gmail, a hosted mailbox product, or a bulk-email tool. It does not require Grok, Codex, Claude, Cursor, or the Cloudflare Agents SDK. Those can be added later as adapters.

The setup agent must verify current Cloudflare documentation, plan limits, pricing, and configuration before deployment. Nothing in this repository should be treated as a promise that a particular Cloudflare feature is included with a particular plan.

## License

MIT. See [`LICENSE`](LICENSE).
