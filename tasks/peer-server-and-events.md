# Two peer MCP servers watch each other's state

**Status:** Stub, 2026-10-16. The first child of [events-carry-state.md](events-carry-state.md), whose § *Design intent* and § *Decisions* are this file's contract. The phases below are outlined from Larry's plan; their criteria are written after `/review-task` Stage 1 has run on the master. It gates the demo, which the master lists in § *What remains*.

## Objective

**Two instances of one Durable Object class, each an MCP server and an MCP caller, serve state and Events to each other, with every message validated against JSON Schema generated from the spec's TypeScript.**

## Relationships

- **Contract:** [events-carry-state.md](events-carry-state.md) § *Design intent*, § *Constraints* and § *Decisions*. A decision a phase here needs, and the master lacks, goes back to the master first.
- **Open questions it waits on**, in the master's § *Open questions*: which Lumenize packages to depend on, task state, and how a webhook subscriber authenticates.

## Phases (outline)

1. **A minimal MCP server and caller in one Durable Object class.** It validates every message against the spec's `schema.json` with `@cfworker/json-schema`, and works with Cloudflare's MCP playground.
2. **State, read and written through tools, still without Events.** `tools/list` and `tools/call` serve it. The same notes are also served as resources under `https://` URIs, listed by `resources/list` and read with `resources/read`, and the Worker answers a plain `GET` on each URI with an `ETag` and `304 Not Modified`. No legacy `resources/subscribe`.
3. **Events, with subscribe, webhook delivery and poll.** The two instances subscribe to each other's state by webhook, and a poll client reads it by `events/poll`. The Events types and their generated `events.json` come first, and Sri's conformance suites run against the result.
