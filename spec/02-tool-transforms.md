# 02 - Tool transforms

A **transform** modifies, adds, or removes a tool definition before it
reaches the agent. Transforms are the only layer that changes what the agent
sees on `tools/list`.

## Canonical envelope

Every transform is a tagged object with a `kind:` discriminator:

```yaml
Transform:
  kind: enum             # filter | alias | clone | preset-parameter
  target: string | [string]?   # which tool(s) — see "Targeting"
  to: string?            # new name (only for alias / clone)
  description: string?   # new description (only for alias / clone)
  # …kind-specific fields…
  enabled: boolean?      # default true
```

There is no `id`, no `spec:` wrapper, no nested-object form. Other syntaxes
that appeared in earlier drafts (single-key wrappers, kind-keyed maps) are
**not** part of the spec.

## Targeting

`target:` is the only way to point a transform at a tool. It accepts:

| Form | Meaning |
| --- | --- |
| `target: foo` | One tool named `foo` in the matched connection(s). |
| `target: [foo, bar]` | Several tools, by exact name. |
| _omitted_ | Every tool in the matched connection(s). For `alias` and `clone` this is **not allowed** (those require a single source name); for `filter` and `preset-parameter` it means "act on all matched tools". |

`target` matches by the **current** tool name at the point the transform
runs. If an earlier transform aliased `sendEmail` to `informTeam`, a later
`preset-parameter` should use `target: informTeam`.

There is no `target: { sourceId: …, scope: … }` form — those were the old
multi-source / skill keys and are gone. The document's `matches` block (see
_01 - Matching_) is the only connection-scope mechanism.

## Transform kinds

### `filter`

Removes tools from the agent-facing list.

```yaml
- kind: filter
  mode: exclude          # exclude | include
  target: [executeSql, dropTable]
```

* `mode: exclude` — listed tools are removed. Calls to them produce
  `MethodNotFound`.
* `mode: include` — only listed tools are kept; everything else is removed
  (whitelist).

`target` is required for `filter`. A `filter` with no `target` is a startup
error (either accidental "exclude nothing" or accidental "remove
everything").

A filtered tool is still available as a clone source — `filter` only affects
what the agent sees on `tools/list`, not what later transforms can build on.

### `alias`

Renames a tool, optionally changing its description.

```yaml
- kind: alias
  target: send_email     # single string only
  to: email_team
  description: "Sends an email to the engineering team."
```

* `target` and `to` are both required. `target` is the existing name; `to`
  is the new name.
* `description` is optional. Omitting it preserves the original description.
* When `target == to`, the alias is description-only (it changes the
  description without renaming). This is the recommended replacement for
  the old `describe` transform.

Aliases are **terminal** — once a tool has been aliased away, the old name
no longer exists in the materialised list, and later transforms that
reference it by name will fail. The `to` name is the only handle.

A call that arrives for the pre-alias name (stale agent state, or a guess)
MUST be rejected with `MethodNotFound` — the old name is not on the
materialised list, so it is not a valid call target. See "Dispatch surface".

The `to` name is a valid `target` for later `alias`, `preset-parameter`, and
`filter` transforms (it is the live handle). It is **not** a valid `clone`
source: `clone` copies a schema-bearing tool, and an alias only renames one
that another transform may already have consumed. See the `clone` chain table
below.

#### Chaining aliases

Because `target` matches the *current* name and `alias` is terminal for the
*old* name, aliases chain in composition order (document order across
documents, list order within a document). The second alias targets the first
alias's `to` name — the live handle, not the dead original.

| Sequence (only `a` exists initially) | Allowed? | Why |
| --- | --- | --- |
| `alias a→b`, `alias b→c` | yes | the second alias targets `b`, which exists at that point; the net result is `a` materialised as `c`. |
| `alias b→c`, `alias a→b` | no | the first entry's `target: b` resolves to no tool → startup error. |

Order is load-bearing: a chain must be written so each alias's `target`
already exists at its position. The runtime rejects the wrong order at load
time — either a missing `target`, or a `to` that collides with a
not-yet-renamed tool.

### `clone`

Produces a new tool from an existing one. Both tools are visible to the
agent (unless the original is filtered out separately).

```yaml
- kind: clone
  target: executeSql     # single string only
  to: getMySessions
  description: "Returns the current user's recent sessions."
```

* `target` is the existing tool (may be a tool that has been removed by a
  later `filter` — clones run before `filter` materialisation; see _06 -
  Execution chain_).
* `to` is the new tool's name. It MUST be unique in the materialised list.
* The clone inherits the source's `inputSchema` and `outputSchema`. Use a
  separate `preset-parameter` transform to fix or hide parameters on the
  clone.

Clones produce **real** new names that can themselves be the `target` of
later `clone`, `preset-parameter`, or `filter` operations. This is the
key difference from `alias` (which is terminal):

| Sequence | Allowed? | Why |
| --- | --- | --- |
| `clone A → B`, `clone B → C`, `alias A → D` | yes | Each clone introduces a real new name; aliasing the original `A` last is fine because both clones already referenced `A` (resolved at their own application time). |
| `clone A → B`, `alias B → C`, `clone C → D` | no | `C` is the result of an alias, so it is not a valid **`clone` source** (a clone needs a schema-bearing tool to copy). `C` is still a valid `target` for `alias`, `preset-parameter`, and `filter`. The runtime MUST reject the `clone C → D` at load time. |

### `preset-parameter`

Fixes or supplies parameter values that the agent does not need to specify.
The presets are merged into the request body before dispatch; presetted
parameters are hidden from the agent's view of `inputSchema` unless
`hideFromInputSchema: false` is set explicitly.

```yaml
- kind: preset-parameter
  target: getMySessions
  params:
    - name: query
      value: "SELECT * FROM sessions WHERE user_id = ?"
    - name: limit
      value: 50
```

Each entry under `params` has:

| Field | Notes |
| --- | --- |
| `name` | Required. The parameter name in the tool's `inputSchema`. |
| `value` | Required for non-interpolated presets. Any JSON-compatible value. |
| `type` | Optional. Only `interpolate` is defined in v1 (see below). When omitted the preset is a plain value. |
| `replacement` | Required when `type: interpolate`. Map keyed by slot name; each value is a JSON Schema for that slot. |
| `hideFromInputSchema` | Default `true`. When `false`, the agent still sees the parameter and may override the preset. |

A `preset-parameter` with no `target:` applies to every tool in the matched
connection(s). For each tool:

* If the tool's input schema declares the named parameter, the preset takes
  effect.
* If the tool does **not** declare the parameter, the preset is **silently
  skipped** for that tool. The host SHOULD log this at debug level.
* Add `strict: true` to the preset envelope to turn skips into startup
  errors instead.

#### Interpolation

`type: interpolate` lets a preset expose **named slots** to the agent in
place of an opaque value. The agent fills the slots; the host substitutes
them into the template before dispatch.

```yaml
- kind: preset-parameter
  target: informTeam
  params:
    - name: subject
      type: interpolate
      value: "Thanks for contacting me, {name}!"
      replacement:
        name:
          type: string
          default: "buddy"
```

The semantics:

* `value` is a template string. `{slot}` markers reference entries in
  `replacement`.
* `replacement` is a map: key = slot name, value = a JSON Schema for the
  slot. The schema is exposed verbatim to the agent as if the slot were a
  parameter on the tool.
* `name` (the param's `name`) and the slot keys live in different namespaces
  — there is no collision.
* Missing required slots cause a `request`-stage error before dispatch.

The replacement map is the entire interpolation surface — there is no
positional form, no nested slot, no escape syntax. Anything more elaborate
should be a processor, not a preset.

## Ordering and composition

Ordering has no author-controllable knob. It is fixed by two rules:

1. **Across kinds** — the stage pipeline in _06 - Execution chain_ is
   authoritative. On `list` the order is `clone → filter → alias →
   preset-parameter` (clone runs before filter so a tool can be cloned and the
   original hidden in the same overlay); `request` and `response` mirror this.
   This order is a property of what each kind does, not a choice.
2. **Within a kind** — transforms run in composition order: document order
   across documents, then list order within a document.

Across documents that bind to the same connection, transforms compose in
**document order**. Document A's transforms are applied first, producing an
intermediate tool list; Document B's transforms then run against that
intermediate list. A later document cannot reorder its transforms ahead of an
earlier document's within the same kind — document order is the only
precedence (_06_).

## Failure handling

| Situation | Behaviour |
| --- | --- |
| `target` resolves to no tool (literal name not present at apply time) | Startup error — the profile cannot be loaded. |
| `to` collides with an existing tool name | Startup error. |
| Chain rule violated (e.g. cloning an aliased tool) | Startup error. |
| `preset-parameter` lists a param the tool does not declare, `strict: false` | Skip + debug log. |
| `preset-parameter` lists a param the tool does not declare, `strict: true` | Startup error. |
| Filter `target` includes a name that does not exist | Startup error. |

Transforms are static configuration. There is no per-call failure for
transforms — every failure mode is at profile load time.

## Dispatch surface

The materialised `tools/list` is the authoritative set of callable names. A
`tools/call` for any name not present in it — whether removed by `filter`,
renamed away by `alias`, or never defined upstream — MUST be rejected with
`MethodNotFound`. This is a property of dispatch, not a transform failure:
transforms themselves still fail only at load time (see the table above).

In particular, an `alias` that renames a tool to restrict it (e.g. hiding
`executeSql` behind a narrower name) is only effective if the pre-alias name
is unreachable. A host that lets the original upstream name pass through
unchanged does not conform.
