# 00 - Principles, terminology, ordering

## Goals

A protocol that:

1. **Describes** the layer between an agent and its underlying tools — what is exposed, with what shape, with what restrictions, with what credentials.
2. Is **declarative and portable** — a single file can be round-tripped to and from the runtime's storage.
3. Is **modular** — a runtime that only implements guardrails, or only credential injection, can ignore the rest of the document.
4. Treats MCP as the v1 transport but **does not encode MCP's RPC handshake** anywhere. The only MCP concept the spec depends on is the tool definition (name + input schema + optional output schema). This keeps a path open for non-MCP sources later.

## Non-goals

* The agent itself (model, system prompt, activity loop). The protocol describes the agent's _effective interface_, not the agent.
* Replacing skills platforms. Skills in this protocol are **named bundles of in-protocol resources**, nothing more.
* Re-specifying MCP. Where the protocol references MCP types (e.g. `inputSchema`), it does so by reference.

## Terminology

| Term | Meaning |
| --- | --- |
| **Profile** | The top-level document. One profile == one declarative description of an agent's effective tool surface. |
| **Source** | An upstream supplier of tools. Today: an MCP server. Later potentially: a CLI wrapper, an OpenAPI spec, etc. |
| **Tool** | A single callable. The schema is a superset of the MCP `Tool` (see _01 - Tool definition_). |
| **Transform** | An operation that **modifies, adds, or removes a tool definition** before it reaches the agent (alias, clone, filter, preset params). |
| **Processor** | A `pre` or `post` operation that **changes content** (request body, response body) without changing the tool definition. |
| **Guardrail** | A gate. Evaluates a condition and yields an outcome (`allow`, `block`, `redact`, `require-approval`). |
| **Audit** | A read-only observer. Cannot change requests, responses, or outcomes. |
| **Skill** | A named bundle of (sources, transforms, processors, guardrails) scoped under an alias. |
| **Multiplexer** | The runtime behaviour that combines all sources + skills into one exposed surface. |
| **Stage** | One of `list`, `request`, `response`. A point in the chain at which transforms / processors / guardrails can attach. |

## Ordering

This is the v1 ordering decision.

### Decision

Within a single execution of one tool call, layers run in this order:

```
agent
  ↓
[ list ]      transforms (definition)  →  guardrails (list)  →  audit (list)
  ↓
[ request ]   transforms (alias/clone/preset)  →  processors (pre)  →  guardrails (request)  →  audit (request)  →  credential injection
  ↓
upstream source
  ↓
[ response ]  processors (post)  →  guardrails (response)  →  transforms (clone-shape)  →  audit (response)
  ↓
agent
```

Three principles govern this ordering:

1. **Guardrails see what the agent will see.** Postprocessors run _before_ response guardrails so a redaction or block guardrail evaluates the same text the agent will see. Symmetrically, request preprocessors run _before_ request guardrails so a guardrail evaluates the body that will actually be sent.
2. **Audit runs last in each stage.** Audit observes the final agent-facing form (or, on request, the final form just before credential injection). Audit cannot influence the call.
3. **Credential injection is closest to the wire.** It runs after audit so the audit record never contains injected secrets, and after every other request layer so secrets don't leak into earlier processors/guardrails.

### Within a stage

Within a single stage (e.g. `response`), layers of the same kind run in `executionIndex` order (lower first), matching the existing `ConstraintsAndPostProcessorsHook` model. This lets users intermix several guardrails or several processors without forcing them into rigid bands.

### Counter-arguments considered

* _"Apply guardrails on the upstream side so they survive transform changes."_ Rejected — this couples guardrail config to upstream identifiers that the agent never sees, and makes the agent-facing reality harder to reason about. If a transform renames `gmail.send_email` to `email_team`, a guardrail targeting `email_team` is the natural unit; if the upstream is later replaced with a different gmail implementation, the guardrail still applies.
* _"Apply processors after guardrails so guardrails see raw responses."_ Rejected — agents only ever see the post-processed form; guardrails should evaluate that.

## Scope boundaries

The protocol covers everything between the agent and the upstream source's wire format. It deliberately stops short of:

| Boundary | What's in | What's out |
| --- | --- | --- |
| Agent | Effective tool surface, prompts injected as part of skills/instructions. | Model identity, sampling settings, system prompt outside skills, activity loop. |
| Upstream source | The tool definition (`name`, `inputSchema`, `outputSchema`, description) and the binding to a transport. | The internal implementation of the tool, the deployment of the upstream server. |
| Credentials | Provider config, satisfaction flow shape, injection target (env var, header, file). | The encrypted credential value at rest (that's runtime storage). |
| Skill | Local bundling: which sources/transforms/etc. activate when this skill is loaded. | Cross-platform skill semantics (Claude skill format vs others). |

## Open questions

* **Nesting / inheritance.** Should profiles be able to extend other profiles (account-level baseline + per-agent overrides)? v1 says no; we expect to add `extends:` in v1.1 once we see real overlap patterns.
* **Versioning.** `apiVersion: civic.com/agent-profile/v1alpha1`. We're explicit about being pre-stable. Breaking changes are allowed inside `v1alphaN`.
* `actionPath` JSONPath dialect. The spec inherits the Civic MCP guardrail engine's JSONPath dialect (with `^` for parent, `parse(text)` for nested-JSON parsing). We document the dialect but do not re-derive it.
* **Where does "agent identity" live.** `metadata.id` is a stable identifier for the profile; it's not the agent identity. If a runtime needs to bind a profile to a specific agent (e.g. for analytics distinct ID), that's a runtime concern, not a protocol one.

## What changes from the existing Civic model

This is _not_ a redesign. The protocol is a serialization format over the existing model. Specifically:

* `Tool` mirrors what the hub already exposes via `ScopedActionRegistry`.
* `Transform` covers what `FilterHook`, `AliasHook`, `CloneResultHook`, `ParameterHook`, and `CustomDescriptionHook` do today — driven by `ProfileData` rows.
* `Processor` and `Guardrail` map directly onto the unified `ConstraintTemplate` / `Constraint` model with `type: response.transform` and `type: schema.constraint` respectively.
* `Audit` is the existing `AuditHook`.
* `Credentials` mirrors `OAuthCredentials` + `CredentialData` + `CredentialJob` and the init-sidecar injection pattern.
* `Skill` is a `Profile` row with `type: SKILL`, plus its `ProfileLink`.
* The `multiplexer` is what `ScopedActionRegistry` does.

A migration that emits this format from the DB and re-ingests it should be the first concrete consumer of the spec.
