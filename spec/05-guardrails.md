# 05 - Guardrails

A **guardrail** is a _gate_. It evaluates a condition and yields one of: `allow`, `block`, `redact`, `require-approval`. Unlike a processor, a guardrail can stop the chain.

This is exactly the existing `schema.constraint` model in Civic MCP (request- and response-side `ConstraintTemplate` rows), with a tighter separation from processors.

## Guardrail schema

```yaml
Guardrail:
  id: string?
  stage: enum                  # list | request | response
  target: ToolSelector         # which tools / sources / tags this applies to
  enabled: boolean?            # default true
  executionIndex: int?         # within-stage ordering; default 0

  when:
    schemaPath: string         # JSONPath into the relevant body (see "JSONPath dialect" in 04-processors)
    operator: enum             # equals, includes, excludes, exists, regex, less_than, ...
    value: any?                # operator-dependent

  onNoMatch: enum?             # fail-closed | allow; see "No-match behaviour" below
  outcome: enum                # allow | block | redact | require-approval
  outcomeParams: object?       # outcome-specific config

  # Optional human-facing strings:
  reason: string?              # error/explanation shown when outcome triggers
  severityLevel: enum?         # info | warn | critical (informational, no enforcement)
```

`schemaPath` and any path inside `outcomeParams` use the same JSONPath dialect as processors — see [JSONPath dialect](./04-processors.md#jsonpath-dialect) in _04 - Pre- and post-processors_.

`stage` is one of:

| Stage | Evaluated against | When |
| --- | --- | --- |
| `list` | A tool definition | While building `tools/list`. Allows hiding a tool conditionally (e.g. user lacks a permission). |
| `request` | The incoming `tools/call.params.arguments` | Before dispatch. |
| `response` | The post-processed response body the agent will see | After processors, before transforms. |

`when` is the gate condition. The guardrail **fires** when `when` evaluates to true — `outcome` is what to do when it fires.

## No-match behaviour

A guardrail's `schemaPath` can yield **zero matches** — the field the guardrail was meant to inspect simply isn't present in the payload. Without an explicit rule, this becomes a silent leak path: a redact guardrail that finds nothing to redact lets the original body through unchanged.

`onNoMatch` controls what happens in that case:

| Value | Behaviour |
| --- | --- |
| `fail-closed` | Treat zero matches as a guardrail failure. The call is blocked with `reason` (or a generic "guardrail path did not match" message). |
| `allow` | Treat zero matches as "guardrail did not fire". The call continues as if the condition evaluated to false. |

**Defaults**, when `onNoMatch` is omitted:

| Outcome | Default `onNoMatch` |
| --- | --- |
| `block` | `fail-closed` — if you can't evaluate the gate, don't open it. |
| `redact` | `fail-closed` — if you can't find the field to redact, don't pass the payload through. |
| `require-approval` | `allow` — approval gates that can't even match the trigger condition shouldn't pause every call. |
| `allow` | `allow` — a positive override that doesn't match is just a no-op. |

A profile MAY override these per guardrail (e.g. `onNoMatch: allow` on a redact rule when the field is genuinely optional).

## Outcomes

### `allow`

Always-allow. Used as a positive override (rare in practice). Mostly useful for testing.

### `block`

Reject the call/response with `reason` as the agent-facing error.

```yaml
outcome: block
reason: "Sending email to recipients outside @civic.com is not allowed in this profile."
```

### `redact`

Replace the value at `outcomeParams.path` with `outcomeParams.replacement` (default `"[REDACTED]"`). Continue with the modified body.

```yaml
outcome: redact
outcomeParams:
  path: "$.body"          # defaults to the schemaPath if omitted
  replacement: "[REDACTED]"
```

This is the existing `redact` action.

> Use `outcome: redact` for **conditional** rewrites where failure must be fail-closed (e.g. "if this field matches a secret pattern, replace it"). For **unconditional** field stripping, prefer the `removeFields` / `pickFields` processors (see _04 - Pre- and post-processors_) — those are best-effort and don't abort the call if they misfire.

### `require-approval`

Pause the call until a human approves. Behaviour matches today's "Approve If" / `ask` flow:

```yaml
outcome: require-approval
outcomeParams:
  prompt: "Approve sending an email to {{params.to[0]}}?"
  expirySeconds: 300
```

The runtime is responsible for the actual approval surface (UI, push notification, etc.); the protocol declares the intent. A skipped/denied approval is treated as `block`.

## Targeting

`target` uses the same `ToolSelector` as transforms (_03 - Tool transforms_):

```yaml
target:
  sourceId: "gmail"
  toolName: "send_email"
```

`target.toolName` may be `"*"` to match all tools on a source. `target.sourceId: "*"` matches all sources — useful for cross-cutting guardrails (e.g. always-redact PII patterns).

A guardrail with `target.tags: ["destructive"]` matches by `meta.tags`. This is the recommended pattern for "ask before any destructive call":

```yaml
- stage: request
  target: { sourceId: "*", tags: ["destructive"] }
  when: { schemaPath: "$", operator: "exists", value: true }
  outcome: require-approval
  outcomeParams: { prompt: "Approve {{tool.name}}?" }
```

## Multi-level scope

The protocol document is a single profile. Today's runtime layers constraints at account / profile / user levels; that layering is a runtime concern, not a protocol one. The runtime is responsible for merging multi-level configurations into a single effective profile before evaluating it.

(See _00 - Principles_ — `extends:` is a v1.1 candidate.)

## When guardrails run

See _08 - Execution chain_. Summary:

* `list` guardrails: while materialising the tool list, after transforms.
* `request` guardrails: after request processors, before credential injection and dispatch. (Guardrails evaluate the body that will actually be sent.)
* `response` guardrails: after response processors, before response transforms (`clone.responseShape`). (Guardrails evaluate the body the agent will actually see.)

Within a stage, guardrails run in `executionIndex` order. The first guardrail with a non-`allow` outcome wins for `block` and `require-approval`. For `redact`, the runtime continues to evaluate further guardrails on the redacted body (so multiple redactions can stack).

## Worked examples

### Block external recipients on Gmail

```yaml
guardrails:
  - stage: request
    target: { sourceId: "gmail", toolName: "send_email" }
    when:
      schemaPath: "$.to[*]"
      operator: "not_ends_with"
      value: "@civic.com"
    outcome: block
    reason: "Outbound email is restricted to @civic.com recipients."
```

### Redact API keys appearing in any response

```yaml
guardrails:
  - stage: response
    target: { sourceId: "*" }
    when:
      schemaPath: "$..*"
      operator: "regex"
      value: "(?i)api[_-]?key|secret|bearer\\s+[A-Za-z0-9._-]{20,}"
    outcome: redact
    outcomeParams: { replacement: "[REDACTED]" }
    executionIndex: 0
```

### Require approval for any tool tagged `destructive`

```yaml
guardrails:
  - stage: request
    target: { sourceId: "*", tags: ["destructive"] }
    when: { schemaPath: "$", operator: "exists", value: true }
    outcome: require-approval
    outcomeParams: { prompt: "Approve destructive call to {{tool.name}}?" }
```

## What this maps to today

| Field | Today's `ConstraintTemplate` field |
| --- | --- |
| `stage` | `timing` (`request` / `response`) plus a new `list` stage |
| `target.sourceId` | `mcpServerId` |
| `target.toolName` | `operationName` |
| `when.schemaPath` | `schemaPath` |
| `when.operator` | `operator` |
| `when.value` | `value` |
| `outcome` | `action` (with `redact` / `reject` / `ask` mapped to `redact` / `block` / `require-approval`) |
| `outcomeParams` | `actionParams` |
| `reason` | `customErrorMessage` |
| `severityLevel` | `securityLevel` |

## Why guardrails and processors are separate

They share infrastructure (JSONPath, condition shape, the same engine) but they are fundamentally different:

* A processor _cannot fail-stop_. Its failure is logged and swallowed.
* A guardrail _can fail-stop_. Its trigger may abort the call.

If you find yourself reaching for a processor to enforce a rule, it should be a guardrail with `outcome: block` or `outcome: redact`. If you find yourself reaching for a guardrail purely to reshape data, it should be a processor.

The discriminator on the wire is `kind: Guardrail` vs `kind: Processor` (or, equivalently, separate top-level lists `guardrails:` and `processors:`).
