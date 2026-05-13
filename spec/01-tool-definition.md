# 01 - Tool definition

The base `Tool` schema is the bedrock that every other layer references. It is intentionally a **superset of the MCP** `Tool` type so that an MCP source's `tools/list` output can be lifted into the protocol with no transformation.

## Schema

```yaml
# A single agent-callable tool.
Tool:
  name: string                # required, unique within its (sourceId, scope)
  title: string?              # human label (MCP optional)
  description: string         # required; agent-facing
  inputSchema: JSONSchema     # required; MCP-compatible
  outputSchema: JSONSchema?   # optional; MCP-compatible
  annotations:                # optional, MCP-compatible
    readOnly: boolean?
    destructive: boolean?
    idempotent: boolean?
  meta: object?               # protocol-extension metadata; see below
```

`name`, `description`, `inputSchema`, `outputSchema`, and `annotations` are the MCP fields. `meta` is the protocol extension hook.

## `meta` — protocol-extension metadata

`meta` is a free-form object that the protocol uses to attach information the agent does not need to see but the runtime does. It is **never sent to the agent**; the runtime strips it before serializing the tool list to the MCP client.

Reserved keys (other layers may use the same `meta` block):

| Key | Type | Used by | Purpose |
| --- | --- | --- | --- |
| `meta.sourceId` | string | runtime / multiplexer | Which `Source` this tool came from. Required for any tool produced by lifting from a source. |
| `meta.upstreamName` | string | transforms (alias) | The original `name` on the source, if `name` was rewritten. |
| `meta.scope` | `{ type: "base" }` \| `{ type: "skill", skillAlias: string }` | multiplexer / skills | Which scope produced this tool. |
| `meta.tags` | string\[\] | guardrails / audit | Optional labels useful for matching (e.g. `["destructive", "external"]`). |
| `meta.billing` | object | runtime | Per-call cost, rate limits, etc. — not part of v1 spec, runtime-defined. |

A tool that came from a source unmodified looks like:

```yaml
- name: send_email
  description: "Sends an email via Gmail."
  inputSchema:
    type: object
    properties:
      to:      { type: array, items: { type: string, format: email } }
      subject: { type: string }
      body:    { type: string }
    required: [to, subject, body]
  meta:
    sourceId: gmail
```

A tool that has been renamed (alias transform) looks like:

```yaml
- name: email_team
  description: "Sends an email to the team."
  inputSchema: {...}
  meta:
    sourceId: gmail
    upstreamName: send_email
```

## Where `Tool` records appear in a profile

You do **not** typically write `Tool` records by hand at the top level. The runtime materialises them by:

1. Reading the upstream source's `tools/list` (or the source's static schema, if declared inline).
2. Applying any `transforms` that produce/clone/rename/filter tools.
3. Stamping `meta` according to the operations applied.

The materialised list — what the agent sees on `tools/list` — is the result of this pipeline. See _09 - Execution chain_.

That said, **a profile may declare tools inline** for sources that don't supply their own schema (e.g. a `cli-exec` source). When a source declares `tools` inline, those tools are the canonical set for that source.

```yaml
sources:
  - id: my-cli
    kind: exec        # not in v1, illustrative for non-MCP path
    tools:
      - name: list_buckets
        description: "Lists S3 buckets in the region."
        inputSchema:
          type: object
          properties:
            region: { type: string }
          required: [region]
```

### Inline tools vs. upstream `tools/list`

When a source both declares `tools` inline **and** advertises its own `tools/list` (the partial-overlap case), the rule is:

| Situation | Result |
| --- | --- |
| Tool name appears inline only | Inline definition is used. The upstream is never asked about it. |
| Tool name appears upstream only | Upstream definition is used as-is (lifted with `meta.sourceId` stamped). |
| Tool name appears in both | **Inline wins.** The inline definition fully replaces the upstream one (no field-level merge). The runtime SHOULD log a warning so operators can spot accidental shadowing. |

A profile that wants to suppress an upstream tool without redefining it should use a `filter` transform (_03 - Tool transforms_), not an inline override.

## Why this is the bedrock

Every other layer in the protocol references tools by `(sourceId, name)` or by tag. Keeping the tool definition itself MCP-shaped means:

* A v1 runtime can consume an MCP `tools/list` response directly.
* Future non-MCP sources only need to produce something that fits this schema.
* An external implementer who only cares about (e.g.) guardrails can read just this file plus _05 - Guardrails_ and skip the rest.

## What is deliberately **not** in `Tool`

* Auth / credential requirements. These belong in _07 - Credentials_, keyed by `sourceId` (or finer by `(sourceId, name)`).
* Rate limits / billing. These are runtime concerns; the protocol leaves a slot in `meta` but does not standardise the shape in v1.
* "Approval required" / risk levels. These belong in guardrails (_05 - Guardrails_), not on the tool itself, so that the same tool can be unrestricted in one profile and gated in another.
* Hidden / disabled state. Filtering is a transform (_03 - Tool transforms_), not a property of the tool.
