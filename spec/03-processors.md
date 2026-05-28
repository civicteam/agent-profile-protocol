# 03 - Pre- and post-processors

A **processor** changes a request or response body on a specific call without
changing the tool definition. Processors are best-effort: their failure is
logged and swallowed, never aborts the call. Use a guardrail (_04 -
Guardrails_) when you need fail-stop behaviour.

## Processor schema

```yaml
Processor:
  stage: enum                 # request | response — which leg of the chain this attaches to
  target: string | [string]?  # which tools — see "Targeting"
  enabled: boolean?           # default true
  executionIndex: int?        # within-stage ordering; default 1000

  # Optional condition. If present, the processor only runs when this is true.
  when:
    schemaPath: string        # JSONPath into the request or response body
    operator: enum            # equals | exists | includes | regex | …
    value: any?               # operator-dependent

  # Action — the actual operation. See action catalogue below.
  action: enum                # htmlToMarkdown | pickFields | block | …
  actionPath: string?         # JSONPath of where the action applies; defaults to "$"
  actionParams: object?       # action-specific config
```

`stage:` is the discriminator, matching the shape used by guardrails (_04 -
Guardrails_). Transforms use `kind:` because they have distinct operation
kinds (`alias`, `clone`, `filter`, `preset-parameter`); processors and
guardrails attach at a stage of the chain, and the actual operation lives in
`action:` (processors) or `outcome:` (guardrails).

## Targeting

`target:` is the same shape as in transforms (see _02 - Tool transforms_):
bare string, list of strings, or omitted to mean "every tool in the matched
connection(s)".

Processors target by **current tool name** — the name the agent sees after
transforms have materialised. That means:

* A processor with `target: informTeam` runs even though the upstream tool
  is named `sendEmail`.
* A processor with `target: sendEmail` does **not** run if an alias has
  renamed the tool to `informTeam`; the load-time validator MUST reject the
  processor for naming a tool that no longer exists.
* A `clone A → B` past a processor that targets `A` produces a B that is
  **not** covered by the processor. The host SHOULD warn at load time when a
  clone's `target` is the target of a request- or response-stage processor.

There is no `target.sourceId` / `target.scope` form. The document's
`matches` block (_01 - Matching_) is the only connection-scope selector.

## JSONPath dialect

Both `schemaPath` and `actionPath` use JSONPath with the following
extensions over RFC 9535. Guardrails (_04 - Guardrails_) use the same
dialect.

| Construct | Source | Meaning |
| --- | --- | --- |
| Standard JSONPath (`$`, `.`, `[*]`, `[?(@.x==y)]`, …) | RFC 9535-ish | Baseline. |
| `^` (parent) | JSONPath Plus | Selects the parent of the current node. Lets `actionPath` reach a sibling of a `schemaPath` match. |
| `parse(text)` | Civic extension | Treats the string at the named field as nested JSON and continues the path inside it. Common case: tool responses that wrap structured data in `content[*].text`. |

### `actionPath` relative to `schemaPath`

`actionPath` may be either an absolute JSONPath (rooted at `$`) or a path
**relative to each `schemaPath` match**. When `schemaPath` matches inside an
array, each match has its own implicit root and `actionPath` applies once
per match.

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

A constraint of `content[*].parse(text)[*].user.login` against allowed users
`["alice"]`:

1. **Pre-parse.** `parse(text)` converts the JSON-in-string at
   `content[*].text` into objects.
2. **Query.** JSONPath walks `content[*].text[*].user.login` and finds
   `alice`, `bob`.
3. **Match.** `bob` violates the allowed-users constraint.
4. **Navigate.** `actionPath: "^"` selects the parent — the whole `user`
   object — so a redact-style processor can replace the offending subtree.

## Action catalogue

This is the v1 action set. Hosts MAY support additional actions via a
registry; unknown actions MUST cause the host to refuse to start (fail-closed)
so a profile that depends on a missing action does not silently degrade.

### Response actions

| Action | `actionParams` | Purpose |
| --- | --- | --- |
| `htmlToMarkdown` | `{ preserveImages?, preserveLinks?, stripStyles? }` | HTML → Markdown for token reduction. |
| `jsonToCSV` | `{ includeHeaders?, delimiter?, fields? }` | Tabular JSON → CSV. |
| `removeFields` | `{ fields: [string] }` | Strip listed fields (supports `*.field` wildcards). |
| `pickFields` | `{ fields: [string] }` | Inverse of `removeFields` — keep only listed fields. |
| `abbreviate` | `{ maxLength: number, strategy?: "smart"|"hard", suffix?: string }` | Truncate text. |
| `mapRename` | `{ rename: { oldKey: newKey } }` | Rename keys without changing values. |

### Request actions

| Action | `actionParams` | Purpose |
| --- | --- | --- |
| `setDefault` | `{ values: { path: value } }` | Fill in missing fields with defaults. (Distinct from `preset-parameter`, which always overrides and hides from the agent.) |
| `coerce` | `{ type: "string"|"number"|"boolean"|"array" }` | Coerce a value at `actionPath` to the given type. |
| `enforceFormat` | `{ format: "email"|"uri"|"iso8601"|… }` | Reject if value doesn't parse to the format; otherwise normalise. (Borderline guardrail; here when the side-effect is normalisation.) |
| `block` | _none_ | Refuse the call. Equivalent to a `block` guardrail with the same `when` — provided so processor pipelines can short-circuit without a separate guardrail entry. |

The catalogue is small on purpose. Anything that _changes the tool
definition_ belongs in transforms (_02 - Tool transforms_). Anything whose
failure must abort the call belongs in guardrails (_04 - Guardrails_).

> The `block` action above is a convenience: it lets a request processor
> deny the call inline when the condition fires. Use it for "if X then
> refuse" rules that don't need the `redact` / `require-approval` palette.
> For anything else, prefer a guardrail.

## Failure mode

Processors are best-effort. A failing processor MUST NOT abort the
request/response — the host SHOULD log a warning and continue with the
un-processed payload. The exception is `action: block`, which intentionally
aborts the call (with a clear error to the agent).

If a profile genuinely needs a _blocking_ content rewrite, model it as a
guardrail with `outcome: redact` (see _04 - Guardrails_).

## Worked examples

### Convert HTML to Markdown for a scraping connection

```yaml
processors:
  - stage: response
    when: { schemaPath: "$.content", operator: "exists", value: true }
    action: htmlToMarkdown
    actionPath: "$.content"
    actionParams: { preserveImages: true, preserveLinks: true, stripStyles: true }
    executionIndex: 1000
```

### Pick only useful fields from a Jira issue

```yaml
processors:
  - stage: response
    target: getJiraIssue
    action: pickFields
    actionPath: "$"
    actionParams:
      fields: ["key", "fields.summary", "fields.status.name", "fields.assignee.displayName"]
    executionIndex: 1100
```

### Block outbound mail to civic.com on the renamed tool

```yaml
processors:
  - stage: request
    target: informTeam
    when:
      schemaPath: $.to
      operator: regex
      value: '@civic\.com$'
    action: block
```

## Where processors sit in the chain

See _06 - Execution chain_. Summary:

* `request` processors run **after** request-stage transforms but **before**
  request-stage guardrails — so a guardrail evaluates the body that will
  actually be dispatched.
* `response` processors run **before** response-stage guardrails — so a
  guardrail evaluates the body the agent will actually see.

Within a stage, multiple processors run in `executionIndex` order (lower
first).
