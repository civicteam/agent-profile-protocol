# 07 - Credentials

Credentials sit _outside_ the JSON-RPC / MCP message layer. The protocol describes three concerns explicitly, and intentionally keeps them separable so an implementer can adopt one without the others.

The three concerns (matching the call):

1. **Satisfying** — how the runtime obtains a credential when it doesn't have one (auth flow shape).
2. **Persisting / sourcing** — where the credential lives at rest, and how it's retrieved at request time.
3. **Injecting** — how the credential reaches the upstream source (env var, request header, file in a container volume, etc.).

A profile MAY use any subset. A runtime that only implements injection (e.g. it reads pre-shared secrets from local env) can ignore the satisfaction and persistence sections.

## Top-level shape

```yaml
credentials:
  - id: gmail-oauth        # referenced from sources[*].auth.ref as "credentials/gmail-oauth"
    kind: oauth2           # oauth2 | apikey | basic | custom
    satisfy: { ... }
    source: { ... }
    inject: { ... }

  - id: github-pat
    kind: apikey
    satisfy: { ... }
    source: { ... }
    inject: { ... }
```

## `kind`

| `kind` | Notes |
| --- | --- |
| `oauth2` | Full OAuth 2.x with PKCE (default). Authorization-code is the v1 flow. |
| `apikey` | A single opaque string (header, query param, or env var). |
| `basic` | Username + password. |
| `custom` | Free-form fields the satisfy flow collects (e.g. `host`, `port`, `private_key`). The `source` and `inject` blocks treat the result as an opaque object. |

## Satisfy

How the runtime _gets_ a credential the first time, or refreshes one. The schema is `kind`-specific but shares a frame:

```yaml
satisfy:
  flow: enum                       # browser-redirect | device-code | manual-entry | client-credentials
  authorizationUrl: string?
  tokenUrl: string?
  scopes: [string]?
  clientIdRef: string?             # reference to a runtime-managed secret
  clientSecretRef: string?
  pkce: boolean?                   # default true for browser-redirect
  refresh:
    enabled: boolean?              # default true if a refresh_token is present
    grantType: enum?               # refresh_token | client_credentials
    skewSeconds: int?              # refresh this many seconds before expiry
```

For `apikey` / `basic`:

```yaml
satisfy:
  flow: manual-entry
  prompt: "Paste your GitHub personal access token."
  fields:                          # for kind: custom
    - name: host
      label: "Host"
      required: true
    - name: port
      label: "Port"
      type: number
      default: 22
```

The protocol describes the _shape_ of a satisfaction flow. The actual UI and OAuth dance are runtime concerns.

### Visibility

```yaml
satisfy:
  visibility: account              # account | profile | user
```

* `account`: the credential is shared across every profile in the account.
* `profile`: the credential is bound to one profile.
* `user`: the credential is bound to a single user (each user satisfies their own copy).

This mirrors the existing `CredentialJob.visibility` enum.

## Source

Where the credential is read from at request time. This is what a runtime that only implements _injection_ needs.

```yaml
source:
  type: enum                       # ref | env | file | inline-encrypted
  # type=ref — runtime-managed encrypted store, looked up by id (matches today's CredentialData)
  ref: { id: "gmail-oauth-default" }

  # type=env — read from process env at request time
  env: { name: "GMAIL_TOKEN" }

  # type=file — read from a file at request time
  file: { path: "/run/secrets/gmail-token", format: "json" | "raw" }

  # type=inline-encrypted — runtime-decryptable inline blob (avoid for portable profiles)
  inlineEncrypted: { ciphertext: "...", keyRef: "..." }
```

For `oauth2`, the resolved value is an object with at least `access_token` and (optionally) `refresh_token`, `expires_at`. For `apikey`, it's a string. For `custom`, it's the object the satisfy flow produced.

## Inject

How the credential reaches the upstream source.

```yaml
inject:
  - target: header                 # header | env | query | body | configFile
    name: "Authorization"
    value: "Bearer {{access_token}}"

  - target: env
    name: "GOOGLE_APPLICATION_CREDENTIALS"
    value: "/run/config/gcp.json"

  - target: configFile             # written by an init-sidecar at deploy time (matches today's pattern)
    path: "/srv/app/config/gcp.json"
    format: json
    contentTemplate: |
      {
        "type": "service_account",
        "client_email": "{{client_email}}",
        "private_key": "{{private_key}}"
      }
```

A credential MAY produce multiple injections (e.g. an OAuth token used both as a bearer header and as an env var inside a container). Each injection is independent.

The template language is intentionally tiny: `{{path.into.credential}}` interpolation only. Runtimes MUST refuse to start on syntactically invalid templates rather than emit empty strings. Any larger templating belongs in a processor.

### Injection targets and which sources accept them

| `target` | Compatible source `kind` | Behaviour |
| --- | --- | --- |
| `header` | `mcp.http`, `mcp.sse` | Adds to outgoing request headers. |
| `env` | `mcp.stdio`, `mcp.hosted` | Sets in the spawned/hosted process's environment. |
| `query` | `mcp.http`, `mcp.sse` | Appends as a query parameter. |
| `body` | (any RPC source) | Merges into the JSON-RPC request body — rare, used only when the upstream protocol expects the credential in-band. |
| `configFile` | `mcp.hosted`, `mcp.stdio` | Materialises a file via init-sidecar (matches today's pattern in `etc/specs/credential-injection-strategy.md`). |

## Where in the chain

Credential injection runs **just before dispatch**, after every other request-stage layer (transforms, processors, guardrails). This is by design:

* Earlier layers operate on the agent-facing request shape; they should not see secrets.
* Audit (which runs last) sees the agent-facing form, _not_ the injected form, so secrets don't leak into audit records.
* Guardrails do not gate on credential values; if you need to reject "unsigned" requests, that's a guardrail on a different field.

See _09 - Execution chain_.

## Worked example — Gmail OAuth + GitHub PAT

```yaml
credentials:
  - id: gmail-oauth
    kind: oauth2
    satisfy:
      flow: browser-redirect
      authorizationUrl: "https://accounts.google.com/o/oauth2/v2/auth"
      tokenUrl: "https://oauth2.googleapis.com/token"
      scopes: ["https://www.googleapis.com/auth/gmail.send", "https://www.googleapis.com/auth/gmail.readonly"]
      clientIdRef: "secrets/google-client-id"
      clientSecretRef: "secrets/google-client-secret"
      pkce: true
      refresh: { enabled: true, skewSeconds: 60 }
      visibility: user
    source:
      type: ref
      ref: { id: "gmail-oauth-{{user.id}}" }
    inject:
      - target: env
        name: "OAUTH_ACCESS_TOKEN"
        value: "{{access_token}}"

  - id: github-pat
    kind: apikey
    satisfy:
      flow: manual-entry
      prompt: "Paste your GitHub PAT (needs repo + read:org)."
      visibility: profile
    source:
      type: ref
      ref: { id: "github-pat-default" }
    inject:
      - target: header
        name: "Authorization"
        value: "Bearer {{value}}"
```

## What this maps to today

| Section | Today |
| --- | --- |
| `kind: oauth2` `satisfy` | `OAuthCredentials` table + the `AuthenticationHook` flow. |
| `kind: apikey` / `basic` | The same hook with a non-OAuth path. |
| `source.type: ref` | `CredentialData` rows, encrypted at rest. |
| `inject.target: env` / `header` | `McpServer.deployment.authMapping` + request-time header injection. |
| `inject.target: configFile` | The init-sidecar pattern in `etc/specs/credential-injection-strategy.md`. |
| `visibility` | `CredentialJob.visibility` enum (USER / ACCOUNT). The protocol adds `profile` for symmetry; today this is implicit. |

A runtime that does not implement satisfy can still ingest profiles that _use_ a credential, as long as it can produce the resolved value via `source: { type: env|file|... }`. This is the modular guarantee.
