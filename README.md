# agent-profile-protocol

Home of the **Agent Profile Protocol** — a declarative, portable description of the layer between an agent and its underlying tools (sources, transforms, processors, guardrails, audit, credentials, skills).

The protocol is what Civic MCP implements today in code and DB rows, lifted out into a single round-trippable document so:

- configuration can be imported/exported as one file,
- runtimes can adopt subsets (e.g. just guardrails, or just credential injection),
- external implementers — including future non-MCP sources — have a target to build against.

## Where the spec lives

The current draft is in [`spec/`](./spec/README.md). Start there.

Sections (read in order if you're new to it):

1. [00 - Principles, terminology, ordering](./spec/00-principles.md)
2. [01 - Tool definition](./spec/01-tool-definition.md)
3. [02 - Sources and multiplexer](./spec/02-sources-and-multiplexer.md)
4. [03 - Tool transforms](./spec/03-tool-transforms.md)
5. [04 - Pre- and post-processors](./spec/04-processors.md)
6. [05 - Guardrails](./spec/05-guardrails.md)
7. [06 - Audit](./spec/06-audit.md)
8. [07 - Credentials](./spec/07-credentials.md)
9. [08 - Skills](./spec/08-skills.md)
10. [09 - Execution chain](./spec/09-execution-chain.md)
11. [10 - Examples](./spec/10-examples.md)

## Status

Draft (`apiVersion: civic.com/agent-profile/v1alpha1`). This repo is the authoritative source for the spec.
