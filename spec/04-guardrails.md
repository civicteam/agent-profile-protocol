# 04 - Guardrails

A **guardrail** is a _gate_. It evaluates a condition and yields one of:
`allow`, `block`, `redact`, `require-approval`. Unlike a processor (_03 -
Processors_), a guardrail can stop the chain.

## Guardrail schema

```yaml
Guardrail:
  stage: enum                  # list | request | response
  target: string | [string]?   # which tool(s) — same shape as transforms / processors
  enabled: boolean?            # default true
  executionIndex: int?         # within-stage ordering; default 0

  when:
    schemaPath: string         # JSONPath into the relevant body (see "JSONPath dialect" in 03-processors)
    operator: enum             # equals | includes | excludes | exists | regex | less_than | …
    value: any?                # operator-dependent

  onNoMatch: enum?             # fail-closed | allow; see "No-match behaviour" below
  outcome: enum                # allow | block | redact | require-approval
  outcomeParams: object?       # outcome-specific config

  # Optional human-facing strings:
  reason: string?              # error/explanation shown when outcome triggers
  severityLevel: enum?         # info | warn | critical (informational, no enforcement)
```

`schemaPath` and any path inside `outcomeParams` use the same JSONPath
dialect as processors — see [JSONPath dialect](./03-processors.md#jsonpath-dialect)
in _03 - Pre- and post-processors_.

`stage` is one of:

| Stage | Evaluated against | When |
| --- | --- | --- |
| `list` | A tool definition | While materialising the agent-facing tool list. Allows hiding a tool conditionally. |
| `request` | The incoming `tools/call.params.arguments` | Before dispatch. |
| `response` | The post-processed response body the agent will see | After processors, before any response-side transforms. |

`when` is the gate condition. The guardrail **fires** when `when` evaluates
to true — `outcome` is what to do when it fires.

## Targeting

`target:` has the same shape as in transforms (_02 - Tool transforms_) and
processors (_03 - Pre- and post-processors_):

* `target: foo` — one tool.
* `target: [foo, bar]` — several tools.
* _omitted_ — every tool in the matched connection(s).

Like processors, guardrails target by **current tool name** (the name the
agent sees after transforms). A guardrail targeting a name that no transform
produces is a startup error.

There is no `target.sourceId`, no `target.tags`, no `target.scope` — those
keys were for the old multi-source / skill model and have been removed.
Cross-connection rules should live in documents with no `matches` block (see
_01 - Matching_).

## No-match behaviour

A guardrail's `schemaPath` can yield **zero matches** — the field the
guardrail was meant to inspect simply isn't present in the payload. Without
an explicit rule, this becomes a silent leak path: a redact guardrail that
finds nothing to redact lets the original body through unchanged.

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

A profile MAY override these per guardrail (e.g. `onNoMatch: allow` on a
redact rule when the field is genuinely optional).

## Outcomes

### `allow`

Always-allow. Used as a positive override (rare in practice). Mostly useful
for testing.

### `block`

Reject the call/response with `reason` as the agent-facing error.

```yaml
outcome: block
reason: "Sending email to recipients outside @civic.com is not allowed in this profile."
```

### `redact`

Replace the value at `outcomeParams.path` with `outcomeParams.replacement`
(default `"[REDACTED]"`). Continue with the modified body.

```yaml
outcome: redact
outcomeParams:
  path: "$.body"          # defaults to the schemaPath if omitted
  replacement: "[REDACTED]"
```

> Use `outcome: redact` for **conditional** rewrites where failure must be
> fail-closed (e.g. "if this field matches a secret pattern, replace it").
> For **unconditional** field stripping, prefer the `removeFields` /
> `pickFields` processors (see _03 - Pre- and post-processors_) — those are
> best-effort and don't abort the call if they misfire.

### `require-approval`

Pause the call until a human approves.

```yaml
outcome: require-approval
outcomeParams:
  prompt: "Approve sending an email to {{params.to[0]}}?"
  expirySeconds: 300
```

The host is responsible for the actual approval surface (UI, push
notification, etc.); the protocol declares the intent. A skipped / denied
approval is treated as `block`.

## When guardrails run

See _06 - Execution chain_. Summary:

* `list` guardrails: while materialising the tool list, after transforms.
* `request` guardrails: after request processors, before dispatch.
  Guardrails evaluate the body that will actually be sent.
* `response` guardrails: after response processors, before any response-side
  transforms. Guardrails evaluate the body the agent will actually see.

Within a stage, guardrails run in `executionIndex` order. The first
guardrail with a non-`allow` outcome wins for `block` and
`require-approval`. For `redact`, the host continues to evaluate further
guardrails on the redacted body (so multiple redactions stack).

## Worked examples

### Block external recipients on a Gmail-flavoured connection

```yaml
matches:
  name: gmail

guardrails:
  - stage: request
    target: send_email
    when:
      schemaPath: "$.to[*]"
      operator: "not_ends_with"
      value: "@civic.com"
    outcome: block
    reason: "Outbound email is restricted to @civic.com recipients."
```

### Redact API keys appearing in any response across all connections

```yaml
# No `matches:` → applies to every connection.

guardrails:
  - stage: response
    when:
      schemaPath: "$..*"
      operator: "regex"
      value: "(?i)api[_-]?key|secret|bearer\\s+[A-Za-z0-9._-]{20,}"
    outcome: redact
    outcomeParams: { replacement: "[REDACTED]" }
    executionIndex: 0
```

### Require approval for a specific destructive tool

```yaml
guardrails:
  - stage: request
    target: [dropTable, truncateTable]
    when: { schemaPath: "$", operator: "exists", value: true }
    outcome: require-approval
    outcomeParams: { prompt: "Approve destructive SQL call?" }
```

Cross-cutting "any destructive tool" rules now live in a document with no
`matches:` block plus an explicit `target:` list. v1 does not have
annotation-based targeting (e.g. matching `annotations.destructive: true`);
that's a v1.1 candidate once we see real demand.

## Why guardrails and processors are separate (for now)

They share infrastructure (JSONPath, condition shape, the same engine) but
behave differently at failure:

* A processor _cannot fail-stop_. Its failure is logged and swallowed.
* A guardrail _can fail-stop_. Its trigger may abort the call.

If you find yourself reaching for a processor to enforce a rule, it should
be a guardrail with `outcome: block` or `outcome: redact`. If you find
yourself reaching for a guardrail purely to reshape data, it should be a
processor.

A merger (single `rules:` list with a `mayBlock` flag) is on the v1.1 table.
v1 keeps the two separate to make the fail-stop distinction syntactic.
