# 02 - Sources and multiplexer

A `Source` is an upstream supplier of tools. The hub combines all sources in a profile (plus any active skills) into one tool surface visible to the agent. That combination is the **multiplexer**.

## Source schema

```yaml
Source:
  id: string                    # required, unique within profile
  kind: enum                    # required; see below
  name: string?                 # display name
  description: string?          # display description
  transport: TransportConfig    # required; depends on kind
  tools: [Tool]?                # optional inline tools (used when source can't be introspected)
  resources: [Resource]?        # optional inline resources (MCP)
  resourceTemplates: [ResourceTemplate]?  # optional inline templates (MCP)
  auth: AuthRef?                # optional reference to credentials/<id> (see 07-credentials)
  enabled: boolean?             # default true
  prefix: string?               # tool-name prefix; defaults to id
```

### `kind` values (v1)

| `kind` | Transport |
| --- | --- |
| `mcp.http` | MCP over HTTP / streamable HTTP. |
| `mcp.sse` | MCP over SSE. |
| `mcp.stdio` | MCP over stdio (typically containerised). |
| `mcp.hosted` | A hosted MCP endpoint the runtime is responsible for deploying (e.g. via the Civic deployment subsystem). The transport sub-config carries deployment image/env/volumes. |

Reserved for later (not in v1):

* `exec` — wraps a shell-callable program. See _01 - Tool definition_ for inline tools.
* `openapi` — wraps an OpenAPI document.

### `TransportConfig` shapes

Sketches; full schemas live alongside the per-kind reference once stable.

```yaml
# mcp.http / mcp.sse
transport:
  type: http     # or sse
  url: "https://example.com/mcp"
  headers:       # static; for templated headers see 07-credentials
    user-agent: "civic-hub/1.0"

# mcp.stdio
transport:
  type: stdio
  command: ["node", "/srv/foo/server.js"]
  cwd: "/srv/foo"
  env: { LOG_LEVEL: "info" }

# mcp.hosted (deployment-managed)
transport:
  type: hosted
  image: "ghcr.io/civicteam/example-mcp:1.2.3"
  command: ["node", "server.js"]
  env: { PORT: "8080" }
  volumes: []
  configFiles: []   # see 07-credentials for credential-derived files
```

## Tool surface composition

For a profile with sources `A` and `B`, the agent's `tools/list` is:

```
list = ⋃ for each enabled source S:
         for each tool T in materialise(S):
           emit { name: prefix(S, T.name), inputSchema: T.inputSchema, ..., meta: { sourceId: S.id, ... } }
```

Where `materialise(S)` = the result of applying any `transforms` (_03 - Tool transforms_) targeting `S`.

### Prefixing rules

By default `prefix(S, T.name) = "${S.id}-${T.name}"` — same as today's hub. Two opt-outs:

1. `Source.prefix: ""` — emit tool names verbatim. Profile is responsible for ensuring no collisions.
2. `Source.prefix: "<custom>"` — use a custom prefix.

When two sources would collide on a name, the runtime MUST refuse to start and surface the conflict. The protocol does not prescribe a tie-breaking order; collisions are operator error.

### Resources and resource templates

Same model. Tool prefix becomes resource URI prefix. Default prefix for resources is `civic-com-${sourceId}/`, matching today's hub.

## The multiplexer

The multiplexer is **not** a separately configurable layer in the protocol — it's the _behaviour_ of a runtime that owns a set of sources plus the active skills. The protocol declares the inputs (sources, skills) and the runtime is responsible for combining them.

What the multiplexer guarantees:

* One MCP `initialize` exchange exposes all materialised tools, resources, and resource templates from all enabled sources and active skills.
* Tool calls are routed to the originating source by `meta.sourceId` (set during materialisation).
* Skills, when loaded, register additional tools under their own scope without overwriting base-scope tools (see _08 - Skills_).

What the multiplexer does **not** do:

* Bridge MCP-version mismatches. The runtime SHOULD assume sources speak the same MCP version and surface clear errors when they don't.
* Deduplicate tools across sources. Naming collisions are operator errors; see above.
* Cache `tools/list`. That's a runtime concern.

## Worked example

```yaml
sources:
  - id: gmail
    kind: mcp.hosted
    transport:
      type: hosted
      image: "ghcr.io/civicteam/gmail-mcp:1.0.0"
    auth: { ref: "credentials/gmail-oauth" }

  - id: github
    kind: mcp.http
    transport:
      type: http
      url: "https://api.githubcopilot.com/mcp"
    auth: { ref: "credentials/github-pat" }
```

The agent sees a flat list:

```
gmail-send_email
gmail-search_emails
github-create_issue
github-list_pulls
...
```

Each entry carries `meta.sourceId` so downstream layers (transforms, processors, guardrails) can route correctly.

## Sources and credentials

The `auth: { ref: "credentials/..." }` field is the _only_ place sources reference credentials. Everything else about credential satisfaction, persistence, and injection is in _07 - Credentials_. This keeps source records portable: a profile that doesn't need credential management at all (e.g. a public read-only HTTP MCP) just omits `auth`.
