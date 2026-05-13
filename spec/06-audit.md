# 06 - Audit

An **audit** entry is a read-only observer. It cannot modify a request, response, or outcome. It exists so a profile can declaratively say "send a copy of these calls to this sink" without that sink being a guardrail or processor.

> Audit is the least-implemented layer in current Civic MCP — there is no end-to-end `AuditHook` in the runtime today, only references in specs and partial structured-logging coverage. This section is therefore best read as a forward-looking design rather than a serialization of existing behaviour.

## Audit schema

```yaml
Audit:
  id: string?
  stage: enum                  # list | request | response
  target: ToolSelector
  enabled: boolean?
  sink: AuditSink              # required
  redact: [string]?            # JSONPaths to redact in the audit record
  include:                     # what fields to include in the audit record
    request: boolean?          # default true if stage is request/response
    response: boolean?         # default true if stage is response
    metadata: boolean?         # default true (toolName, sourceId, profileId, timestamp)
```

## Sinks

```yaml
AuditSink:
  type: enum                   # webhook | log | none
  # type=webhook
  url: string
  headers: object?             # static headers; templated via {{secrets.X}}
  signing: { kind: "hmac-sha256", secretRef: string }?

  # type=log
  level: enum?                 # info | debug; default info
  channel: string?             # logger name; runtime-specific

  # type=none — explicit "do not audit" override (rare; used to suppress an inherited audit)
```

In v1, only `webhook` and `log` are required to be supported by a runtime that implements audit at all. `none` exists so a profile can declaratively turn off an audit that came from a base profile (when `extends:` lands).

## Redaction

`redact` lists JSONPaths that should be replaced with `[REDACTED]` _in the audit record_, not in the actual request/response. This is so audit can be enabled even on calls that carry sensitive payloads:

```yaml
audit:
  - stage: request
    target: { sourceId: "internal-db" }
    sink: { type: webhook, url: "https://audit.example.com/calls" }
    redact:
      - "$.params.password"
      - "$.params.api_key"
      - "$.headers.authorization"
```

## Where audit runs

Audit runs as the **last** layer in each stage (after transforms, processors, and guardrails). This is deliberate:

* Audit sees the same shape the agent will see (or did see) — useful for forensics.
* Audit cannot influence the call, even by side-effect on shared state.

A failing audit MUST NOT abort the call. The runtime SHOULD log the failure but continue.

## Worked example

```yaml
audit:
  - stage: response
    target: { sourceId: "*", tags: ["destructive"] }
    sink:
      type: webhook
      url: "https://audit.example.com/destructive"
      headers: { authorization: "Bearer {{secrets.AUDIT_TOKEN}}" }
    include: { request: true, response: true, metadata: true }
    redact: ["$.body.token", "$..password"]
```

## What this maps to today

There is no audit hook in the current Civic MCP runtime; the closest analogues are partial structured-logging coverage and ad-hoc per-deployment webhooks. The fields in this section therefore describe the target shape rather than a 1:1 mapping from existing code.
