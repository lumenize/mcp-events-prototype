# MCP Events carries watched state, shown on Cloudflare

**Status:** Master plan for this repo, Pass 1 drafted 2026-10-16 for Larry's read. It holds the design intent that every child task file conforms to; the children are listed in § *What remains* and written one at a time. Nothing gates it, and Lumenize's pre-alpha is on hold until it is done. Decided 2026-10-16: an independent implementation on Cloudflare, rather than inside the official TypeScript SDK or on `cloudflare/agents`, with the spec's own `schema.ts` as its only schema. Also decided that day: a minimal MCP that answers every request with JSON, opens no SSE stream, and implements only the methods Events needs.

## Objective

**A small MCP server on Cloudflare, written from the spec, that carries discrete events and watched state over the same `events/*` methods, passes the working group's conformance suites, and puts Cloudflare's Workers Issues events in front of a live Claude Code session.**

## What remains

1. **The build**, in [peer-server-and-events.md](peer-server-and-events.md). Two instances of one Durable Object class, each an MCP server and an MCP caller, serve state and Events to each other.
2. **The demo.** Not written yet. A browser form edits state on one instance through a tool, the change reaches the other instance by webhook, and a second browser, polling `events/poll` on a short interval, shows it. Both browsers are MCP clients, so nothing in the demo sits outside MCP. It should show an offline client catching up from its cursor, updates coalescing while an endpoint is down, a patch against a full resync, a deleted key, and a panel of what crossed the wire. An optional instrumentation channel, labeled as such and outside the data path, could show the instant a webhook lands and count down to the next poll.
3. **The Issues relay**, if it stays, per § *Open questions*.
4. **The report to the working group.** The list of wire additions, the Events types and their `events.json`, a field report on the working group's repo, and the reply to Peter on the SEP thread. It also says whether to propose a `state` contract at all, or only conditional `GET` for web resources plus Events for push and collections.

## Context

**SEP-3415 defines Events for discrete occurrences and leaves watched state as two open questions.** Open Question 2 asks whether resource subscriptions should be re-expressed as events, with a reserved `mcp.resource.updated` type whose argument is `{uri}`. Open Question 6 asks the same of task state, with `mcp.task.updated` and `{taskId}`, and says its considerations are those of question 2. Today a resource watcher gets `notifications/resources/updated`, which says only that a URI changed, and a task watcher polls `tasks/get`.

**The PR thread has narrowed the question to one of API shape.** Larry asked whether an event type could declare that it coalesces, and whether coalescing should set `truncated`. Peter Alexander agreed it should coalesce without truncating, then questioned whether watching state and consuming messages are the same abstraction, and tied the question to task state. mib00038 proposed that each event type declare one of two contracts in `events/list`: `message`, where every occurrence matters, and `state`, where only the newest value per key does. Larry's reply argued that questions 2 and 6 share that `state` contract, so it belongs once in the extension, and offered a prototype to show where, if anywhere, the API has to diverge.

**The official SDK work is spoken for, which frees this one to be small.** Core Maintainers review an Extensions Track SEP only once an official SDK implements it. On 2026-10-16 Peter asked the working group's Discord for an up-to-date implementation, described his own TypeScript branch as an early attempt against an older spec and SDK v1, and recommended starting fresh on v2. Sriram Panyam took it on. He also wrote the conformance suites (conformance #521 for servers, #541 for clients) and has an experimental Go implementation. This prototype answers questions 2 and 6 and nothing more, so it implements only as much core MCP as Events needs.

**Cloudflare already ships Events, and Claude Code can already take pushed messages, but the two do not meet.** Cloudflare's API MCP server offers one event type, Workers Issues, by webhook only. A Claude Code channel is a local stdio process that pushes `notifications/claude/channel` into a session, and it must stay on the handshake from before 2026-07-28. A laptop cannot receive Cloudflare's webhook, so something on Cloudflare has to hold the subscription.

The rest of this file sets out the goals, what exists to build on, the contract the children must meet, and five decisions still open.

## Goals

1. **Watched state and discrete events share one set of `events/*` methods, and differ only in a contract each event type declares.** Today the SEP has only the discrete kind, so state has no home in it.
2. **A state subscriber holds the current value of every key it covers, with no separate read and no `truncated`.** Today `truncated: true` tells a client that events were skipped and to fetch authoritative state again, which for state is a second mechanism bolted onto the first.
3. **An update can travel as a patch.** Today a resource watcher re-reads the whole resource on every change.
4. **Cloudflare's Workers Issues reach a live Claude Code session.** Today nothing bridges a webhook-only server to a local channel.
5. **Runtime validation comes from the spec's TypeScript, with no hand-kept second schema.** Today the official TypeScript SDK keeps zod schemas beside a copy of the spec's types, and a test to keep the two in step.
6. **The evidence holds at the wire, not only in this code.** The server passes Sri's conformance suites, and the Events types it is written against go to the working group in the spec's own style. Today no TypeScript types exist for Events, so an SDK's zod schemas have nothing to be checked against.

## Relationships

- **SEP-3415** ([PR #3415](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/3415)) is tracked at its head. It changed twice this month, so a change there is an input to this file.
- **Sri's conformance suites** ([conformance #521](https://github.com/modelcontextprotocol/conformance/pull/521), [#541](https://github.com/modelcontextprotocol/conformance/pull/541)) are open PRs, run from their branches until they merge.
- **`../lumenize/tasks/nebula-pre-alpha.md`** resumes when this is done.

## Current state

Nothing is built in this repo beyond its scaffold. What it builds on, and what becomes of each:

- **The spec's `schema/2026-07-28/schema.ts`** (about 98 KB, Apache-2.0). Carried over unchanged, with its license header, as the only core schema. The spec repo generates its `schema.json`, in JSON Schema 2020-12, and its reference docs from this file, using `typescript-json-schema`.
- **`merge-patch.ts` in `@lumenize/structured-clone`** (MIT). Carried over: an RFC 7396 implementation that passes the RFC's 15 Appendix A vectors, with a `diff` that produces the patch between two values.
- **`routeDORequest` in `@lumenize/routing`** (MIT). Carried over as the shape of the Worker: it forwards a request to its Durable Object intact, after an `onBeforeRequest` hook that can handle auth.
- **`json-rpc-mixin`** (Larry's, 0.0.5, May 2025, MIT). Adapted, not depended on. Its processing core shared across transports, its error-to-status mapping, and its tests of the JSON-RPC spec's examples carry over. Left behind: zod, `agents`, dispatch by reflection on `this[method]`, batches, and its WebSocket envelope.
- **Sri's conformance suites.** Carried over as the test oracle: server scenarios with a requirement-traceability file, and client scenarios stacked on them.
- **Cloudflare's MCP playground.** Used to smoke-test the server before Events exist. **Checkable claim:** it connects to a server that speaks only the 2026-07-28 handshake. `cloudflare/agents` 0.28.1 ships both the v1 MCP SDK (1.32.1), which speaks only the older handshake, and the v2 client (2.3.1). If the claim is false, the build either adds the older `initialize` handshake or skips the playground.
- **Cloudflare's API MCP server**, `https://mcp.cloudflare.com/mcp`. Used as the upstream. On 2026-10-15 its `events/list` offered one type, `cloudflare.alert.workers_observability_real_time_issue`, by webhook only, and `events/poll` and `events/stream` answered `-32601`. A session needs an API token with Notifications Write and Workers Observability Write, plus `account:read` or `user:read`.
- **Claude Code's channel contract.** A server that declares `claude/channel` pushes `notifications/claude/channel` with a `content` string and string-valued `meta`. The documented webhook receiver is adapted into the local channel server.
- **Peter's TypeScript SDK branches** (`events-spec-align`, `events-bufferemits-and-examples`). Left behind, on his own recommendation. Their stress examples (Slack, Kubernetes, Stripe, Gmail and others) show which upstreams the design was tried against.
- **The subscriptions in `@lumenize/resources`.** Left behind: the package is UNLICENSED, and it sends the full value on every change.

## Design intent

**The server speaks only as much MCP as Events needs.** It answers every request with `application/json`, which the 2026-07-28 HTTP transport allows, so it never opens an SSE stream. Of the core methods it needs `server/discover`, `tools/list` and `tools/call`, plus `resources/list` and `resources/read` for the one resource-backed type. State is read and written through tools, so nothing ties it to resources. The legacy `resources/subscribe` path is left out, along with any method the spec requires that this prototype does not need. For this server's clients, webhook and poll do the work a WebSocket push used to.

**Each Durable Object is a peer: an MCP server and an MCP caller in one class.** Two instances of the class subscribe to each other's state, so one demonstration exercises both sides of a webhook subscription: subscribing, refreshing and verifying on one side, delivering and coalescing on the other.

**A request reaches the Durable Object intact.** The Worker forwards each HTTP request to the object that owns the server, the way `routeDORequest` does, after an edge hook handles auth. The object answers from a table of handlers keyed by JSON-RPC method name, such as `events/subscribe`, so the table can later be bundled into a scoped mesh node.

**The state contract lives inside the SEP's own shapes wherever they suffice.** Event types, occurrences, cursors and delivery modes are the SEP's. The prototype adds to the wire only what those cannot carry, and keeps a list of every addition, because that list is what the working group decides on. Three are expected now: the contract declaration in `events/list`, a resync-complete signal, and what a null cursor means for a `state` type, below.

**A `state` occurrence names a key and a version.** The key is the one thing it updates: a note's id, a task id, an issue, a resource URI. Here is a first guess at an update to a watched note, with the state fields inside the SEP's `data`:

```json
{
  "eventId": "note-42@7",
  "name": "example.note.updated",
  "timestamp": "2026-10-16T19:02:11.000Z",
  "data": {
    "noteId": "note-42",
    "version": "7",
    "baseVersion": "5",
    "patch": { "done": true }
  },
  "cursor": "c-1931"
}
```

Versions 6 and 7 landed while the subscriber held 5, so it gets one patch from 5 to 7. The field names inside `data` are this prototype's first guess.

**Resync is ordinary delivery.** A subscription that starts without a cursor, or with one the server can no longer serve, first receives the current value of every key it covers, then a resync-complete signal, then changes. After the signal the client drops any key it holds that was not sent, the way a Kubernetes client relists. So a state type never sets `truncated`. For a `message` type, the SEP's `events/poll` reads a null cursor as "start from now" and returns no events. For a `state` type the same null cursor starts a resync, which is one of the additions listed above.

**Poll is the browser's delivery mode, and the SEP specifies all of it.** Each `events/poll` request is self-contained: event name, arguments and cursor in, then events, a new cursor, `hasMore` and `nextPollMs` out, with no state the server owes any client. That lets a Durable Object hibernate between polls. The SEP asks a client to apply a configurable floor to `nextPollMs`, by default 1000 ms. The demo keeps that default, which reads plainly as polling.

**Each key coalesces to its newest version.** While a webhook endpoint is down, or between polls, the server keeps one pending update per key.

**A patch goes out only when its base is still held.** The server keeps each key's recent versions for a bounded window. When the subscriber's version is inside that window and the content is JSON, the update is an RFC 7396 merge patch from it. Otherwise it is the full value, and a deleted key is a tombstone. The window is small on purpose, so tests exercise both paths. Merge patch replaces an array whole, so the savings are measured on realistic shapes.

**The `state` types answer questions 2 and 6.** Each sits beside an ordinary `message` type.

- **A tool-backed type**, like the note above, carries the demo and shows that the contract owes nothing to resources.
- **`mcp.resource.updated` answers question 2 by name**, built in the child that builds Events and kept out of the demo. The same notes are served as resources, listed by `resources/list` under `https://` URIs that carry the note's id, such as `https://events-demo.example/notes/note-42`, and read with `resources/read`. One subscription watches one URI, as the SEP's `{uri}` argument implies. Two kinds of key on one contract is the evidence that resources are only an example.
- **`mcp.task.updated` answers question 6**, with the name and argument the SEP floats. Its task state comes from the smallest real slice of the Tasks extension. **Checkable claim:** one tool run as a task is enough to drive `mcp.task.updated`, without the rest of the Tasks extension.

The tool-backed type's subscription covers every note, so it is the one that exercises resync and tombstones, while a resource subscription covers one. Whether a subscription naming a resource template should cover a whole collection is a question this prototype puts to the working group rather than builds.

**For a web resource, HTTP already does the polling, and the prototype measures where Events still earns its place.** The spec says a server should use an `https://` URI only when the client can fetch the resource directly from the web. So the Worker also answers a plain `GET` on each note's URI with an `ETag`, and a conditional `GET` with `If-None-Match` gets `304 Not Modified` and no body when nothing changed. For one web resource, that duplicates `events/poll`. Events still earns its place in three cases, and the report says which held up:

1. **Push.** A webhook needs no polling at all; poll exists for a client that cannot receive one.
2. **Collections.** A conditional `GET` checks one resource. It cannot say that a note appeared or was deleted, which one `events/poll` over every key does.
3. **Resources that are not on the web.** Most MCP resources are not fetchable over HTTP, and `resources/read` has no conditional form. Core MCP's only change signal for them is `notifications/resources/updated` on a held `subscriptions/listen` stream, plus an optional `lastModified` annotation.

**Issues reach Claude Code through a relay on Cloudflare.** A Durable Object subscribes to Cloudflare's server by webhook, verifies each delivery's Standard Webhooks signature, and refreshes the subscription from an alarm before its `refreshBefore`. It re-serves the issues by `events/poll` to a local channel server, which pushes each one into the Claude Code session. The relay offers them twice: as a `message` type, one occurrence per delivery, and as a `state` type keyed by issue, so a recurring issue is one entry that updates. **Checkable claim:** deliveries for a recurring issue carry an identifier that stays the same across recurrences. If they do not, the relay offers only the `message` type.

**The wire is plain JSON unless both ends declare otherwise.** Structured-clone is an optional preprocess callback, and with none registered, values travel as plain JSON. Merge patch runs downstream of whichever encoding is in use, and a server uses structured-clone only with a client that declared it.

**The spec's TypeScript is the only schema, and JSON Schema generated from it is what validates.** The core validates against the spec's committed `schema.json`. The Events types are written as `events.ts` in the spec's style, and `events.json` is generated from it with the spec's own generator and committed, with a check that fails when it is stale. `@cfworker/json-schema` validates at runtime by interpreting the schemas, so nothing generates code inside a Worker and nothing builds in the dev loop.

**No Durable Object holds a connection open.** Subscriptions, cursors and version windows live in Durable Object SQLite storage, and TTLs run on alarms, so every object can hibernate between deliveries. The server offers webhook and poll, and advertises only those two in `events/list`. **Checkable claim:** Sri's server suite skips its push scenarios for an event type that does not advertise push.

## Constraints

- **SEP-3415 at its PR head**, extension id `io.modelcontextprotocol/events`, which requires core revision `2026-07-28`.
- **The 2026-07-28 HTTP transport.** Each POST carries one JSON-RPC message. A request carries the `MCP-Protocol-Version` and `Mcp-Method` headers, and an events request also carries `Mcp-Name`, which SEP-3415 began requiring on 2026-10-16. An unknown method gets `404` with error `-32601`, and an accepted notification gets `202` with no body.
- **A webhook between two instances goes out through the public internet.** Without the `global_fetch_strictly_public` compatibility flag, a Worker's `fetch()` to a hostname in its own zone goes to the zone's origin and skips Workers, and on `workers.dev` it fails with error 1042. With the flag it comes back through Cloudflare's front door to the same Worker, which routes it to the other instance.
- **Licenses.** This repo is MIT. The spec's `schema.ts` and `schema.json` are Apache-2.0, so they keep their notices and get an attribution entry. Nothing imports `@lumenize/resources`, which is UNLICENSED.
- **Lumenize's ADRs do not bind.** The repo chose ADR-001's principle on its own, per CLAUDE.md § *Lumenize's ADRs do not bind here*. The rules CLAUDE.md imports or points to apply, among them `critical.md`, `workflow.md` § *No build during development*, `durable-objects.md`, `raw-comm.md` and `testing.md`.
- **Claude Code channels are a research preview.** A custom channel runs only under `--dangerously-load-development-channels` in an interactive session, on the earlier handshake (`MCP_PROTOCOL_NEGOTIATION=legacy`, which applies to the whole session).
- **Webhook delivery needs an authenticated subscriber.** The SEP lets an unauthenticated server offer poll and push, but not webhook.

## Future state

- **If the working group adopts a `state` contract**, Sri's SDK implementation adds it, and the Events types here become what SDK schemas are checked against.
- **Lumenize's own subscriptions can take the same conventions:** cursor and resume, per-key coalescing, tombstones, and patches from a held version. Lumenize means to adopt this server's shape too: `routeDORequest` carries the request to the object, `onBeforeRequest` handles auth at the edge, and the handler table is bundled for scoped mesh nodes. ⚠️ Design consideration: keep the state machinery free of the HTTP transport, so Lumenize can run it over Mesh.
- ⚠️ Design consideration: the `list_changed` family in question 2 is state too, with the list as its value, so nothing here should assume a `state` key is always a URI or an id.
- ⚠️ Design consideration: question 5, one subscription spanning several event names under one cursor, would let a client watch notes and tasks together. Leave room for it.
- ⚠️ Design consideration: a Lumenize Resource is closer to this file's state than to an MCP resource. When this ports to Lumenize's data plane, state maps onto Lumenize Resources, and an MCP resource is only one way of exposing them.

## Open questions

1. **What the contract declaration is called, and where it sits.** mib00038's values, `message` and `state`, are on the thread already. Options: a new descriptor field (`semantics`, `contract` or `consumption`), or a key under `_meta`. Recommendation: a descriptor field named `semantics`, the word the thread already uses, with mib's values. Gates the Events types.
2. **Which Lumenize MIT packages to depend on.** Candidates: `@lumenize/structured-clone` for merge patch, and `@lumenize/routing` for `routeDORequest`. Recommendation: depend on both. They are the code Lumenize will build on, and a copy would fork it. Gates the build's first phase.
3. **Task state from a real slice of the Tasks extension, or a stand-in, and in which child.** Recommendation: the real slice, in the build's Events phase, because a stand-in proves nothing to the people working on tasks. Gated by the checkable claim in § *Design intent*.
4. **How a webhook subscriber authenticates to this server.** Options: MCP's OAuth flow, a static bearer token, or whatever Sri's server scenarios use. Recommendation: match the conformance suites, once it is known what they use. Gates the build's Events phase, where the two instances first subscribe to each other.
5. **Whether the Issues relay stays.** Options: a child of its own after the demo, or dropped from this prototype. Recommendation: keep it as a child after the demo. The build's peer already has the subscribing side, so the relay is mostly that code pointed at Cloudflare, plus the local channel server.

## Decisions

| Decision | Rejected alternative — why |
|---|---|
| An independent implementation, written from the spec | Inside the official TypeScript SDK: Sri owns that work, and the SDK keeps zod beside the spec's types. On `cloudflare/agents`: its state is one value per agent over its own WebSocket messages, which never touches `events/*`. Porting Peter's branch: he recommended starting fresh. |
| A minimal MCP: JSON responses only, and only the methods Events needs | Full Streamable HTTP with SSE: nothing here needs it, and a held stream keeps a Durable Object awake. |
| Webhook and poll delivery only | Push through `events/stream`: it needs the SSE stream this server never opens, keeping it open or reconnecting it is wiring nothing else here needs, and a Claude Code channel cannot consume it anyway. |
| State is read and written through tools (Larry, 2026-10-16) | Resources as the state: MCP has no method that writes a resource, and resources would invite the reading that the contract is about resources. |
| One resource-backed type, `mcp.resource.updated`: one subscription per URI, notes listed by `resources/list` under `https://` URIs the Worker also serves to a plain `GET` (Larry, 2026-10-16) | No resource type: question 2 names resources, so a reviewer would look for one. A resource template: it gives a URI's shape but not which notes exist, which a per-URI subscriber needs. A subscription naming a template: it tests the SEP's choice of arguments rather than the contract, so the report raises it instead. A custom scheme such as `note://`: an `https://` URI lets the same note be polled with plain HTTP, the comparison the report needs. |
| The demo's browsers poll `events/poll` (Larry, 2026-10-16) | A WebSocket to the browser: it sits outside MCP, it would need explaining in a demo about Events, and it would leave poll mode unexercised. |
| Validation from JSON Schema generated from the TypeScript, checked by `@cfworker/json-schema` (Larry, 2026-10-16) | Typia: its transform must run in vitest and `wrangler dev`, or as a generate step in build and deploy. `@lumenize/ts-runtime-parser-validator`: it needs dynamic Workers or a build step. Hand-written checks: a second copy of the schema. |
| Plain JSON on the wire by default, with structured-clone as an opt-in preprocess | Structured-clone by default: the working group could not evaluate a wire it does not speak. |
| Adapt `json-rpc-mixin`'s core | Depend on it: it brings zod and `agents`, dispatches by reflection, and handles batches and a WebSocket envelope that MCP does not use. |
| A master plan holding the design intent, with each child task file written when it starts | One task file with five phases: the demo iterates back into the build, and its result is a recording rather than tests, which `/build-task`'s phases in order cannot carry. |
