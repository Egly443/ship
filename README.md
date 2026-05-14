# Ship Protocol

> **One command. One guided session. A revenue-ready, globally distributed stack.**

```
/ship
```

---

## The problem

Starting a company in 2026 still requires a founder to open eight browser tabs, create accounts on four platforms, copy secrets from one dashboard and paste them into another, pray nothing leaks in the terminal history, and repeat the process for staging, preview, and production.

It looks like this:

1. Create a Vercel account. Find the project settings. Generate a deploy token.
2. Create a Supabase project. Wait for provisioning. Copy the anon key. Copy the service role key. Copy the database URL. Paste each one into Vercel's environment variables panel. One at a time. For each environment.
3. Create a Stripe account. Complete KYC. Create products and prices. Copy the publishable key. Copy the secret key. Navigate to the webhook settings. Register an endpoint. Copy the webhook secret. Paste each one into Vercel. Again, for each environment.
4. Buy a domain. Point nameservers at Cloudflare. Create DNS records pointing at Vercel. Wait. Verify TLS.
5. Realise you forgot to add `NEXT_PUBLIC_` to the Supabase URL. Start over.

This is not a skill problem. It is a tooling problem. The bottleneck for startups is not intelligence — it is trusted execution across fragmented infrastructure providers.

The cost is not just time. Every secret that passes through a human's clipboard is a credential at risk. Every manually-typed environment variable is a potential typo in production. Every dashboard hop is a context switch that breaks momentum at exactly the wrong moment.

---

## The idea

What if a founder could type `/ship`, answer a handful of questions, and walk away while an AI agent provisioned, wired, and verified their entire stack?

Not a wrapper around Terraform. Not a hosted service that owns your billing. Not a proprietary lock-in play.

An **open protocol** that lets any AI agent — Claude, Codex, Cursor, Copilot, anything — orchestrate provisioning across any compliant provider, using each provider's own identity system, billing infrastructure, and compliance controls.

The agent orchestrates. The providers execute. The user retains sovereignty over every account, every invoice, and every secret.

This proposal is designed to be submitted to Anthropic's [Model Context Protocol](https://modelcontextprotocol.io) specification as a formal MCP extension. MCP is the open protocol Anthropic publishes and maintains for agent-tool communication — it already defines how agents talk to services. Ship extends it with a provisioning and secret-exchange contract that infrastructure providers can implement natively. The coordination path is: one spec owner (Anthropic/MCP), three reference implementors (Stripe, Vercel, Supabase), and a test suite that certifies compliance. No separate standards body required.

---

## How it works

### The session

```
> /ship

? Project name: awesomeidea
? Services needed: database (Supabase), hosting (Vercel), payments (Stripe), DNS (Cloudflare)
? Region: eu-west-2
? Start on free tiers where available? Yes
? Domain: awesomeidea.ai

─── Compliance review ──────────────────────────────────────────────
  Stripe will perform KYC directly with you before live payments
  are enabled. The agent cannot bypass or proxy this step.

  Your Stripe secret key will never be read by this agent. It will
  be pushed directly to Vercel via a scoped ephemeral token.

  PCI-DSS SAQ-A compliance is your responsibility once live.

  [I understand and agree] ›
────────────────────────────────────────────────────────────────────

  ✓ Identity federated via OIDC
  ✓ Supabase project provisioned (us-east-1)
  ✓ Stripe account created — KYC pending with Stripe directly
  ✓ Secrets pushed: Supabase → Vercel (production, preview)
  ✓ Secrets pushed: Stripe → Vercel (production)
  ✓ DNS configured: awesomeidea.ai → Vercel
  ✓ Next.js project deployed
  ✓ Smoke tests passed

  Stack is live. No secrets were read by this agent.
  Audit trace written to execution.trace
```

All questions are asked upfront. The user does not babysit the run.

---

### The design

#### Federated identity, not consolidated billing

Every provider bills the user directly. Stripe invoices the founder. Vercel invoices the founder. Supabase invoices the founder. The AI vendor is not in the financial chain.

What the agent holds is a **Delegation Token** — a short-lived, scoped JWT issued by each provider's own authorization server after the user consents via PKCE OAuth2. The token expires in 15 minutes. It can only provision, not delete or upgrade. It is revoked at session end.

```
Founder → authenticates to → Ship Agent (OIDC)
Ship Agent → presents identity to → Vercel /mcp/authorize
Vercel → shows consent screen to → Founder (PKCE, not proxied)
Vercel → issues Delegation Token (TTL: 15m, scope: provision-only) → Ship Agent
Ship Agent → calls → Vercel /mcp/provision (with Delegation Token)
```

The agent never sees a master API key. It never touches billing configuration. It cannot upgrade a plan or transfer a domain without explicit human approval.

#### Secrets never transit the orchestrator

The most common credential leak pattern is: provision a service, copy the output secret, paste it into another service's config. Every step is a human touchpoint and an attack surface.

Ship eliminates this entirely with **provider-to-provider secret push**.

```
Agent: "Supabase, push DATABASE_URL and SUPABASE_SERVICE_ROLE_KEY 
        to Vercel project awesomeidea, environments [production, preview]."

Supabase → mTLS → Vercel /mcp/secret-receive

Agent sees: { "DATABASE_URL": "pushed", "SUPABASE_SERVICE_ROLE_KEY": "pushed" }
```

The agent instructs. It never reads. The secret value travels directly between providers over mutual TLS using a session-scoped inter-provider token.

#### Compliance is explicit, not abstracted

Ship does not try to hide the fact that Stripe requires KYC, or that DNS transfers are irreversible, or that PCI-DSS compliance is the founder's responsibility.

Every service in `ship.json` can declare compliance acknowledgements that the user must explicitly confirm before the provisioning step begins. These are not checkbox theatre — they are surfaced as first-class steps in the execution graph, before any API call is made.

Certain actions — domain purchase, plan upgrade, production deletion, payment routing changes — require an explicit `/approve` command regardless of what the manifest says. The agent cannot proceed without it.

#### Rollback is defined, not assumed

Partial failures happen. The protocol defines what to do.

- Resources provisioned before the failure point are destroyed (idempotent rollback via `/mcp/rollback`)
- Resources that cannot be auto-destroyed after compliance verification (Stripe accounts post-KYC) are flagged with exact manual steps
- Every state transition is snapshotted before mutation so the user has a recovery path

#### The dependency graph is explicit

```
identity_federation
    └─ billing_authorization
           └─ provider_account_linking
                  └─ compliance_acknowledgement        ← user confirms all compliance items
                         ├─ database_provisioning
                         └─ payment_provisioning
                                └─ secret_exchange     ← provider-to-provider push
                                       └─ dns_configuration
                                              └─ application_deployment
                                                     └─ smoke_tests
```

Parallelism where possible. Strict ordering where dependencies require it. The manifest is the source of truth.

---

## The manifest

`ship.json` is committed to the repo. It contains no secrets. It describes what to provision, for whom, and under what constraints. It is the input to `/ship` and the audit record of intent.

A minimal excerpt:

```json
{
  "provisioning": {
    "payments": {
      "provider": "stripe",
      "compliance": {
        "kyc_required": true,
        "pci_scope": "SAQ-A",
        "user_must_acknowledge": [
          "Stripe will perform KYC directly with you before live payments are enabled.",
          "Your Stripe secret key is never read by the agent."
        ]
      },
      "rollback_behavior": {
        "auto_destroyable": false,
        "manual_steps_if_not": "Visit dashboard.stripe.com → Settings → Account → Close account."
      }
    }
  },
  "connections": {
    "wires": [
      {
        "from": "supabase.service_role_key",
        "to": "vercel.env.SUPABASE_SERVICE_ROLE_KEY",
        "scope": "production",
        "token_type": "ephemeral_oidc"
      }
    ]
  },
  "policy": {
    "approval_required": ["dns_transfer", "plan_upgrade", "production_delete"],
    "secret_policy": {
      "long_lived_tokens": false,
      "max_token_ttl_minutes": 15
    }
  }
}
```

The full spec is in [`ship.json`](./ship.json).

---

## What providers need to implement

Ship is proposed as an extension to the [Model Context Protocol](https://modelcontextprotocol.io) — the open standard Anthropic publishes for agent-to-tool communication. MCP already defines how an agent discovers and calls tools. Ship adds a provisioning and secret-exchange layer on top of it.

A provider that implements Ship is, in MCP terms, an **MCP server** that exposes a standard set of provisioning tools. Any MCP-compatible agent can then call those tools without provider-specific integration work.

A provider declares compatibility via MCP's existing well-known discovery mechanism:

```
GET https://api.yourprovider.com/.well-known/mcp.json
```

And registers six capabilities under the `ship/v1` MCP namespace:

| Endpoint | Purpose |
|---|---|
| `POST /mcp/authorize` | Accept the agent's OIDC token, show the user a PKCE consent screen, return a scoped Delegation Token |
| `POST /mcp/provision` | Provision the declared resource. Must be idempotent on `session_id` + resource spec |
| `POST /mcp/secret-push` | Push a named output to another provider's `/mcp/secret-receive` over mTLS |
| `POST /mcp/secret-receive` | Accept a secret from another provider and write it to your secret store |
| `POST /mcp/rollback` | Destroy all resources provisioned under a `session_id`. Idempotent. Return what was destroyed and what was skipped with reasons |
| `GET /mcp/audit-stream` | Stream audit events for the session as newline-delimited JSON. The user — not the agent — controls export and retention |

### The authorization flow in detail

```
POST /mcp/authorize
{
  "agent_oidc_token": "<signed JWT from Ship agent>",
  "session_id": "ship_01JTY7J9K3P8X4",
  "requested_scopes": ["provision", "secret-push"]
}

→ 200 OK
{
  "delegation_token": "<JWT>",
  "granted_scopes": ["provision", "secret-push"],
  "expires_at": "2026-05-11T09:15:00Z",
  "revocation_endpoint": "https://api.yourprovider.com/mcp/revoke"
}
```

The consent screen must be shown directly to the user. It must not be proxied through the agent. This is the moment the user sees exactly what they are authorising and for how long.

### Token constraints providers must enforce

- Tokens are bound to `session_id` + `provider_id`. They cannot be used across sessions.
- `dns:transfer`, `billing:upgrade`, and `database:delete` scopes must never be granted without a separate explicit human approval step, regardless of what the agent requests.
- Providers must implement [RFC 7009](https://www.rfc-editor.org/rfc/rfc7009) token revocation. When a session ends, all tokens for that session must be revocable in a single call.
- Tokens must not be issuable for longer than 15 minutes without a fresh consent interaction.

### The secret push contract

When the agent issues a secret-push instruction, the source provider calls the destination's `/mcp/secret-receive` directly:

```
Source provider → mTLS → Destination provider /mcp/secret-receive
{
  "session_id": "ship_01JTY7J9K3P8X4",
  "secrets": { "DATABASE_URL": "<encrypted value>" },
  "source_provider": "supabase"
}
```

The inter-provider token authorising this call is issued during the authorization step and is scoped to `secret-push` only. The agent never participates in this data path. It only receives a push confirmation:

```json
{ "pushed": ["DATABASE_URL"], "failed": [] }
```

---

## Why each party benefits

**Vercel** — Developers arrive with a fully configured project rather than an empty dashboard. Reduced time-to-first-deployment, higher activation rates on paid plans.

**Supabase** — Agent-driven provisioning collapses the gap between account creation and first query. Compliance with `expose_raw_engine_env: false` means service role keys never appear in human-readable logs.

**Stripe** — Faster path from account creation to first test payment, without compromising the KYC gate that Stripe needs to maintain. Stripe retains full billing ownership. The protocol makes Stripe's agentic payment APIs a first-class citizen of the startup stack.

**Anthropic and agent vendors** — A more capable agent is a more valuable agent. `/ship` is a showcase capability: the first time a developer types one command and has a production-ready stack is the moment they understand what AI-native tooling actually means. Every new provider that joins compounds the value.

**Founders** — Hours of dashboard-hopping replaced by a single guided session. No secrets in clipboard history. No typos in environment variable names. No `NEXT_PUBLIC_` prefix forgotten in production. A paper trail of every action the agent took, stored with the code.

---

## Governance

**MCP is owned by Anthropic.** The Model Context Protocol specification is maintained at [modelcontextprotocol.io](https://modelcontextprotocol.io) and the [modelcontextprotocol GitHub org](https://github.com/modelcontextprotocol). This is the right home for Ship.

Rather than create a new standards body, the proposal is:

1. **Submit Ship as a formal MCP extension** to the `modelcontextprotocol` spec repository — the same process used for other MCP capability extensions.
2. **Recruit three reference implementors** — Stripe, Vercel, and Supabase — to build and ship the six endpoints. Each retains full control of their implementation; the spec only defines the interface contract.
3. **Anthropic publishes a compliance test suite** alongside the extension. Any provider that passes is listed in the MCP registry as Ship-compatible. No working group votes, no membership fees, no gatekeeping.

This framing changes the coordination problem entirely. Instead of convincing four companies to co-found a new organisation, it becomes: convince Anthropic to accept one extension PR, then convince three providers to implement six endpoints they already have the infrastructure for. Each of those providers already has a relationship with Anthropic via the MCP ecosystem.

The strawman spec lives in [`ship.json`](./ship.json). The right next step is a PR against [github.com/modelcontextprotocol/specification](https://github.com/modelcontextprotocol/specification).

---

## Status

This is a draft proposal — `v0.1`, `status: draft`. The spec in `ship.json` is a starting point for conversation, not a finished standard.

The open questions that need resolution in the MCP extension PR:

- How should inter-provider token trust be bootstrapped? (mutual TLS certificate pinning, or a shared OIDC federation via MCP's existing identity model?)
- What is the right revocation model when a session times out mid-provision?
- How should providers signal the `compliance_acknowledgement` step for regions with different KYC requirements (UK, EU, US)?
- Should the audit stream event schema be standardised in the extension, or left as a provider implementation detail?
- How does Ship interact with MCP's existing tool-call authorization model — extension on top, or replacement?

If you work at Vercel, Supabase, Stripe, Cloudflare, Resend, or Anthropic — this is an invitation. The spec is short. The endpoint contract is six tools. The right venue is a PR against the MCP specification repo.

The goal is a world where the hardest part of starting a company is the idea, not the infrastructure.

---

*Ship Protocol v0.1 · [ship.json](./ship.json) · Proposed MCP extension for [modelcontextprotocol/specification](https://github.com/modelcontextprotocol/specification)*
