# Agent Profile Protocol

> **Status:** Draft proposal. This directory is the authoritative source for the spec.
>
> A declarative format for the manipulation layer between an agent client and one or more
> upstream MCP servers. A profile is a multi-document YAML stream; each document declares
> which connection(s) it applies to (via `matches`) and what overlays — transforms,
> processors, guardrails — should attach.
>
> The protocol intentionally does **not** cover connection setup, transport, or
> credential management. Those are host concerns; profiles attach to connections the host
> already has.

## Why a protocol

Today Civic MCP combines tool-list overlays (alias / clone / filter / preset params),
pre- and post-processors, and guardrails in code and DB rows. Pulling them out into a
portable spec lets:

1. Configuration round-trip between storage and a single file.
2. Hosts adopt subsets (just guardrails, just transforms, …).
3. External implementers build against a defined target.

## Sections

Read in order if you're new to it; the section docs assume the principles and matching
model from sections 00 and 01.

| # | File | Covers |
| --- | --- | --- |
| 00 | [00-principles.md](./00-principles.md) | Goals, terminology, ordering decision, scope boundaries, open questions. |
| 01 | [01-matching.md](./01-matching.md) | Multi-document YAML structure, `matches` block, application lifecycle, multi-document composition. |
| 02 | [02-tool-transforms.md](./02-tool-transforms.md) | `alias` / `clone` / `filter` / `preset-parameter` — operations that modify, add, or remove tool definitions. |
| 03 | [03-processors.md](./03-processors.md) | Request preprocessors and response postprocessors — best-effort content reshaping. |
| 04 | [04-guardrails.md](./04-guardrails.md) | Gates: rule-based block / redact / require-approval. |
| 05 | [05-credentials.md](./05-credentials.md) | Per-connection credential persistence strategy (`file`, `authz-api`). |
| 06 | [06-execution-chain.md](./06-execution-chain.md) | The full pipeline, ordering, and how the layers compose end-to-end. |
| 07 | [07-examples.md](./07-examples.md) | Worked example profile in YAML. |

## Top-level shape (preview)

A profile file is a YAML stream — each document independent, separated by `---`:

```yaml
version: 0.1.0
matches:
  name: github
  version: "^0.1"     # `type:` defaults to `mcp`

transforms: []       # alias / clone / filter / preset-parameter
processors: []       # request preprocessors + response postprocessors
guardrails: []       # gates (block / redact / approve)
---
version: 0.1.0
# No `matches:` — applies to every registered connection.
transforms: []
```

Each list entry is a tagged object: transforms use `kind:` (`alias`, `clone`, `filter`,
`preset-parameter`); processors and guardrails use `stage:` (the leg of the chain they
attach to). The top-level list a record appears under is itself a discriminator, so a
host that only cares about (say) `guardrails` can ignore everything else.

## What this proposal does not cover

* The agent itself (model, prompting, activity loop) — out of scope.
* Connection setup and transport — host concerns. The protocol attaches to connections
  the host has already established.
* Credential **acquisition flows** (OAuth dances, MFA, refresh schedules). The protocol
  does declare _where_ a credential blob is persisted (see _05 - Credentials_), but not
  _how_ it is obtained.
* Skill platforms. A profile is overlays on connections, not a bundle of prompts +
  lifecycle.
* LLM-driven guardrails (prompt-based pass/fail). Possible v1.1 extension.
* Non-MCP transports. The `matches.type` field is shaped for future expansion but v1
  only defines `mcp`.

See [00 - Principles, terminology, ordering](./00-principles.md) for the full scope
discussion.
