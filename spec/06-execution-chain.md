# 06 - Execution chain

This is the canonical end-to-end ordering. Every layer doc references back
here. If the per-layer text and this chain disagree, **this chain wins**.

## Inputs

For each MCP connection **C** the host has registered, the host has
collected an effective overlay — the concatenation, in document order, of
every profile document whose `matches` block accepts C (_01 - Matching_).
The overlay is three ordered lists:

* `transforms` (alias, clone, filter, preset-parameter)
* `processors` (request and response)
* `guardrails` (list, request, response)

The chain below runs against that overlay.

## Stages

The chain has three observable stages:

* `list` — the host is computing what to return on `tools/list` for C.
* `request` — a `tools/call` is in flight, before the upstream executes.
* `response` — a `tools/call` has returned, before the result reaches the
  agent.

## `list` stage

Triggered on every `tools/list` for C, and whenever the host invalidates the
materialised list (e.g. after a connection reports a tool change).

```
raw = upstream(C).tools/list

materialised = raw

# 1. Clone — introduce derived tools BEFORE filter (so a tool can be cloned
#    and the original hidden in the same overlay).
apply transforms with kind = clone

# 2. Filter — drop tools the agent should not see.
apply transforms with kind = filter

# 3. Alias — rename remaining tools. Terminal: source name is gone afterwards.
apply transforms with kind = alias

# 4. Preset-parameter — fix or hide parameters on the final names.
apply transforms with kind = preset-parameter

# 5. List-stage guardrails — may hide more tools conditionally.
apply guardrails with stage = list

emit materialised → agent-facing tool list for C
```

Within each step, items run in `executionIndex` order (ascending). The
default execution indices (recommended; not enforced) are listed under
"Within-stage ordering" below.

## `request` stage

Triggered on every `tools/call`. The agent supplies `name + arguments`. The
host resolves `name` to the upstream tool by walking the materialised list
in reverse — undoing aliases and clones until it reaches the underlying
upstream name.

```
incoming { name, arguments } from agent

# 1. Reverse alias / clone — recover the upstream-shaped call.
apply transforms with kind = alias        (request stage: rewrite name)
apply transforms with kind = clone        (request stage: merge clone presets)

# 2. Apply parameter presets and interpolation.
apply transforms with kind = preset-parameter

# 3. Request processors.
apply processors with kind = request

# 4. Request guardrails — may block, redact, or require approval.
apply guardrails with stage = request
if outcome != allow: short-circuit

# 5. Dispatch to upstream.
result = upstream(C).tools/call(...)
```

One invariant worth calling out:

* **Guardrails see what will be sent.** Processors run before guardrails so
  a guardrail evaluates the body that's about to leave.

## `response` stage

Triggered when `result` returns from upstream.

```
upstream result

# 1. Response processors — collapse to agent-facing shape (htmlToMarkdown,
#    pickFields, etc).
apply processors with kind = response

# 2. Response guardrails — evaluate the form the agent will see.
apply guardrails with stage = response
if outcome == block:             surface error
if outcome == redact:            continue with redacted body
if outcome == require-approval:  pause and wait

# 3. Response-side transforms — map upstream result back to clone-shaped
#    output if the clone declared output reshaping.
apply transforms with kind = clone        (response stage)

→ return to agent
```

## Within-stage ordering

Within any single bullet (e.g. "request guardrails"), items run in
ascending `executionIndex`.

Recommended default execution indices (not enforced — a host MAY assign
these when a profile omits explicit values, so that profiles written without
explicit ordering still materialise predictably):

| Step | Default range |
| --- | --- |
| Clone transforms (list) | 100 |
| Filter transforms (list) | 200 |
| Alias transforms (list) | 300 |
| Preset-parameter transforms (list) | 400 |
| List guardrails | 0–999 |
| Request transforms (alias / clone reverse) | 100–200 |
| Preset-parameter (request) | 400 |
| Request processors | 1000+ |
| Request guardrails | 0–999 |
| Response processors | 1000+ |
| Response guardrails | 0–999 |
| Response clone transforms | 200 |

## Composition across profile documents

When several profile documents bind to C, their `transforms`,
`processors`, and `guardrails` lists are concatenated in **document order**
before the chain runs (_01 - Matching_, "Composition"). Document order is
the only precedence — there is no implicit "more specific match wins".

Concretely, if Doc A has `[T1, T2]` and Doc B has `[T3]`, C's effective
transform list is `[T1, T2, T3]`. T1 may rename a tool that T3 then
preset-parameterises.

`executionIndex` overrides document order: if T3 has `executionIndex: 50`
and T1 has the default `100`, T3 runs first within its stage even though it
came from a later document.

## Failure handling

| Layer | On exception | On `schemaPath` matching zero nodes |
| --- | --- | --- |
| Transform | Startup error — profile cannot load. There is no per-call transform failure. | n/a |
| Processor | Skip + warn. Continue with un-processed payload. The exception is `action: block`, which aborts intentionally. | Skip + warn. |
| Guardrail | Treat as `block` with the runtime error as `reason`. (Fail-closed.) | Per the guardrail's `onNoMatch` (see _04 - Guardrails_); defaults to `fail-closed` for `block` and `redact`, `allow` for `require-approval` and `allow`. |

## Diagram

```
                         ┌───────────────────────────────┐
agent ──tools/list──────►│  list stage                   │
agent ◄──filtered list───┤   transforms → guardrails     │
                         └───────────────────────────────┘

                         ┌───────────────────────────────┐
agent ──tools/call──────►│  request stage                │
                         │   transforms (reverse) →      │
                         │   processors → guardrails     │
                         └────────────┬──────────────────┘
                                      │
                                      ▼
                              upstream MCP server (C)
                                      │
                                      ▼
                         ┌───────────────────────────────┐
                         │  response stage               │
                         │   processors → guardrails     │
                         │   → transforms                │
                         └────────────┬──────────────────┘
                                      ▼
agent ◄──result──────────────────────
```
