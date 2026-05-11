# 09 - Execution chain

This is the canonical end-to-end ordering. Every layer doc references back here. If the per-layer text and this chain disagree, **this chain wins**.

## Stages

The chain has three observable stages:

* `list` — the runtime is computing what to return on `tools/list` (and equivalents for resources).
* `request` — a `tools/call` is in flight, before the upstream source executes.
* `response` — a `tools/call` has returned, before the result reaches the agent.

## `list` stage

Triggered on `tools/list`, on `resources/list`, on `resources/templates/list`, and whenever the runtime invalidates the materialised list (e.g. after a skill loads).

```
for each enabled source S (base scope):
  raw = upstream(S).tools/list           # or inline source.tools
  materialised = raw

  # 1. Filter (drops definitions early)
  apply transforms with kind=filter,    target matches S
  materialised → filter applied

  # 2. Alias (rename + meta.upstreamName)
  apply transforms with kind=alias,     target matches S

  # 3. Clone (produce new tools from existing)
  apply transforms with kind=clone,     target matches S

  # 4. Preset-param + describe (final tweaks)
  apply transforms with kind=preset-param, kind=describe

  # 5. List-stage guardrails (may hide more tools conditionally)
  apply guardrails  with stage=list

  # 6. Audit observes the final list
  apply audit       with stage=list

  emit materialised → final tool surface for source S

# repeat for every active skill, in skill-scope
```

Within each step, items run in `executionIndex` order (lower first).

The output is the list returned to the agent.

## `request` stage

Triggered on every `tools/call`. The agent supplies `name + arguments`. The runtime resolves `name` to `(sourceId, scope, upstream tool name)` from the multiplexer.

```
incoming { name, arguments } from agent

  # 1. Reverse alias (so subsequent layers see the upstream-shaped call)
  apply transforms with kind=alias       (request stage)

  # 2. Apply preset-params and clone-merged params
  apply transforms with kind=clone       (request stage, preset merge)
  apply transforms with kind=preset-param

  # 3. Request processors
  apply processors with kind=request

  # 4. Request guardrails — may block, redact, or require-approval
  apply guardrails with stage=request
  if outcome != allow: short-circuit

  # 5. Audit observes the agent-facing form (just before injection)
  apply audit      with stage=request

  # 6. Credential injection (closest to the wire)
  apply credentials.inject for source

  # 7. Dispatch to upstream
  result = upstream(source).tools/call(...)
```

Two ordering invariants worth calling out:

* **Guardrails see what will be sent.** Processors run before guardrails so a guardrail evaluates the body that's about to leave.
* **Audit doesn't see secrets.** Audit runs _before_ credential injection, so the audit record never contains injected secrets.

## `response` stage

Triggered when `result` returns from upstream.

```
upstream result

  # 1. Reverse credential injection (rare; only if the upstream echoes them)
  # (no-op in practice — listed for completeness)

  # 2. Response processors — collapse to agent-facing shape (htmlToMarkdown, removeFields, ...)
  apply processors with kind=response

  # 3. Response guardrails — evaluate the form the agent will see
  apply guardrails with stage=response
  if outcome=block:    surface error
  if outcome=redact:   continue with redacted body
  if outcome=require-approval:  pause & wait

  # 4. Response transforms — clone.responseShape, mapping back to agent-facing names
  apply transforms with kind=clone (response stage)

  # 5. Audit observes the final agent-visible response
  apply audit with stage=response

  → return to agent
```

## Within-stage ordering

Within any single bullet (e.g. "request guardrails"), items run in ascending `executionIndex`.

Default `executionIndex` ranges (recommended; not enforced):

| Layer | Default range |
| --- | --- |
| Filter transforms | 0 |
| Alias transforms | 100 |
| Clone transforms | 200 |
| Preset-param / describe | 300 |
| List guardrails | 0–999 |
| Request processors | 1000+ |
| Request guardrails | 0–999 |
| Response processors | 1000+ |
| Response guardrails | 0–999 |
| Response clone | 200 |
| Audit | always last in stage |

A runtime emitting a profile from existing storage MAY assign default `executionIndex`s in these ranges so that imported profiles match the existing convention.

## Failure handling

| Layer | On exception |
| --- | --- |
| Transform | Abort the stage (operator error — definitions are static config). |
| Processor | Skip + warn. Continue with un-processed payload. |
| Guardrail | Treat as `block` with the runtime error as `reason`. (Fail-closed.) |
| Credential injection | Abort the call with a clear error. |
| Audit | Log and continue. |

This matches today's behaviour: `schema.constraint` failures abort, `response.transform` failures skip, audit best-effort.

## Multi-skill composition

When several skills are active in addition to base, the chain runs once per scope:

* Base scope first (its own materialised list, its own guardrails, etc.).
* Each skill scope second, in load order.
* A `tools/call` is routed to a single scope (the one the called name belongs to).
* _Cross-scope guardrails_: a guardrail with `target.scope: any` (or omitted) applies in every scope. That's how a base-profile guardrail still gates skill calls.

## Diagram

```
                         ┌───────────────────────────────┐
agent ──tools/list──────►│  list stage                  │
                         │   transforms → guardrails    │
agent ◄──filtered list───┤   → audit                    │
                         └───────────────────────────────┘

                         ┌───────────────────────────────┐
agent ──tools/call──────►│  request stage               │
                         │   transforms → processors    │
                         │   → guardrails → audit       │
                         │   → credential injection     │
                         └────────────┬──────────────────┘
                                      │
                                      ▼
                              upstream source
                                      │
                                      ▼
                         ┌───────────────────────────────┐
                         │  response stage              │
                         │   processors → guardrails    │
                         │   → transforms → audit       │
                         └────────────┬──────────────────┘
                                      ▼
agent ◄──result──────────────────────
```
