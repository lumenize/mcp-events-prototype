# MCP Events prototype

A working prototype for the MCP Events extension, [SEP-3415](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/3415), MIT-licensed. It has two goals. The first is a `state` delivery contract, which covers watching a resource or a task (the SEP's Open Questions 2 and 6), running over the same `events/*` methods as ordinary `message` events. The second is Cloudflare's Workers Issues events arriving in a live Claude Code session through a channel. The plan lives in `tasks/`.

This repo sits next to `../lumenize` and borrows its guidance, because Larry's rules, skills and memories live there. This file says which parts apply here, so nothing specific to Lumenize leaks in.

## Imported from lumenize

These load every session:

@../lumenize/.claude/rules/critical.md
@../lumenize/.claude/rules/calibration.md
@../lumenize/.claude/rules/workflow.md
@../lumenize/.claude/rules/prose-voice.md

They were written for Lumenize. Read them with these exceptions:

- **`critical.md`** applies, except that docs do not live in `/website/docs/` here. This repo's docs are its README and the field reports it posts to the working group.
- **`calibration.md`**: the habits it corrects apply everywhere, though its evidence is all from Lumenize. §11 (Mesh shims), §12 (Vue templates), §14 (dominion) and §15 (Studio's agent) are about Lumenize code and have nothing to fire on here.
- **`workflow.md`** applies, except for its ADR index, its npm scripts and its Releases section, which belong to Lumenize.
- **`prose-voice.md`** applies to everything written here, and most of all to anything posted where the MCP community reads it. It loads only on matching paths in lumenize, which is too late for drafting, so it is imported here.

## Lumenize's ADRs do not bind here

The ADRs are commitments for Lumenize's codebase. This repo's readers are the MCP community, and its job is to show the SEP working, so the SEP and the spec's own schema set the constraints. When an ADR's reasoning bears on a choice here, argue it on its merits in the task file's Decisions table, never cite it as binding.

ADR-001's principle holds here, but because this repo chose it (Larry, 2026-10-16), not because the ADR binds. This is an independent implementation, not built on the official SDK. The spec's `schema.ts` is its only schema, and runtime validation is derived from those TypeScript types, with no zod and no hand-kept second copy. Showing that working is one of the repo's deliverables. The other is TypeScript types for the Events extension, written in the spec's style, which the SDK's implementation can test its zod schemas against.

`@lumenize/resources` is UNLICENSED, so nothing here imports it. The MIT packages (`@lumenize/structured-clone`, `@lumenize/mesh`) are allowed. Whether to depend on them is a decision for the task file, teed up for Larry.

## Path-scoped rules

Lumenize's path-scoped rules cannot be linked in. A linked rule from outside the repo loads only if it has no `paths:` field. So `.claude/rules/` holds short local rules that load when you touch a matching file and say which lumenize rule to read.

## Skills

Linked from lumenize: `/write-task`, `/review-task`, `/build-task` and `/refactor-efficiently`. The Cloudflare skills come from `~/.claude/skills`.

- When a linked skill names a repo path that does not exist here, such as `tasks/README.md`, read it from `../lumenize/`. New files the skill writes, a task archive included, go in this repo.
- Run the prose checker as `node ../lumenize/scripts/check-prose.mjs <file>`.
- `/review-task`'s product lens is not Lumenize's `docs/vision/`. Here it is the SEP, the working group's charter, and what earns standing with the working group. The task file's design intent states it.
- Not linked: `/live`, `/nightly-pass` and `/release-workflow`, which drive Lumenize's systems; the doc-example skills, which are for Lumenize's website; and `/task-management`, because work here is always task-file-first.

## Sibling repos

- `../modelcontextprotocol` is the fork for SEP PRs (origin `lumenize/modelcontextprotocol`, with `upstream`).
- `../mcp-typescript-sdk` is the fork for SDK PRs (origin `lmaccherone/mcp-typescript-sdk`). It is far behind upstream, so sync it before branching.
- `../lumenize` is where the guidance above lives. Do not edit it from here unless Larry asks.

In a sibling fork, follow that repo's own CLAUDE.md and tooling. The TypeScript SDK uses pnpm, and `critical.md`'s npm-only rule is for repos Larry owns, not for upstream forks.

This repo's remote is the public `lumenize/mcp-events-prototype`. Pushing publishes, so push only when Larry asks.
