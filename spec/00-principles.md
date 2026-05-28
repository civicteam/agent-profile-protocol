# 00 - Principles, terminology, ordering

## Goals

A protocol that:

1. **Describes** the manipulation layer between an agent client and one or
   more upstream MCP servers — what the agent sees on `tools/list`, what
   happens to a `tools/call` request on its way to the server, and what
   happens to the response on its way back.
2. Is **declarative and portable** — a single file (a multi-document YAML
   stream) can be round-tripped to and from a host's storage.
3. Is **modular** — a host that implements only guardrails, or only
   transforms, can ignore the rest of the document.
4. Treats MCP as the v1 transport but **does not encode MCP's RPC handshake**
   anywhere. The only MCP concept the spec depends on is the tool definition
   (name + input schema + optional output schema). This keeps the door open
   for non-MCP transports later.

## Non-goals

* **Connection setup.** How a host discovers, configures, and authenticates
  an upstream MCP server is out of scope. The protocol attaches to
  connections the host already has.
* **Credential acquisition flows.** OAuth dances, MFA challenges, API-key
  prompts, refresh schedules — all host concerns. The protocol does declare
  where the resulting credential blob is **persisted and retrieved** (see
  _05 - Credentials_), but it never describes how that blob is obtained or
  what shape it has.
* **The agent itself.** Model identity, sampling settings, system prompt,
  activity loop. The protocol describes the agent's _effective tool
  interface_, not the agent.
* **Skill bundling.** A profile is not a skill platform. It overlays
  behaviour on connections; it does not bundle prompts, lifecycles, or
  cross-platform skill semantics.
* **Re-specifying MCP.** Where the protocol references MCP types (e.g.
  `inputSchema`), it does so by reference.

## Terminology

| Term | Meaning |
| --- | --- |
| **Profile** | The top-level document stream. One profile == one YAML file containing one or more documents. |
| **Document** | One YAML object within the profile stream. Bound to zero or more connections via `matches`. |
| **Connection** | An upstream MCP server the host has registered. Identified to profiles by `name`, `type`, and (optionally) `version`. The connection itself is the host's concern; the protocol only references it. |
| **Match** | The act of binding a document to a connection because the document's `matches` block accepts the connection's metadata. |
| **Tool** | A single callable exposed by a connection, in MCP's `Tool` shape. The protocol does not redefine this. |
| **Transform** | An operation that modifies, adds, or removes a tool definition from what the agent sees (`alias`, `clone`, `filter`, `preset-parameter`). |
| **Processor** | A best-effort operation that reshapes a request or response body without changing the tool definition (`htmlToMarkdown`, `pickFields`, …). |
| **Guardrail** | A gate. Evaluates a condition on a request or response body and yields `allow`, `block`, `redact`, or `require-approval`. May abort the call. |
| **Stage** | One of `list`, `request`, `response` — the points at which transforms, processors, and guardrails may attach. |

## Ordering

Within a single execution of one tool call, layers run in this order:

```
agent
  ↓
[ list ]      transforms  →  list-stage guardrails
  ↓
[ request ]   request-stage transforms  →  processors  →  guardrails
  ↓
upstream MCP server
  ↓
[ response ]  processors  →  guardrails  →  response-stage transforms
  ↓
agent
```

Two principles govern this ordering:

1. **Guardrails see what the agent will see.** Response processors run
   _before_ response guardrails so a redact or block guardrail evaluates the
   same body the agent will see. Symmetrically, request processors run
   _before_ request guardrails so a guardrail evaluates the body that will
   actually be sent.
2. **Transforms bracket the chain.** Tool-definition transforms run first on
   list (because they decide what the agent sees) and last on response (so
   clone-shape edits map upstream results back into the agent-facing
   structure).

Within a stage, items of the same kind run in `executionIndex` order (lower
first). See _06 - Execution chain_.

## Scope boundaries

The protocol covers everything between the agent and the upstream MCP
server's wire format, **and nothing else**.

| Boundary | What's in | What's out |
| --- | --- | --- |
| Agent | Effective tool surface (post-transform), agent-facing error reasons from guardrails. | Model identity, sampling settings, system prompt, activity loop. |
| Connection | The tool definition (`name`, `inputSchema`, `outputSchema`, description) the host receives from the upstream. | The connection's transport, auth, lifecycle. The host owns these. |
| Profile | Multi-document YAML stream, `matches` block, layered overlays. | Profile storage, version control, distribution. |

## Open questions

* **Inheritance.** Should one document be able to extend another (e.g. a
  base profile + per-tenant overlay)? v1 says no — composition via document
  order already covers most overlay use cases. `extends:` is a v1.1
  candidate once we see real overlap patterns.
* **Versioning.** `apiVersion: civic.com/agent-profile/v1alpha1`. Explicit
  about being pre-stable; breaking changes are allowed inside `v1alphaN`.
* **Processors vs guardrails.** They share infrastructure (JSONPath,
  condition shape, the same engine). v1 keeps them separate because
  processors are best-effort and guardrails can fail-stop. A merger (e.g. a
  single `rules:` list with a `mayBlock` flag) is on the table for v1.1.
* **Non-MCP transports.** The spec is shaped so a CLI- or OpenAPI-style
  transport could slot in by adding a new `type` to `matches`. No
  implementation in v1.
