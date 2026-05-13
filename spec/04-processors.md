# 04 - Pre- and post-processors

A **processor** changes content (request body or response body) without changing the tool definition. Today these are the `response.transform` constraints (htmlToMarkdown, jsonToCSV, removeFields, abbreviate) implemented by the unified guardrail engine. Preprocessors are the symmetric concept for requests.

> Why processors and transforms are separate: a transform changes what the agent _sees_ (it's a `list`-stage concept). A processor changes what _flows through_ on a specific call (`request` or `response` stage). The agent does not see the processed/unprocessed distinction.

## Processor schema

```yaml
Processor:
  id: string?
  kind: enum                    # request | response
  target: ToolSelector          # which calls this applies to
  enabled: boolean?             # default true
  executionIndex: int?          # within-stage ordering; default 1000

  # Condition (optional). If present, processor only runs when this evaluates to true.
  when:
    schemaPath: string          # JSONPath into request or response body
    operator: enum              # equals, exists, includes, regex, etc.
    value: any?                 # operator-dependent

  # Action.
  action: enum                  # see action catalogue below
  actionPath: string?           # JSONPath of where the action applies (defaults to "$")
  actionParams: object?         # action-specific config
```

The shape mirrors the existing `ConstraintTemplate`. The key changes from today:

* `kind` is `request` or `response` (vs the existing `timing` field).
* The condition section uses `when:` for clarity (`schemaPath`/`operator`/`value` together).
* `action` no longer overlaps with `redact`/`reject` — those move to guardrails (_05 - Guardrails_).

## JSONPath dialect

Both `schemaPath` and `actionPath` use JSONPath, but with a small set of extensions over the standard. Guardrails (_05 - Guardrails_) use the same dialect.

| Construct | Source | Meaning |
| --- | --- | --- |
| Standard JSONPath (`$`, `.`, `[*]`, `[?(@.x==y)]`, …) | RFC 9535-ish | Baseline. |
| `^` (parent) | JSONPath Plus | Selects the parent of the current node. Lets `actionPath` reach a sibling of a `schemaPath` match. |
| `parse(text)` | Civic extension | Treats the string at the named field as nested JSON and continues the path inside it. Common case: tool responses that wrap structured data in `content[*].text`. |

### `actionPath` relative to `schemaPath`

`actionPath` may be either an absolute JSONPath (rooted at `$`) or a path **relative to each `schemaPath` match**. This matters when `schemaPath` matches inside an array: each match has its own implicit root, and `actionPath` applies once per match.

### Worked example

For a response shaped like:

```json
{
  "content": [
    { "text": "{\"user\": {\"login\": \"alice\"}}" },
    { "text": "{\"user\": {\"login\": \"bob\"}}" }
  ]
}
```

A constraint of `content[*].parse(text)[*].user.login` against allowed users `["alice"]`:

1. **Pre-parse.** `parse(text)` converts the JSON-in-string at `content[*].text` into objects.
2. **Query.** JSONPath walks `content[*].text[*].user.login` and finds `alice`, `bob`.
3. **Match.** `bob` at `$['content'][1]['text'][0]['user']['login']` violates the constraint.
4. **Navigate.** `actionPath: "^"` selects the parent — `$['content'][1]['text'][0]['user']` — so a redact action can replace the whole user object, not just the offending login.

## Action catalogue

This is the v1 action set. Runtimes MAY support additional actions via a registry; unknown actions MUST cause the runtime to refuse to start (fail-closed) so a profile that depends on a missing action does not silently degrade.

### Response actions

| Action | `actionParams` | Purpose |
| --- | --- | --- |
| `htmlToMarkdown` | `{ preserveImages?, preserveLinks?, stripStyles? }` | HTML → Markdown for token reduction. |
| `jsonToCSV` | `{ includeHeaders?, delimiter?, fields? }` | Tabular JSON → CSV. |
| `removeFields` | `{ fields: [string] }` | Strip listed fields (supports `*.field` wildcards). |
| `pickFields` | `{ fields: [string] }` | Inverse of removeFields — keep only listed fields. |
| `abbreviate` | `{ maxLength: number, strategy?: "smart"|"hard", suffix?: string }` | Truncate text. |
| `mapRename` | `{ rename: { oldKey: newKey } }` | Rename keys without changing values. |

### Request actions

| Action | `actionParams` | Purpose |
| --- | --- | --- |
| `setDefault` | `{ values: { path: value } }` | Fill in missing fields with defaults. (Not the same as `preset-param`; this only fills if absent.) |
| `coerce` | `{ type: "string"|"number"|"boolean"|"array" }` | Coerce a value at `actionPath` to the given type. |
| `enforceFormat` | `{ format: "email"|"uri"|"iso8601"|... }` | Reject if value doesn't parse to the format; otherwise normalise. (Borderline guardrail; here when the side-effect is normalisation.) |

The catalogue is small on purpose. Anything that _blocks_ belongs in guardrails; anything that _changes the tool definition_ belongs in transforms.

## Failure mode

Processors are best-effort. A failing processor MUST NOT abort the request/response — the runtime SHOULD log a warning and continue with the unprocessed payload.

This matches today's `ConstraintsAndPostProcessorsHook` behaviour: `response.transform` errors skip the transform; only `schema.constraint` failures abort.

If a profile genuinely needs a _blocking_ content rewrite, model it as a guardrail with `outcome: redact` (see _05 - Guardrails_).

## Worked examples

### Convert HTML to Markdown for a scraping source

```yaml
processors:
  - kind: response
    target: { sourceId: "puppeteer" }
    when: { schemaPath: "$.content", operator: "exists", value: true }
    action: htmlToMarkdown
    actionPath: "$.content"
    actionParams: { preserveImages: true, preserveLinks: true, stripStyles: true }
    executionIndex: 1000
```

### Pick only useful fields from a Jira issue

```yaml
processors:
  - kind: response
    target: { sourceId: "jira", toolName: "getJiraIssue" }
    action: pickFields
    actionPath: "$"
    actionParams:
      fields: ["key", "fields.summary", "fields.status.name", "fields.assignee.displayName"]
    executionIndex: 1100
```

### Default a region on every call to an AWS source

```yaml
processors:
  - kind: request
    target: { sourceId: "aws-mcp" }
    action: setDefault
    actionPath: "$"
    actionParams:
      values: { region: "us-east-1" }
```

## Where processors sit in the chain

See _09 - Execution chain_ for the canonical order. In short:

* `request` processors run **after** request-stage transforms but **before** request-stage guardrails — so a guardrail evaluates the body that will actually be dispatched.
* `response` processors run **before** response-stage guardrails — so a guardrail evaluates the body the agent will actually see.

Within a stage, multiple processors run in `executionIndex` order (lower first).

## What this maps to today

| Concept | Today |
| --- | --- |
| `Processor` (response) | `ConstraintTemplate` with `type: response.transform` |
| `Processor` (request) | Not yet implemented as a generic mechanism; today's request-side rewrites live in `ParameterHook`/`ProfileData`. The protocol unifies them. |
| `actionPath` | Same field name today (renamed from `redactPath`). |
| `actionParams` | Same field today. |
| Failure mode | `response.transform` failure = skip + warn (matches today). |
