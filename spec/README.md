# Agent Profile Protocol

> **Status:** Draft proposal. This directory is the authoritative source for the spec.
>
> Captures a config/protocol model for the layer that Civic MCP currently implements between an agent and its underlying tools. The goal is a portable, declarative description of "what an agent can and can't do" that:
>
> * is independent of any specific DB schema, so it can be imported/exported as a single file,
> * is modular — a self-host that runs only the guardrail engine, only credential injection, etc. should be able to consume the relevant subset,
> * stays close to MCP today (the in-scope transport for v1) but does not foreclose CLI / non-MCP tool sources later.

## Why a protocol

Today the Civic MCP hub combines: per-server tool definition transforms (alias/clone/filter/preset params), pre- and post-processors, guardrails (gates), audit observation, credential acquisition + injection, skill bundling, and multiplexing of multiple upstream servers behind one endpoint. All of these live in code and DB rows. None of them is portable.

Pulling a declarative protocol out lets us:

1. Round-trip configuration between the DB and a single file (import/export).
2. Run subsets of the system on their own (e.g. just the guardrail engine, or just credential injection).
3. Give external implementers a target — including, eventually, non-MCP tool sources.

## Sections

The proposal is split into the following files. Read in order if you're new to it; the section docs each assume the principles and the tool definition.

| # | File | Covers |
| --- | --- | --- |
| 00 | [00-principles.md](./00-principles.md) | Goals, terminology, ordering decision, scope boundaries, open questions. |
| 01 | [01-tool-definition.md](./01-tool-definition.md) | Base `Tool` schema — a superset of the MCP tool definition that all other layers reference. |
| 02 | [02-sources-and-multiplexer.md](./02-sources-and-multiplexer.md) | `Source` definitions and how multiple sources compose into one exposed endpoint. |
| 03 | [03-tool-transforms.md](./03-tool-transforms.md) | Alias, clone, filter, preset-params — operations that **modify, add, or remove** tool definitions. |
| 04 | [04-processors.md](./04-processors.md) | Request preprocessors and response postprocessors — content transforms that do **not** change the tool definition. |
| 05 | [05-guardrails.md](./05-guardrails.md) | Gates: rule-based block / redact / require-approval. |
| 06 | [06-audit.md](./06-audit.md) | Read-only observation hooks. |
| 07 | [07-credentials.md](./07-credentials.md) | The non-JSON-RPC concerns: satisfying, persisting/sourcing, and injecting credentials. |
| 08 | [08-skills.md](./08-skills.md) | Bundles of tools + instructions, scoped under an alias. |
| 09 | [09-execution-chain.md](./09-execution-chain.md) | The full pipeline, ordering, and how the layers compose end-to-end. |
| 10 | [10-examples.md](./10-examples.md) | Full and minimal worked profiles in YAML. |

## Top-level shape (preview)

```yaml
apiVersion: civic.com/agent-profile/v1alpha1
kind: AgentProfile
metadata:
  id: "default"
  name: "My toolkit"

sources:        # upstream tool suppliers (MCP servers today)
  - { id: gmail, ... }

skills: []      # optional named bundles

transforms: []  # alias / clone / filter / preset-params

processors: []  # request preprocessors + response postprocessors

guardrails: []  # gates (block / redact / approve)

audit: []       # read-only observers

credentials: []  # provider configs + injection rules
```

Each list item is a self-describing record with a `kind` discriminator, so a runtime that only cares about (say) `guardrails` can ignore everything else.

## What this proposal does not cover

* The agent itself (model, prompting, activity loop) — out of scope.
* Skill platform semantics (Claude skills vs Civic skills vs others). Skills here are referenced, not redefined.
* LLM-driven guardrails (prompt-based pass/fail). Possible future extension, not in v1.
* Stream-processing model (Kafka/Flink-style). Interesting framing, not v1.
* Full CLI / non-MCP tool support. The schema is shaped so a `safeExec` / CLI-wrapped source could slot in later, but no implementation here.

See [00 - Principles, terminology, ordering](./00-principles.md) for the full scope discussion.
