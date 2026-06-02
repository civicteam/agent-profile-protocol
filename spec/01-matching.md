# 01 - Matching

A **profile** is a stream of YAML documents. Each document declares which
**connection(s)** it applies to and what transforms / processors / guardrails
should attach to those connections. This section defines the matching
mechanism, the application lifecycle, and how multiple documents compose when
they target the same connection.

## Document structure

A profile file is parsed as a YAML stream — documents separated by `---`. Every
document is independent: an unrelated config can be appended without touching
existing documents.

```yaml
apiVersion: civic.com/agent-profile/v1alpha1
matches:
  name: github
  version: "^0.1"

transforms: [...]
processors: [...]
guardrails: [...]
---
apiVersion: civic.com/agent-profile/v1alpha1
matches:
  name: atlassian
  version: "^0.1"

transforms: [...]
```

`type:` is omitted in both blocks — it defaults to `mcp`. See "`matches`
schema" below.

Required top-level fields per document:

| Field | Required? | Notes |
| --- | --- | --- |
| `apiVersion` | yes | Must appear in every document (the host parses each one independently). |
| `matches` | no | When omitted, the document applies to **every** connection the host has registered. |
| `transforms` | no | List of `kind: …` entries. |
| `processors` | no | Same. |
| `guardrails` | no | Same. |

A document with no `transforms`, `processors`, or `guardrails` is legal but a
no-op — useful as an assertion that a profile is aware of a connection without
modifying it.

## `matches` schema

`matches:` may be **either** a single matcher block or a list of matcher
blocks:

```yaml
# Single block — matches one connection family.
matches:
  name: string?            # connection name as advertised by the host (e.g. "github")
  type: enum?              # transport type; defaults to "mcp" when omitted (v1 only defines "mcp")
  version: string?         # SemVer range; matched against the connection's reported version

# List of blocks — matches any connection that satisfies ANY block.
matches:
  - name: github
  - name: gitlab
    version: "^17"
```

Within one block, every field that is present must match (**AND**). Across
blocks in a list, a connection matches if it satisfies **any** block (**OR**).
This is the only way to compose matchers — there is no negation, no
globbing, no `not`.

A field that is omitted is treated as a wildcard. `name` matching is literal
equality against the connection's host-assigned name; regex and globbing are
not in v1.

`type:` defaults to `mcp` when omitted (v1 only defines `mcp`, so this is
purely a boilerplate reduction).

`version` accepts the npm SemVer range grammar:

| Range | Meaning |
| --- | --- |
| `"0.1.4"` | Exact version. |
| `"^0.1"` | `>=0.1.0 <0.2.0` (compatible with `0.1.x`). |
| `"~0.1.4"` | `>=0.1.4 <0.2.0`. |
| `">=0.1.0 <1.0.0"` | Explicit range. |
| `"*"` or omitted | Any version (including connections that don't report one). |

A connection that does not report a version field is treated as matching only
the wildcard form. A profile that needs to opt in to unversioned connections
must omit `version` or set `version: "*"`.

## Application lifecycle

The host owns the set of MCP connections it has been configured with. The
profile describes overlays. The lifecycle of attaching a profile to those
connections is:

1. **Connections register.** The host loads its own config (out of scope of
   this protocol — usually a `.mcp.json` or equivalent) and instantiates one
   or more MCP connections. Each connection has at least `name` and `type:
   mcp`; most also report a `version` via the MCP `initialize` exchange.

2. **Profile parses.** The host loads the profile YAML stream and validates
   each document against its `apiVersion` schema. Parse errors fail loud —
   the profile does not partially apply.

3. **Per-document matching.** For each profile document **D**, the host
   evaluates `D.matches` against every registered connection. The result is
   the **bound set** for that document: zero or more connections to which
   `D.transforms`/`processors`/`guardrails` will attach.

   * One document MAY bind to several connections.
   * A document with no bound connections is logged at warn level and
     skipped. (See "No-match handling" below.)

4. **Per-connection composition.** For each connection **C**, the host
   collects every document whose bound set contains C, **in document order
   within the profile file**, and concatenates their `transforms`,
   `processors`, and `guardrails` lists. The result is C's effective
   overlay.

5. **Materialisation.** The host applies the effective overlay to C as
   described in _06 - Execution chain_:
   * On `tools/list`, transforms materialise the agent-facing tool list.
   * On `tools/call`, the request and response stages run the relevant
     processors and guardrails.

6. **Re-evaluation.** If the host's connection set changes (a connection is
   added, removed, or its reported version changes), steps 3–5 re-run for
   the affected connections.

## Composition: multiple documents per connection

When several profile documents bind to the same connection, their effects
**compose in document order**. The semantics are:

* `transforms`: concatenated. Earlier transforms feed later ones (e.g. a Doc
  A alias produces a name that a Doc B preset-parameter can target).
* `processors`: concatenated. They run in document order within their stage
  (list order within a document).
* `guardrails`: concatenated. They run in document order within their stage
  (list order within a document). The runtime evaluates them all even if an
  earlier one fires (so `block` always wins, but `redact`s stack).

There is no implicit precedence by `matches` specificity. A more specific
match (`name + version`) does **not** override a more general match (`name`
only). If a profile needs precedence, it should be expressed via document
order.

### Composition example

```yaml
# Doc 1: rename `sendEmail` on every MCP connection.
apiVersion: civic.com/agent-profile/v1alpha1
transforms:
  - kind: alias
    target: sendEmail
    to: informTeam

---
# Doc 2: preset the recipient on the Gmail connection specifically.
apiVersion: civic.com/agent-profile/v1alpha1
matches:
  name: gmail
transforms:
  - kind: preset-parameter
    target: informTeam
    params:
      - name: recipient
        value: team@civic.com
```

For the `gmail` connection both documents bind. The effective transform list
is `[alias sendEmail→informTeam, preset-parameter on informTeam]`, in that
order. By the time the preset-parameter is evaluated, the alias has already
renamed the tool, so `target: informTeam` resolves correctly.

For any non-gmail MCP connection, only Doc 1 binds; the rename happens but no
preset is applied.

## No-match handling

| Situation | Host behaviour |
| --- | --- |
| Document with `matches` block, zero connections match | Log a warning, skip the document. The profile is otherwise loaded. |
| Document with no `matches` block, host has zero connections | Same — warning + skip. |
| Document targets an unknown tool by name (e.g. a processor `target: foo` and no connection's tool list contains `foo`) | Startup error — the profile cannot be loaded. (See _03 - Processors_.) |
| Connection registers later (after the profile loaded) and now matches a previously-skipped document | The host MUST re-evaluate matching for that connection; the document attaches retroactively. |

The asymmetry between "no connection matched" (warn) and "no tool matched"
(error) is deliberate. The first happens routinely when a profile bundle
covers several environments; the second is almost always a typo.

## What `matches` is **not**

* It is not a query language. The only composition operator is **OR across a
  list of matcher blocks**; there is no negation, no globbing, no nesting.
  Each field within a block is a single literal (or a SemVer range for
  `version`).
* It does not match on connection identity (URL, host, credentials). Those
  are the host's concern; the protocol matches on declared metadata only.
* It does not pin a profile to a specific connection instance. Re-creating a
  connection with the same `name`/`type`/`version` rebinds the profile.

## Reserved fields

The following keys at the top level of a document are reserved and MUST NOT
be repurposed by an implementation:

`apiVersion`, `matches`, `transforms`, `processors`, `guardrails`,
`metadata` (reserved for v1.1 doc-level annotations such as authorship and
description).

Any other top-level key MUST cause the host to refuse to load the profile —
this keeps the file format closed for unambiguous evolution.
