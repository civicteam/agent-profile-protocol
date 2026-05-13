# 09 - Examples

Two worked example profiles — a fully-populated one demonstrating every layer, and a minimal one showing the smallest useful shape.

## Full example

Demonstrates every layer of the protocol with a realistic combination:

* Two MCP sources (Gmail + GitHub) plus one hosted source (Postgres) plus an HTTP Atlassian source.
* Tool transforms (filter, alias, clone, preset-param, describe).
* Pre/post processors (HTML→Markdown, default region, field stripping).
* Guardrails (block external email, redact secrets, require approval).
* Credentials (OAuth + PAT + service account file).
* One always-loaded skill (Jira) and one on-demand skill (GitHub-readonly).

```yaml
apiVersion: civic.com/agent-profile/v1alpha1
kind: AgentProfile
metadata:
  id: "engineering-default"
  name: "Engineering toolkit"
  description: "Default profile for engineering agents."

# ─── SOURCES ──────────────────────────────────────────────────────────────────
sources:
  - id: gmail
    kind: mcp.hosted
    name: "Gmail"
    transport:
      type: hosted
      image: "ghcr.io/civicteam/gmail-mcp:1.0.0"
    auth: { ref: "credentials/gmail-oauth" }

  - id: github
    kind: mcp.http
    name: "GitHub"
    transport:
      type: http
      url: "https://api.githubcopilot.com/mcp"
    auth: { ref: "credentials/github-pat" }

  - id: postgres
    kind: mcp.hosted
    name: "Postgres (read-only)"
    transport:
      type: hosted
      image: "ghcr.io/civicteam/postgres-mcp:0.5.1"
      env:
        DATABASE_URL: "postgresql://reader@db.internal/analytics"
    auth: { ref: "credentials/postgres-readonly" }

  - id: atlassian
    kind: mcp.http
    name: "Atlassian"
    transport:
      type: http
      url: "https://app.civic.com/atlassian/mcp"
    auth: { ref: "credentials/atlassian-oauth" }

# ─── TRANSFORMS (alias / clone / filter / preset-param / describe) ────────────
transforms:
  # Hide destructive Gmail tools.
  - kind: filter
    target: { sourceId: "gmail" }
    spec:
      mode: blacklist
      tools: ["delete_message", "trash_message"]

  # Rename gmail.send_email → email_team and preset the recipient pattern.
  - kind: alias
    target: { sourceId: "gmail", toolName: "send_email" }
    spec:
      name: "email_team"
      description: "Sends an email to the engineering team."

  # Provide a second narrowed view of github.create_issue as create_bug_report.
  - kind: clone
    target: { sourceId: "github", toolName: "create_issue" }
    spec:
      name: "create_bug_report"
      description: "Files a bug against the engineering project."
      preset:
        owner: "civicteam"
        repo: "civic-mcp"
        labels: ["bug"]
      responseShape:
        pick: ["id", "number", "html_url", "state"]

  # Tag every Postgres write with destructive so a guardrail can require approval.
  - kind: describe
    target: { sourceId: "postgres", tags: ["destructive"] }
    spec:
      descriptionPrefix: "[Requires approval] "

  # Postgres always queries in read-only role.
  - kind: preset-param
    target: { sourceId: "postgres" }
    spec:
      values: { role: "reader" }
      hideFromInputSchema: true

# ─── PROCESSORS (request preprocessors + response postprocessors) ─────────────
processors:
  # Trim noisy fields from Postgres results.
  - kind: response
    target: { sourceId: "postgres" }
    when: { schemaPath: "$.rows", operator: "exists", value: true }
    action: pickFields
    actionPath: "$.rows[*]"
    actionParams:
      fields: ["id", "summary", "status", "updated_at"]
    executionIndex: 1100

  # Convert HTML in any response body to Markdown for token efficiency.
  - kind: response
    target: { sourceId: "*" }
    when: { schemaPath: "$.body", operator: "exists", value: true }
    action: htmlToMarkdown
    actionPath: "$.body"
    actionParams: { preserveImages: true, preserveLinks: true, stripStyles: true }
    executionIndex: 1000

  # Always default GitHub repo owner if the agent forgets to specify it.
  - kind: request
    target: { sourceId: "github" }
    action: setDefault
    actionPath: "$"
    actionParams:
      values: { owner: "civicteam" }

# ─── GUARDRAILS (gates) ───────────────────────────────────────────────────────
guardrails:
  # Block email to non-civic recipients.
  - stage: request
    target: { sourceId: "gmail" }
    when:
      schemaPath: "$.to[*]"
      operator: "not_ends_with"
      value: "@civic.com"
    outcome: block
    reason: "Outbound email is restricted to @civic.com recipients."
    executionIndex: 0

  # Redact API keys appearing anywhere in any response.
  - stage: response
    target: { sourceId: "*" }
    when:
      schemaPath: "$..*"
      operator: "regex"
      value: "(?i)api[_-]?key|secret|bearer\\s+[A-Za-z0-9._-]{20,}"
    outcome: redact
    outcomeParams: { replacement: "[REDACTED]" }
    executionIndex: 0

  # Require approval for any tool tagged destructive.
  - stage: request
    target: { sourceId: "*", tags: ["destructive"] }
    when: { schemaPath: "$", operator: "exists", value: true }
    outcome: require-approval
    outcomeParams:
      prompt: "Approve destructive call to {{tool.name}}?"
      expirySeconds: 300
    executionIndex: 100

# ─── CREDENTIALS ──────────────────────────────────────────────────────────────
credentials:
  - id: gmail-oauth
    kind: oauth2
    satisfy:
      flow: browser-redirect
      authorizationUrl: "https://accounts.google.com/o/oauth2/v2/auth"
      tokenUrl: "https://oauth2.googleapis.com/token"
      scopes:
        - "https://www.googleapis.com/auth/gmail.send"
        - "https://www.googleapis.com/auth/gmail.readonly"
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

  - id: postgres-readonly
    kind: custom
    satisfy:
      flow: manual-entry
      visibility: account
      fields:
        - { name: "host", required: true }
        - { name: "user", required: true, default: "reader" }
        - { name: "password", required: true, type: "password" }
    source:
      type: ref
      ref: { id: "postgres-readonly-default" }
    inject:
      - target: env
        name: "DATABASE_URL"
        value: "postgresql://{{user}}:{{password}}@{{host}}/analytics"

  - id: atlassian-oauth
    kind: oauth2
    satisfy:
      flow: browser-redirect
      authorizationUrl: "https://auth.atlassian.com/authorize"
      tokenUrl: "https://auth.atlassian.com/oauth/token"
      scopes: ["read:jira-work", "write:jira-work", "manage:jira-project"]
      clientIdRef: "secrets/atlassian-client-id"
      clientSecretRef: "secrets/atlassian-client-secret"
      visibility: user
    source: { type: ref, ref: { id: "atlassian-oauth-{{user.id}}" } }
    inject:
      - target: header
        name: "Authorization"
        value: "Bearer {{access_token}}"

# ─── SKILLS ───────────────────────────────────────────────────────────────────
skills:
  - alias: jira
    name: "Jira (Civic NEXUS)"
    description: "Jira tools pre-configured for the NEXUS project."
    loadMode: always
    instructions: |
      When working with Jira:
      - Default project is NEXUS on cloud 6f06dc08-85bd-46af-ae2f-afabc2ccd057.
      - Story points custom field id is customfield_10021.
      - Use transition id 2 to move issues to "Selected for Development".
    sources:
      - { ref: "sources/atlassian" }
    transforms:
      - kind: preset-param
        target: { sourceId: "atlassian", scope: skill }
        spec:
          values:
            cloudId: "6f06dc08-85bd-46af-ae2f-afabc2ccd057"
            projectKey: "NEXUS"
          hideFromInputSchema: true

  - alias: github-readonly
    name: "GitHub (read-only)"
    description: "GitHub access without write operations."
    loadMode: on-demand
    sources:
      - { ref: "sources/github" }
    transforms:
      - kind: filter
        target: { sourceId: "github", scope: skill }
        spec:
          mode: whitelist
          tools:
            - "search_issues"
            - "get_pull_request"
            - "list_pulls"
            - "get_repository"
```

## Minimal example

The smallest useful profile: one Gmail source with one guardrail. Demonstrates that every layer is optional except `sources`.

```yaml
apiVersion: civic.com/agent-profile/v1alpha1
kind: AgentProfile
metadata:
  id: "personal-gmail"
  name: "Personal Gmail (read-only outside @civic.com)"

sources:
  - id: gmail
    kind: mcp.hosted
    transport:
      type: hosted
      image: "ghcr.io/civicteam/gmail-mcp:1.0.0"
    auth: { ref: "credentials/gmail-oauth" }

guardrails:
  - stage: request
    target: { sourceId: "gmail", toolName: "send_email" }
    when:
      schemaPath: "$.to[*]"
      operator: "not_ends_with"
      value: "@civic.com"
    outcome: block
    reason: "Outbound email is restricted to @civic.com recipients."

credentials:
  - id: gmail-oauth
    kind: oauth2
    satisfy:
      flow: browser-redirect
      authorizationUrl: "https://accounts.google.com/o/oauth2/v2/auth"
      tokenUrl: "https://oauth2.googleapis.com/token"
      scopes: ["https://www.googleapis.com/auth/gmail.send"]
      clientIdRef: "secrets/google-client-id"
      clientSecretRef: "secrets/google-client-secret"
      pkce: true
      visibility: user
    source: { type: ref, ref: { id: "gmail-oauth-{{user.id}}" } }
    inject:
      - target: env
        name: "OAUTH_ACCESS_TOKEN"
        value: "{{access_token}}"
```
