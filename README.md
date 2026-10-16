# MCP Events prototype

A working prototype for the MCP Events extension ([SEP-3415](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/3415)), built to answer two questions the SEP leaves open:

- Can watching state, such as a resource or a task, ride the same `events/*` methods as consuming discrete events, with only a contract declared per event type? This is the SEP's Open Questions 2 and 6.
- What does it take to get webhook-delivered events, such as Cloudflare's Workers Issues, into a live agent session?

It is an independent implementation, not built on an official SDK, and it has two side deliverables:

- TypeScript types for the Events extension, in the style of the spec's `schema.ts`. An SDK's runtime schemas can be checked against them.
- Runtime validation derived directly from the spec's TypeScript schema, with no second schema to keep in sync.

Status: design. Nothing runs yet.

License: MIT.
