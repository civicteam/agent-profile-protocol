# agent-profile-protocol

Home of the **Agent Profile Protocol** — a declarative, portable description of the
manipulation layer between an agent client and the upstream MCP servers a host has
connected to. A profile attaches transforms, processors, and guardrails to one or more
MCP connections via a `matches` block, expressed as a multi-document YAML stream.

The protocol is lifted out of what Civic MCP implements today in code and DB rows, so:

- configuration can be imported/exported as one file,
- hosts can adopt subsets (e.g. just guardrails),
- external implementers — including future non-MCP transports — have a target to build
  against.

The protocol is deliberately **scoped to manipulation and persistence**: it does not
cover connection setup, transport, or credential acquisition flows. It does declare
**where** credentials are persisted (file, authz-api, …) so a profile can move between
deployments without rewriting everything else.

## Where the spec lives

The current draft is in [`spec/`](./spec/README.md). Start there.

Sections (read in order if you're new to it):

1. [00 - Principles, terminology, ordering](./spec/00-principles.md)
2. [01 - Matching](./spec/01-matching.md)
3. [02 - Tool transforms](./spec/02-tool-transforms.md)
4. [03 - Pre- and post-processors](./spec/03-processors.md)
5. [04 - Guardrails](./spec/04-guardrails.md)
6. [05 - Credentials](./spec/05-credentials.md)
7. [06 - Execution chain](./spec/06-execution-chain.md)
8. [07 - Examples](./spec/07-examples.md)

## Status

Draft (`version: 0.1.0`). This repo is the authoritative
source for the spec.
