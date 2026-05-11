# 03 - Tool transforms

A **transform** modifies, adds, or removes a tool definition before it reaches the agent. Today this is what the hub's `FilterHook`, `AliasHook`, `CloneResultHook`, `ParameterHook`, and `CustomDescriptionHook` do, driven by the `Hook_*` categories in `ProfileData`.

> Naming note: this used to be called "profile data". That label is misleading — "profile data" is also used for free-form per-profile memory. In this spec, the things that change tool definitions are **transforms**.

## When transforms run

| Stage | Effect |
| --- | --- |
| `list` | Emitted when materialising the tool list. Affects what the agent sees. |
| `request` | Applied to the request body before dispatch (e.g. preset params). |
| `response` | Applied to the response body after dispatch (e.g. clone). |

A single transform can declare effects at multiple stages — see `clone` below.

## Transform kinds

The `kind` discriminator selects one of:

| `kind` | Stage(s) | Description |
| --- | --- | --- |
| `filter` | `list` | Disable specific tools or whitelist a subset. |
| `alias` | `list`, `request` | Rename a tool (and optionally adjust its description). The original `name` is preserved in `meta.upstreamName` so the runtime can route calls. |
| `clone` | `list`, `request`, `response` | Produce a derived tool from an existing one with overrides (description, preset params, response-shape edits). |
| `preset-param` | `list`, `request` | Inject or fix parameter values so the agent never sees them in the input schema. |
| `describe` | `list` | Override or augment a tool's description. |

Each kind has its own payload, but they share a common envelope:

```yaml
Transform:
  id: string?                  # optional stable id for ordering / referencing
  kind: enum                   # required
  target: ToolSelector         # required; what tool(s) this applies to
  enabled: boolean?            # default true
  executionIndex: int?         # within-stage ordering; default 0
  spec: object                 # kind-specific payload
```

## `ToolSelector`

```yaml
ToolSelector:
  sourceId: string             # required; "*" matches all
  toolName: string?            # exact name; default "*"
  tags: [string]?              # match tools with all listed tags (in meta.tags)
```

Selectors match the tool **as it appears at the time the transform applies**. Stage matters:

* A `list`-stage transform matches tools at materialisation. Earlier transforms feed later ones — if `alias` runs before `describe`, the `describe` selector should use the new name.
* A `request`-stage transform matches by the renamed name (since that is what the runtime received from the agent).

Within a stage, transforms run in `executionIndex` order.

## `filter`

```yaml
- kind: filter
  target: { sourceId: "github" }
  spec:
    mode: blacklist        # or whitelist
    tools: ["delete_repo", "transfer_repo"]
```

Behaviour:

* `blacklist`: tools in `tools` are removed from `tools/list`. Calls to them produce an error (`MethodNotFound`).
* `whitelist`: only tools in `tools` are kept; everything else is removed.

Filter is the only transform that **removes** definitions. (Block-via-guardrail is different: it allows the tool to appear in the list and only blocks at call time, with a custom error.)

## `alias`

```yaml
- kind: alias
  target: { sourceId: "gmail", toolName: "send_email" }
  spec:
    name: "email_team"
    description: "Sends an email to the configured team distribution list."
```

Behaviour:

* `list`-stage: replaces the tool's name (and optionally description). Sets `meta.upstreamName` to the original name.
* `request`-stage: rewrites incoming `tools/call.params.name` from `email_team` back to `send_email` before dispatch.

## `clone`

```yaml
- kind: clone
  target: { sourceId: "github", toolName: "create_issue" }
  spec:
    name: "create_bug_report"
    description: "Files a bug against the engineering project."
    preset:                   # request-stage: shallow-merged into params before dispatch
      project: "engineering"
      labels: ["bug"]
    responseShape:            # response-stage: optional shape edit
      pick: ["id", "url"]
```

Behaviour:

* `list`-stage: emits a _new_ tool named `create_bug_report` while leaving the original `create_issue` visible (unless filtered).
* `request`-stage: when `create_bug_report` is called, the runtime merges `preset` into `params`, then routes to the upstream `create_issue`.
* `response`-stage (optional): trims the response to the picked fields before returning.

`clone` is the way to produce **multiple narrowed views** of one upstream tool — the use case from the call where one underlying tool is exposed both as `email_team` (preset domain) and as `email_external` (different preset).

## `preset-param`

```yaml
- kind: preset-param
  target: { sourceId: "notion", toolName: "create_page" }
  spec:
    values:
      database_id: "abc123"
    hideFromInputSchema: true
```

Behaviour:

* `list`-stage: when `hideFromInputSchema: true`, removes the listed properties from the agent-visible `inputSchema` (and from `required`). The agent cannot specify them.
* `request`-stage: merges `values` into the request before dispatch.

`preset-param` is a narrower form of `clone` — same source, same name, just preset values. Use `clone` when you also need a different name or response shape; use `preset-param` when you only need to fix some inputs.

## `describe`

```yaml
- kind: describe
  target: { sourceId: "*", toolName: "*", tags: ["destructive"] }
  spec:
    descriptionPrefix: "[Requires approval] "
    descriptionAppend: "

Note: this tool will trigger an approval flow."
```

Behaviour:

* `list`-stage only. Adjusts the description; leaves the rest of the tool untouched.

## Ordering and composition

Within a stage, all transforms with the same selector match run in `executionIndex` order. A few rules:

1. `filter` runs **first** within `list` (so later transforms don't bother with definitions that won't be exposed).
2. `alias` runs **before** `clone` and `describe` (so they can target the new name if they want).
3. `clone` runs after `alias`. Inside `clone`, the `target` should refer to the _current_ name (after alias).
4. `preset-param` and `describe` run last in `list`.

This is a recommended convention — the protocol does not enforce these category-level orderings beyond `executionIndex`. A runtime MAY assign default `executionIndex` values matching this convention when materialising from imports.

## What this maps to today

| Transform | Today's hook(s) | Today's `ProfileData` category |
| --- | --- | --- |
| `filter` | `FilterHook` | `Hook_Filter` (`{enabled: [...]}` / `{disabled: [...]}`) |
| `alias` | `AliasHook` | (in-code aliasing today; mostly used internally) |
| `clone` | `CloneResultHook` + `ParameterHook` | `Hook_Clone` |
| `preset-param` | `ParameterHook` | `Hook_Parameter` |
| `describe` | `CustomDescriptionHook` | `Hook_CustomDescription` |

A round-trip migration that converts these `ProfileData` rows to `transforms` entries and back is the first concrete consumer of this spec.
