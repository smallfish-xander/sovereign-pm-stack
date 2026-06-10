# n8n Workflow Showcase — Operational Automation for a Self-Hosted AI Stack

A curated, **sanitized** snapshot of production n8n workflows from my self-hosted AI/PM stack. They run a real system every day — a multi-agent RAG platform with a Confluence↔Git↔vector-store knowledge base ([full case study](https://github.com/smallfish-xander/sovereign-pm-stack)).

I picked these because they show the boring, load-bearing parts of running AI in production: keeping a knowledge base consistent, gating quality, governing cost, and staying up. The fun demo is the agent; these are what make it trustworthy.

> **Six workflows are included as importable JSON; a seventh (the regression + security suite) is described but deliberately not published** — see the note in section 5. Curating *out* the one file that maps my live attack surface is the same judgment these workflows are meant to demonstrate.

---

## About this snapshot — and why it's sanitized

These are real exports, de-identified before sharing:

- **No credentials.** n8n stores credentials in an encrypted store separate from workflow definitions. Exports reference them by *name* (`Outline API Key`, `Gilbert Database`) — never the secret value. That's n8n working as designed, and it's the first thing I'd want a reviewer to verify: there is nothing to leak here because the secrets were never in the JSON.
- **Secrets stay in env.** Every sensitive value is a runtime reference — `$env.MISTRAL_API_KEY`, `$env.GITHUB_TOKEN` — resolved from `.env`, never inlined.
- **Identifiers genericized.** My domain, email addresses, GitHub handle, and container names were replaced with placeholders (`example.com`, `sovereign-stack-*`). Internal Docker service hostnames (`http://qdrant:6333`, `http://litellm:4000`) are kept because they're architecture, not secrets — they're only reachable inside the private network.
- **Pinned execution data stripped.** `pinData` can carry real records pulled through a test run. For an AI pipeline over customer data that's a GDPR exposure, so it's emptied here.

The fact that this snapshot *is* sanitized is itself the point: the same stack enforces "never return a raw exception to a user" as a lint rule and runs a security suite before every commit. I sanitize before sharing for the same reason.

---

## The workflows (six included, one described)

### 1. `git-embed-sync.json` — Git → vector store, idempotent and loop-safe
**Trigger:** GitHub webhook + 15-min polling fallback

Embeds changed Markdown/PDF into Qdrant when content lands in Git. The product decisions worth noting:

- **Loop prevention.** This stack syncs content in both directions (Git ↔ Outline wiki). Naive bidirectional sync loops forever. The guard: sync-originated commits are prefixed `[sync]`, and this workflow skips any push where *every* commit carries that marker — `isSyncCommit = msg => msg.startsWith('[sync]')`. One convention, no infinite loops.
- **Config-driven, not hardcoded.** Which files embed is governed by a `sync.yaml` glob spec fetched at runtime, not baked into the workflow. Adding a content type is a YAML edit, not a workflow change.
- **Idempotent.** A `sync_lock` and per-file state table mean re-runs are safe and a webhook + a poll can't double-process the same change.

*Why it matters (PM lens):* most "AI knowledge base" failures aren't model failures — they're content-pipeline failures. This is where retrieval quality is actually won or lost.

### 2. `outline-reverse-sync.json` — wiki → Git, the other half of the loop
**Trigger:** Outline document webhook (create / update / delete / move / rename)

Commits wiki edits back to Git so the vector store re-embeds them. It's the counterpart to #1 and shows the discipline that makes bidirectional sync safe:

- **Writes the `[sync]` marker** that #1 filters on — the two workflows close the loop by convention.
- **Echo suppression** via a sync-lock window: a change that just arrived *from* Git won't be committed straight back.
- **Handles the hard cases** — folder renames, document moves, tree restructuring — not just the happy path.
- **Fails loud:** an error path branches to an email notification instead of silently dropping.

### 3. `embedding-pipeline.json` — the RAG core
**Trigger:** sub-workflow (called by #1 and other producers)

The reusable heart of retrieval: validate → chunk → embed (via the LLM proxy) → store in Qdrant → log. Every content producer in the stack calls this one workflow rather than re-implementing embedding, so chunking strategy and vector schema live in exactly one place.

- **One embedding path, many callers.** Centralizing it means a change to chunk size or model is a single edit, not a hunt across workflows.
- **Idempotent storage.** Deterministic point IDs in Qdrant mean re-embedding the same content updates in place instead of duplicating — the same discipline as the sync layer, one level down.

*Why it matters (PM lens):* "add RAG" is a sentence; the product is in the chunking, the schema, and making re-runs safe. Owning this as shared infrastructure is what keeps retrieval consistent as the surface area grows.

### 4. `agentic-tool-loop.json` — native tool calling, done right
**Trigger:** sub-workflow

The multi-turn loop that lets a model call tools, read results, and call again until it's done. This is the workflow behind a hard-won lesson in the [case study](../README.md): text-based tool-calling fallbacks hallucinate malformed calls under load; native function-calling through the proper API contract is what makes tool use reliable.

- **Bounded loop** with a turn cap — no runaway tool-calling.
- **Budget-aware** — it reads spend limits so an agent can't loop its way into a surprise bill.

*Why it matters (PM lens):* "supports tools" spans a 10× reliability range depending on *how* it's wired. Having built the loop, I scope agent features around the integration mechanism, not the marketing checkbox.

### 5. Regression + security gate — *described, deliberately not published*
**Trigger:** webhook (callable like a CI check)

A single comprehensive suite of regression + security tests covering every service in the stack — health endpoints, auth flows, database integrity, tool behavior, RAG retrieval, and the negative security cases (SSRF guards, path-traversal blocks, exception-message sanitization). It's the automation behind a rule I hold myself to: **tests run before every commit, and security checks are part of that suite, not a separate afterthought.** It runs on every change and is callable on demand like a CI check.

**Why it's not in this repo:** unlike the others, this workflow is an exhaustive, literal map of my running stack — every service, every internal endpoint, the recovery and exec patterns. That's exactly the kind of detail you hand to a hiring team under an application, not something you publish to a search index next to your name. Leaving it out — and saying why — is the same security judgment the workflow itself encodes. Happy to walk through it live or share it directly on request.

*Why it matters (PM lens):* "we'll harden it later" doesn't survive contact with tools that can read email and run code. Security belongs in the definition of done — including the decision of what not to publish.

### 6. `llm-budget-monitor.json` — cost governance for LLM spend
**Trigger:** hourly schedule

Reads budget state from Postgres, aggregates the day's LLM spend, evaluates it against thresholds, emails an alert if a limit is approaching, and auto-resets the daily counter. Pairs with rate/budget limits enforced at the LLM-proxy layer.

*Why it matters (PM lens):* AI features have a variable cost-per-use that most roadmaps ignore until the bill arrives. Observability and a guardrail are how you ship AI without a runaway-cost incident.

### 7. `service-health-monitor.json` — uptime with self-healing
**Trigger:** 2-min schedule

Health-checks the stack and attempts recovery on failure before escalating to an email alert. Cheap, constant, and the reason small outages self-resolve instead of becoming my evening.

---

## What this set is meant to demonstrate

| Principle | Shown by |
|---|---|
| Content pipelines are an architecture problem | #1, #2 (loop prevention, idempotency, config-driven) |
| RAG is shared infrastructure, not a feature | #3 (centralized, idempotent embedding) |
| Tool calling reliability is an engineering choice | #4 (native, bounded, budget-aware) |
| Quality & security are infrastructure, not phases | #5 (described) |
| AI cost is a product constraint to govern | #6 |
| Operational reliability is designed, not hoped for | #7 |
| Knowing what *not* to publish | #5 left out by choice |
| Secrets management done right | all of them (credential externalization, `$env` refs) |

A handful of workflows, not all of them — curation over completeness. The stack runs ~19 active workflows; these are the ones that best show how I think about running software, not just building it.

---

## Importing one

n8n → **Workflows** → **Import from File** → pick a JSON. Each imports as an inactive template; you'd recreate the referenced credentials (by name) and set the `$env` values to make it run. They're shared to read and discuss, not to deploy as-is.
