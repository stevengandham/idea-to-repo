# Idea-to-Repo Builder — Design Spec

Date: 2026-09-30

## Purpose

A Claude Code skill that takes a product idea from conversation, helps the
user choose open-source projects to integrate for each part of the stack,
builds a full working implementation of the idea against that stack, and
publishes the result as a new public GitHub repository with a generated
README.

This is subsystem #2 of a larger two-part concept. Subsystem #1 (a
discussion platform where people propose and develop product ideas
together) is a separate, independently-scoped project and is out of scope
for this spec. This tool is designed to work standalone today (idea
supplied directly in chat) and could later take its input from subsystem
#1 instead, without changing its own phases.

## Why a skill, not a standalone app

The workflow requires human approval at two points (choosing OSS
integrations, approving the build plan) and needs GitHub search plus a
local build/test loop. Claude Code already provides the chat interface,
tool-use (Bash/gh CLI, file read/write), and execution environment for
this. Building a standalone hosted version would mean re-implementing a
chat UI, an LLM orchestration backend, and a sandboxed execution
environment — a much larger and riskier project with no added benefit for
the current single-user use case. A standalone version remains a possible
future step (swap the orchestration layer), not a rewrite of this spec.

A background `Workflow` (autonomous multi-agent orchestration) was
considered and rejected: `Workflow` runs subagents without pausing for
human choices mid-run, which conflicts with the two required approval
gates in this flow.

## Phases

1. **Ideate** — chat-based brainstorm (same questioning style as the
   `brainstorming` skill) to nail down what the product idea is, its core
   features, and rough scope.
2. **Decompose into integration slots** — Claude proposes the categories
   of off-the-shelf OSS the idea needs (e.g. backend framework, auth,
   database, frontend UI), based on the ideation output.
3. **Search & present candidates** — for each slot, search GitHub (via
   `gh search repos` / `gh api`), rank by a blend of stars and recent
   activity, and show the top 3 candidates: name, description, stars,
   last-updated, and license.
   - Permissive licenses (MIT, Apache-2.0, BSD) are marked with a ✓.
   - Any other license (GPL/AGPL family, no license, unrecognized) is
     marked with a ⚠ and a one-line note on why it's flagged.
   - The user can also supply their own repo for a slot directly,
     bypassing search.
4. **User picks** — user selects one repo per slot, or explicitly skips a
   slot.
5. **Plan** — Claude writes a build plan (idea summary + chosen stack +
   architecture + file layout + proposed repo name), presents it in chat,
   and waits for explicit user approval before writing any code.
6. **Build** — on approval, Claude implements a full working version of
   the idea against the chosen stack, locally, following TDD (tests
   first, watch them fail, implement, re-verify).
7. **Publish** — Claude creates a new **public** GitHub repository (name
   from the plan, overridable at approval time) under the user's account
   via the already-authenticated `gh` CLI, and pushes the built code plus
   a generated README as the initial commit. No PR step — it's a brand
   new repo, so there's nothing to review against.

There is no approval checkpoint between build and publish — approving the
plan in phase 5 is sufficient authorization to build and publish.

## GitHub search mechanics

- Search is run per slot using `gh search repos` / `gh api`, with a query
  built from the slot's category plus relevant idea context.
- Candidates are ranked by a blend of star count and recency of activity
  (e.g. deprioritizing repos with no pushes in the last ~2 years) to avoid
  surfacing abandoned projects.
- Top 3 results per slot are shown with license flagged as above.

## Error handling

- **No good candidates for a slot**: tell the user, offer to broaden the
  search query, or let them skip the slot or supply their own repo.
- **`gh` not authenticated or rate-limited**: surface the error directly;
  do not retry silently.
- **Build failures** (tests fail, an integration doesn't work as
  expected): stop and surface the failure to the user. No automatic
  retry loops — broken code is never pushed.
- **Repo name collision on GitHub**: ask the user for a different name
  rather than guessing a suffix.

## Output contract

- Repo is created public.
- README is auto-generated and must include:
  - What the project does (idea summary).
  - Which OSS projects were integrated, with links and license.
  - Setup/run instructions.
  - A note that the project was scaffolded via this tool.
- Code and README are pushed together as the initial commit.

## Out of scope for this spec

- Subsystem #1 (the discussion/collaboration platform for developing
  ideas with other people).
- A standalone (non-Claude-Code) version of this tool.
- Any deploy/hosting step beyond creating and pushing to the GitHub repo.
