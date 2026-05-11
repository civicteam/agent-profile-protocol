# 08 - Skills

A **skill** in this protocol is a named, scoped bundle of (sources, transforms, processors, guardrails, audit) that the agent can load on demand. Skills do **not** redefine the wider concept of "a skill" — Claude skills, Civic skills, etc. — they describe how a profile groups protocol-level resources under a togglable name.

This maps to today's `Profile` rows with `type: SKILL` plus `ProfileLink`.

## Skill schema

```yaml
Skill:
  alias: string                   # required, unique within profile
  name: string?                   # display name
  description: string?            # display description
  instructions: string?           # text injected when this skill is loaded (for system prompts / context)
  loadMode: enum?                 # always | on-demand; default on-demand

  sources: [SourceRef|Source]?    # may reference base sources by id, or define new ones
  transforms: [Transform]?
  processors: [Processor]?
  guardrails: [Guardrail]?
  audit: [Audit]?
  credentials: [Credential]?      # skill-scoped credentials (rare)
```

A `SourceRef` is just `{ ref: "sources/<id>" }`. This lets a skill _enable_ an existing base source rather than re-declaring it.

## Scoping

When a skill is loaded, its tools are exposed under a scope identified by `alias`:

```yaml
meta:
  sourceId: "<sourceId>"
  scope: { type: "skill", skillAlias: "<alias>" }
```

The default tool prefix becomes `<alias>__<sourceId>-<toolName>`. The base profile's tools (those outside any skill) carry `meta.scope: { type: "base" }` and are not prefixed with a skill alias.

Two skills can each enable the same base source without colliding because each has its own scope. This matches today's `ScopedActionRegistry` behaviour.

## Lifecycle

| `loadMode` | Behaviour |
| --- | --- |
| `always` | The skill is active whenever the profile is. Its tools appear immediately on `tools/list`. |
| `on-demand` | The skill is inactive until the agent invokes a runtime-provided `load-skill` tool. |

`on-demand` skills require the runtime to expose a built-in management tool (today's `SkillManagementHook`). The protocol declares the skill's intent; the runtime decides how the agent activates it.

## Composition rules

When a skill is active, its protocol layers compose with the base profile's layers as follows:

1. **Sources**: union. A skill may add new sources or `ref` existing base sources. Tools from `ref`'d sources show up only under the skill scope.
2. **Transforms**: union. Skill transforms target the skill's scope (`target.scope: skill` is implicit). To target a base-scope tool from inside a skill, the transform must explicitly declare `target.scope: base`.
3. **Processors**: union. Same scoping rule as transforms.
4. **Guardrails**: union. **Important:** skill guardrails do not weaken base guardrails. If both a base guardrail and a skill guardrail target the same call, both run; the more restrictive outcome wins (`block` > `require-approval` > `redact` > `allow`).
5. **Audit**: union.
6. **Credentials**: skill credentials are scoped to the skill — they do not leak to the base profile or other skills.

## Skill-injected instructions

`instructions` is a text block appended to the agent's context when the skill loads. This is what makes a skill more than just a tool bundle — it can ship the prompt fragment that explains _how to use_ the bundled tools.

```yaml
skills:
  - alias: jira
    name: "Jira (Civic)"
    instructions: |
      When working with Jira:
      - Always use the NEXUS project (key: NEXUS) on cloud 6f06dc08-...
      - Use transition id 2 to move issues to "Selected for Development".
      - Story points live in customfield_10021.
    sources:
      - { ref: "sources/atlassian" }
    guardrails:
      - stage: request
        target: { sourceId: "atlassian", toolName: "createJiraIssue" }
        when: { schemaPath: "$.projectKey", operator: "equals", value: "NEXUS" }
        outcome: allow
```

The `instructions` field is the protocol's hook for the kind of "embed knowledge close to the tools" use case raised on the call. It deliberately doesn't try to _be_ a skills platform; it just ships text alongside the bundle.

## Worked example

```yaml
skills:
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
          tools: ["search_issues", "get_pull_request", "list_pulls", "get_repository"]
```

When the agent calls `load-skill('github-readonly')`, the runtime exposes the four whitelisted GitHub tools under that skill's scope.

## What this maps to today

| Concept | Today |
| --- | --- |
| `Skill` | `Profile` row with `type: SKILL` |
| `alias` | `Profile.alias` |
| `instructions` | `Profile.instructions` |
| `loadMode: always` | A skill marked as always-loaded in `ProfileLink` |
| `loadMode: on-demand` | The default for `ProfileLink` |
| Scoping | The `skill` scope in `ScopedActionRegistry` |
| Skill-scoped sources | `AccountServer` rows with `profileId = skillProfileId` |
