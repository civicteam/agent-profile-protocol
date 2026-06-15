# 05 - Credentials

A profile MAY declare a **credentials strategy** that tells the host how to
**persist and retrieve** the credentials a connection needs. The protocol
intentionally stops at persistence/retrieval — it does **not** describe the
auth flow (OAuth dance, MFA, refresh scheduling, etc.). Those remain host
concerns; this section just declares where the credential blob lives.

## Where credentials live in a document

`credentials:` is a top-level field of a document, alongside `transforms`,
`processors`, and `guardrails`. When present, it applies to every connection
the document's `matches` block accepts (_01 - Matching_).

```yaml
version: 0.1.0
matches:
  name: github

credentials:
  kind: file
  path: "~/.civic/creds/github.json"

transforms: []
```

A document with no `credentials:` block makes no assertion about
persistence; the host falls back to whatever default it would otherwise use
(e.g. an environment-defined backing store). A document with `credentials:`
**replaces** that default for the matched connections.

## Schema

```yaml
Credentials:
  kind: enum                  # file | authz-api
  # …kind-specific fields…
```

`kind:` is the discriminator, matching the rest of the protocol. Hosts MUST
refuse to load profiles that declare an unknown `kind` (fail-closed).

Exactly one credentials block per document. Chains and fallbacks (e.g.
"try file first, then authz-api") are a v1.1 candidate; v1 keeps the
strategy single.

## Composition across documents

When multiple documents bind to the same connection, only **one** of them
may declare `credentials:` for that connection. If two binding documents
both declare `credentials:`, the host MUST refuse to load the profile —
this is unlike the `transforms`/`processors`/`guardrails` lists, which
concatenate. Persistence is a non-list concern.

If the conflicting documents need to coexist, the conflicting `credentials:`
block should be moved into a document with a more specific `matches:` (or
removed from the more general one).

## `kind: file`

Persists credentials as a single file on the host's filesystem.

```yaml
credentials:
  kind: file
  path: string               # absolute path, ~ expansion allowed
  format: enum?              # "json" (default) | "yaml"
  mode: int?                 # POSIX file permissions; default 0600
```

* `path` interpolates `{name}` against the matched connection's name,
  enabling one document with a list-form `matches:` to point at several
  connections without duplicating the block:

  ```yaml
  matches:
    - name: github
    - name: gitlab

  credentials:
    kind: file
    path: "~/.civic/creds/{name}.json"
  ```

* `format` and `mode` are advisory hints to the host. A host that always
  writes JSON at `0600` MAY ignore these; a stricter host MUST honour them.
* The file's **content** is not specified by the protocol — it is whatever
  the host writes when it persists the credential. For OAuth2 the host will
  typically write `{ access_token, refresh_token?, expires_at? }`; for an
  API key, the raw string. The profile does not constrain this shape.

The file backend is the right default for self-hosted setups, smoke
testing, and any environment without an external credential service.

## `kind: authz-api`

Delegates persistence to a Civic-AuthZ-shaped HTTP service. The host issues
authorization _jobs_ against the service, awaits completion, and retrieves
the resulting token via the service's REST endpoints.

```yaml
credentials:
  kind: authz-api
  baseUrl: string            # base URL of the AuthZ API
  serverId: string?          # override the identifier sent to AuthZ; defaults to the connection name
```

The host is expected to talk to the following endpoints (matching the
[Civic MCP AuthZ API OpenAPI](../civic-mcp/apps/authz/api/openapi.yaml)):

| Endpoint | Used for |
| --- | --- |
| `POST {baseUrl}/user/jobs` | Create a new authorization job. The host supplies the connection identity (`serverId`) and any auth-requirement metadata it has discovered locally. |
| `GET {baseUrl}/user/jobs/{id}` | Poll job status (`pending` / `completed` / `expired` / `cancelled`). |
| `GET {baseUrl}/user/jobs/{id}/token` | Retrieve the resolved credential once the job is `completed`. |
| `GET {baseUrl}/user/authorizations` | List existing authorizations (for reusing a still-valid grant). |
| `DELETE {baseUrl}/user/authorizations/{id}` | Revoke. |

The protocol does **not** prescribe how the host authenticates **to** the
AuthZ service (bearer JWT, network-level trust, basic auth on admin
endpoints, etc.). That's a host-deployment concern.

### Why this strategy exists in v1

This is the strategy Civic MCP itself uses today. Carrying it in the spec
means a profile can be moved between a self-hosted file-backed deployment
and a Civic-AuthZ-backed deployment by changing only the `credentials:`
block — every other layer stays identical.

## What this section is **not**

* **Not an auth-flow specification.** Whether the host runs an OAuth2 PKCE
  flow, prompts the user for an API key, or pulls a token from a hardware
  module — out of scope. The profile only declares where the resulting
  blob lives.
* **Not a permissions model.** Per-tool authorization (e.g. "user must have
  scope X to call tool Y") belongs in guardrails (_04 - Guardrails_), not
  here.
* **Not a refresh schedule.** Refresh cadence, expiry windows, and
  retry/back-off live in the host. A profile that needs to disable refresh
  on a specific job uses authz-api job metadata, which is opaque to the
  protocol.

## Worked examples

### File-backed self-hosted setup

```yaml
version: 0.1.0
matches:
  name: github

credentials:
  kind: file
  path: "~/.civic/creds/github.json"
  mode: 0600
```

The host writes/reads the OAuth blob to that file. Suitable for local
development and single-tenant deployments.

### Two connections sharing one file pattern

```yaml
version: 0.1.0
matches:
  - name: github
  - name: gitlab

credentials:
  kind: file
  path: "/var/lib/civic/creds/{name}.json"
```

Each matched connection gets its own file, named for the connection.

### AuthZ-API-backed production setup

```yaml
version: 0.1.0
matches:
  name: github

credentials:
  kind: authz-api
  baseUrl: "https://app.civic.com/authz/api"
```

The host treats the connection name (`github`) as the AuthZ `serverId`. To
override (e.g. when the AuthZ catalogue uses a different identifier):

```yaml
credentials:
  kind: authz-api
  baseUrl: "https://app.civic.com/authz/api"
  serverId: "github-prod"
```
