# OpenApe Agent Recipe

**Version:** 1.0-draft
**Status:** Draft
**Date:** 2026-05-17
**Author:** Patrick Hofmann (Delta Mind GmbH)

## Abstract

An **Agent Recipe** is a declarative, self-contained format that makes a
scheduled, capability-using agent **one-step deployable** from a git
repository. Today such an agent is assembled by hand from disjoint
artefacts — scripts, a password "somewhere", cron entries, a system prompt.
A Recipe collapses that into a description file plus a code part in a repo:
the platform (troop as broker/orchestrator, `@openape/proxy` as the
credential egress) provisions everything from the manifest. This profile
builds on the [Grants](grants.md) / [Delegation](delegation.md) model — a
Recipe's capabilities are brokered, scoped, revocable credentials, not
plaintext secrets the human places.

## Table of Contents

1. [Introduction](#1-introduction)
2. [Artifact & Repo Layout](#2-artifact--repo-layout)
3. [Manifest Schema](#3-manifest-schema)
4. [Sourcing & Trust](#4-sourcing--trust)
5. [Capability & Secret Delivery](#5-capability--secret-delivery)
6. [Params & Augmentation](#6-params--augmentation)
7. [Execution Model](#7-execution-model)
8. [Lifecycle](#8-lifecycle)
9. [Security Considerations](#9-security-considerations)
10. [Schemas](#10-schemas)

---

## 1. Introduction

### 1.1 Purpose

A Recipe author publishes a repo. A user points the platform at
`<repo>@<ref>`. The platform reads the manifest, collects typed params,
obtains explicit one-time consent for each declared capability, binds the
declared credential placeholders (proxy-brokered by default, locally
materialised when required), installs the schedule(s), places the tool code
into the agent runtime, and runs the agent. **Nothing is hand-wired** —
that is why a previously tedious setup becomes trivial.

### 1.2 Requires

- [Grants Protocol](grants.md) / [Delegation Protocol](delegation.md) — a
  capability is a brokered, scoped, revocable grant.
- An orchestrator/broker (reference: **troop**) that owns capability
  lifecycle and the agent↔orchestrator channel.
- A credential egress (reference: **`@openape/proxy`**) for the default,
  agent-never-sees-the-secret delivery path.

This profile defines the **format and its contract**, not a specific
runtime implementation.

## 2. Artifact & Repo Layout

```
<repo>/
  ape-agent.yaml        # the manifest — "the description file, all data"
  tools/                # the code part — handlers/tools the agent invokes
  README.md
```

The repo is the unit of distribution. The manifest is the single source of
truth; `tools/` is real, testable code (not inlined glue).

## 3. Manifest Schema

`ape-agent.yaml` — see [schemas/agent-recipe.json](schemas/agent-recipe.json).

| Field | Req | Description |
|-------|-----|-------------|
| `name` | yes | Recipe identifier. |
| `kind` | yes | `agent` (v1 only — LLM agent with shipped tools). |
| `intent` | yes | The prepared system prompt / task the agent runs each tick. |
| `capabilities` | no | Declared credential **placeholder env-var names** the recipe consumes, e.g. `BLUESKY_HANDLE`, `BLUESKY_APP_PASSWORD`. Each MAY carry `prefer: proxy\|local` — a declarative hint only; delivery remains a platform decision (§5). |
| `params` | no | Typed, deploy-time parameters (e.g. `topic: { type: string, required: true }`). Validated and interpolated into `intent`, tool args and schedules. |
| `schedules` | no | Array of cron expressions. Multiple times are trivial. Empty = on-demand only. |
| `user_addendum` | no | `true` declares that a free-text behavioural layer is allowed (§6). |
| `tools` | no | Tool/entrypoint manifest the agent may invoke (paths under `tools/`). |

The manifest MUST NOT contain secret values — only placeholder names.

## 4. Sourcing & Trust

A Recipe is referenced as `<repo>@<ref>`. **The ref pin is mandatory** — a
floating branch (e.g. `main`) MUST be rejected; integrity derives from
pinning a commit or tag.

`@openape-official-ape-agents/*` repos form a **curated discovery index**
only. They carry **technically identical trust** to any arbitrary repo:
there is no privileged publisher, no auto-granted capability. Every deploy
— first-class or arbitrary — obtains the **same explicit, one-time user
consent** for each declared capability (grant-shaped). "Official" means
findable, not trusted-more.

## 5. Capability & Secret Delivery

The Recipe declares only placeholder env-var names. The broker (troop) owns
the credential lifecycle. Two delivery paths, **transparent to the Recipe**
(the distinction is cosmetic at the manifest layer):

- **Proxy injection (default, principled).** The credential lives at the
  broker; `@openape/proxy` injects/authorises it on the agent's egress and
  enforces the grant. The agent never holds the secret.
- **Local materialisation (when the capability cannot be proxy-brokered —
  non-HTTP, SDK requiring env).** The broker pushes the value over the
  existing orchestrator↔agent channel; the agent writes it to a
  **disposable, broker-owned `.env.local` cache** — never agent-owned
  durable state.

In **both** cases the broker is the single authority that can grant,
**rotate, and revoke** — rotate/revoke are pushed over the same channel
(`.env.local` rewritten or cleared). A Recipe that uses a placeholder env
var MUST declare it; how it is satisfied is not the Recipe's concern.

## 6. Params & Augmentation

Two distinct layers:

- **`params`** — typed, validated at deploy, interpolated into `intent`,
  tool arguments and schedule fields. Example: a recipe whose intent is
  "summarise my feed about {{topic}}" declares `topic` as a required
  param; the user supplies it at deploy.
- **`user_addendum`** — an optional free-text layer **appended to the
  prepared `intent` at runtime**, editable at any time **without
  re-deploy** ("also additionally do X"). Structured slots vs. free
  behavioural extension are kept separate so params stay validatable.

## 7. Execution Model

`kind: agent`: the platform places `tools/` into the agent runtime and
configures an LLM agent from `intent` + `user_addendum` + interpolated
`params` + authorised `capabilities`. Each schedule tick triggers one agent
run with that intent; the agent invokes the shipped tools. Capability
rotate/revoke propagate over the orchestrator↔agent channel.

## 8. Lifecycle

- **Deploy** — one step: `apes agent deploy <repo>@<ref>` or the troop UI
  (equivalent surfaces). Collect params → consent capabilities → bind
  placeholders → install schedules → place tools → run.
- **Update** — re-point the ref; same one-step path.
- **Revoke** — revoking a capability makes the broker push a clear over the
  channel; the next run lacks it. Revoking the Recipe removes
  schedules + bindings.
- **Audit** — every capability use is logged (who/what/when), consistent
  with grants.md.

## 9. Security Considerations

- **No privileged publisher** (§4). Trust comes from explicit user consent,
  not from a repo being "official" — consistent with DDISA's "authority is
  not status".
- **Secret-at-rest.** Local `.env.local` is plaintext at rest. The grant
  model survives **only because the broker owns its lifecycle** (active
  rotate/revoke over the channel) and the agent treats it as a disposable
  cache, never durable state. If `.env.local` became agent-owned, this
  reduces to "password somewhere" and the model is void.
- **Ref pinning** prevents a moved branch from silently changing executed
  code or declared capabilities under an existing consent.
- **Consent is per-capability and one-time**, uniform across sources; a
  Recipe cannot widen beyond what the user authorised.
- **Proxy path is preferred** precisely because the agent never holds the
  secret; local is the acknowledged, declared exception.

## 10. Schemas

| Schema | Reference |
|--------|-----------|
| [agent-recipe.json](schemas/agent-recipe.json) | §3 |
| [grant.json](schemas/grant.json) | grants.md (capability = grant) |
| [delegation.json](schemas/delegation.json) | delegation.md |
