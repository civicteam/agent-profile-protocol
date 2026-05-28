# 07 - Examples

A worked example demonstrating the full surface of the v1 protocol: matching,
all four transform kinds, processors, the interaction between cloning and
processors, and both credential persistence strategies. See _01 - Matching_
for the application lifecycle, _02 - Tool transforms_ for the canonical
envelope, _03 - Pre- and post-processors_ and _04 - Guardrails_ for the
per-kind details, and _05 - Credentials_ for the persistence strategies
illustrated below.

## Model in one paragraph

A profile file is a sequence of YAML documents separated by `---`. Each document
declares an `apiVersion`, an optional `matches` block that selects one or more
connections by metadata (`name`, `type`, `version`), and any combination of
`transforms`, `processors`, and `guardrails` to apply to the matched connections.
Documents that omit `matches` apply to **every** matched connection in the host
app's config. Multiple documents can target the same connection — their effects
compose in document order.

## Canonical syntax

* Each entry is a tagged object. **Transforms** use `kind:` (`kind: alias`,
  `kind: clone`, `kind: filter`, `kind: preset-parameter`). **Processors and
  guardrails** use `stage:` (`stage: request`, `stage: response`, and
  `stage: list` for guardrails) — they attach at a stage of the chain, and
  the actual operation lives in `action:` (processors) or `outcome:`
  (guardrails). Other syntaxes (single-key wrappers, kind-keyed maps) are
  **not** part of the spec.
* `target:` is the **only** way to point at a tool. It accepts:
  * a bare string — one tool;
  * a list of strings — several tools;
  * omitted — every tool in the matched connection(s).
* Operations that introduce a new tool name (`alias`, `clone`) take `target:`
  for the source tool and `to:` for the new name. `target:` for these kinds
  must be a single string (you cannot alias or clone several tools to one
  name).
* Version matchers use SemVer range strings (`"^0.1"`, `">=0.1.0 <1.0.0"`).
* Regex conditions use `operator: regex` with a plain string value (no `/…/`
  delimiters).
* There is no `describe` transform. To change a tool's description without
  renaming it, use `alias` with `target == to` and a `description` field.

## Worked example

A four-document profile covering matching, every transform kind, processors, and
the interaction between cloning and processors.

```yaml
# ─── Doc 1: GitHub passthrough + file-backed credentials ─────────────────────
# Matches the host's GitHub MCP connection. No transforms — but a `credentials:`
# block tells the host to persist the GitHub OAuth blob to a local file.
# Suitable for self-hosted single-tenant setups.
apiVersion: civic.com/agent-profile/v1alpha1
matches:
  name: github
  version: "^0.1"
  # `type:` omitted — defaults to `mcp`.

credentials:
  kind: file
  path: "~/.civic/creds/github.json"
  mode: 0600

---
# ─── Doc 2: Atlassian — preset params + description-only alias ────────────────
apiVersion: civic.com/agent-profile/v1alpha1
matches:
  name: atlassian
  version: "^0.1"

transforms:
  # Preset projectName on two named tools.
  - kind: preset-parameter
    target: [getIssue, createIssue]
    params:
      - name: projectName
        value: NEXUS

  # Preset projectName on EVERY tool in the connection that takes one.
  # `target` is omitted → all matched tools. Tools that don't declare a
  # `projectName` parameter are silently skipped. Set `strict: true` to
  # turn that into a startup error instead.
  - kind: preset-parameter
    params:
      - name: projectName
        value: NEXUS

  - kind: preset-parameter
    params:
      - name: cloudId
        value: "12345"

  # Change a tool's agent-facing description without renaming it: alias to
  # itself and supply a description. (There is no separate `describe` kind —
  # alias already covers both "rename" and "re-describe".)
  - kind: alias
    target: getProject
    to: getProject
    description: Get my Atlassian projects

---
# ─── Doc 3: Cross-connection rules (no `matches`) ─────────────────────────────
# With no `matches` block this document applies to every MCP connection the
# host config exposes. Useful for cross-cutting rules like "rename sendEmail
# everywhere" or "block outbound mail to .org domains regardless of source".
apiVersion: civic.com/agent-profile/v1alpha1

transforms:
  # Rename sendEmail → informTeam wherever it appears.
  - kind: alias
    target: sendEmail
    to: informTeam

  # Produce a second, narrower copy of the original tool. Clones reference
  # the upstream tool name (`sendEmail`), not any alias of it — aliases are
  # terminal (see "Clone / alias chaining rules" below).
  - kind: clone
    target: sendEmail
    to: notifyMe

  # Preset the recipient on the renamed tool.
  - kind: preset-parameter
    target: informTeam
    params:
      - name: recipient
        value: daniel@civic.com

  # Interpolated preset — the agent now sees a single string slot `name`
  # (typed via the `replacement` JSON Schema), and the runtime substitutes
  # it into the template before dispatch.
  - kind: preset-parameter
    target: informTeam
    params:
      - name: subject
        type: interpolate
        value: "Thanks for contacting me, {name}!"
        replacement:
          # Map keyed by slot name. Each value is plain JSON Schema for the slot.
          name:
            type: string
            default: buddy

processors:
  # Block outbound mail to civic.com addresses on the renamed tool.
  - stage: request
    target: informTeam
    when:
      schemaPath: $.to
      operator: regex
      value: '@civic\.com$'
    action: block

  # Same rule, applied to two tools at once. Note the FOOTGUN below.
  - stage: request
    target: [informTeam, draftEmail]
    when:
      schemaPath: $.to
      operator: regex
      value: '\.org$'
    action: block

# ─── Notes about Doc 3 ───────────────────────────────────────────────────────
# * The clone `sendEmail → notifyMe` is NOT covered by the two processors
#   above, because processors match by current tool name and `notifyMe` is
#   not in either target list. Cloning a tool past a processor effectively
#   bypasses the processor. A conformant runtime SHOULD emit a warning when
#   a clone's `target` is also the `target` of a processor.
# * If `sendEmail` had been removed (e.g. via a `filter` transform) before
#   this document was applied, the processors here would fail to load with
#   a parse error — processors targeting an unknown name are rejected at
#   startup, not at call time.

---
# ─── Doc 4: Postgres — filter + clones, AuthZ-API credentials ─────────────────
# Same connection, but persistence is delegated to a Civic-AuthZ service.
# A profile can move from file-backed to authz-api-backed by editing only
# this block — every transform/processor/guardrail stays as it was.
apiVersion: civic.com/agent-profile/v1alpha1
matches:
  name: postgres

credentials:
  kind: authz-api
  baseUrl: "https://app.civic.com/authz/api"

transforms:
  # Hide the raw SQL escape hatch from the agent.
  - kind: filter
    mode: exclude
    target: [executeSql]

  # Two narrowed views of the (now hidden) executeSql. Clones may reference
  # tools that were filtered out — the filter only affects what the agent
  # sees, not what later transforms can build on.
  - kind: clone
    target: executeSql
    to: getMySessions

  - kind: clone
    target: executeSql
    to: countUsers
    description: Get the current user count in the database

  - kind: preset-parameter
    target: getMySessions
    params:
      - name: query
        type: interpolate
        value: "SELECT * FROM sessions WHERE user_id = {user_id}"
        replacement:
          user_id:
            type: number

  - kind: preset-parameter
    target: countUsers
    params:
      - name: query
        value: "SELECT COUNT(*) FROM users"
```

## Clone / alias chaining rules

Aliases are **terminal**: once a tool has been aliased away, the old name no
longer exists in the materialised list, and the alias target is not itself a
base for further clones. Clones, by contrast, produce a real new name that can
be cloned or aliased again.

| Sequence | Allowed? | Why |
| --- | --- | --- |
| `clone A → B`, `clone B → C`, `alias A → D` | yes | Each clone introduces a real new name; aliasing the original `A` at the end is fine. |
| `clone A → B`, `alias B → C`, `clone C → D` | no | `C` is the result of an alias, not a tool that can be cloned. Aliases rename; they don't duplicate. |

A linter for the profile MUST reject the second form at load time.

## Targetless `preset-parameter` semantics

A `preset-parameter` with no `target:` applies to every tool in the matched
connection(s). For each tool:

* If the tool's input schema declares the named parameter, the preset takes
  effect (the parameter is hidden from the agent and injected on dispatch).
* If the tool does **not** declare the parameter, the preset is **silently
  skipped** for that tool. The runtime SHOULD log this at debug level.
* Set `strict: true` on the preset to turn skips into startup errors instead.

## Why processors target by name

Processors and guardrails match the **current** tool name — what the agent
sees after transforms. This keeps the rule files reading naturally ("block
calls to `informTeam` …" rather than "block calls to whatever-tool-was-once-
called-sendEmail"), at the cost of letting clones slip past rules whose
target list wasn't updated. The runtime warning described in Doc 3 is the
recommended mitigation.
