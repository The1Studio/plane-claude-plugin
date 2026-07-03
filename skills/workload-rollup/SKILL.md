---
name: workload-rollup
description: Explains how Plane's workload estimated-hours, due date, and progress % roll up from sub-items to a parent work item, why parent estimates are read-only, and how to read rollup data via the API or MCP. Use when a user asks about estimated hours, due dates, or progress % on a work item that has sub-items, or when a workload estimate write returns a PARENT_HAS_CHILDREN error.
---

# Plane Workload — Parent Rollup Semantics

> **Scope:** this feature ships in The1Studio's self-hosted Plane fork (the `workload`
> Django app). Plane Cloud — the hosted MCP server this plugin wires to by default — does
> not have it. This skill is for deployments where the MCP/API endpoint has been pointed
> at a self-hosted instance running the fork; self-hosted support in this plugin itself is
> still on the [Roadmap](../../README.md#roadmap).

## Core rule: estimates live on leaf issues only

A work item that has sub-items ("countable" children — see below) is a **parent**. Parents
do not carry their own estimate:

- **Estimated hours, due date, and progress %** are all *derived* from the parent's
  countable leaf descendants — they are not stored on the parent.
- The estimate field is **read-only in the UI** for a parent.
- `PUT` on a parent's workload estimate returns **HTTP 400**:
  ```json
  { "error": "<human message>", "error_code": "PARENT_HAS_CHILDREN" }
  ```
  If you're driving the API/MCP directly and see this error, don't retry the write —
  redirect the user to set estimates on the leaf (sub-)items instead.

A **leaf** is a countable issue with no countable children. Estimates can only be entered
on leaves.

## What counts as "countable"

An issue is countable (i.e. contributes to a parent's rollup, and can itself be a parent
or a leaf) when it is **not** deleted, **not** archived, **not** a draft, and its state
group is **not** `cancelled` or `triage`. Cancelled and triage sub-items are invisible to
this feature entirely — a parent whose children are *all* cancelled reverts to being a
plain leaf (its own estimate becomes editable again).

## Rollup math

For a parent, the rollup is computed over the full tree of countable descendants
(recursion depth capped at 10):

- **`hours`** — sum of estimated hours across countable leaf descendants only.
- **`done_hours`** — sum of estimated hours across countable leaf descendants whose state
  group is `completed`.
- **`percent`** — hours-weighted progress, `done_hours / hours`, as a `0..1` fraction.
  `null` when `hours` is `0` (no estimated leaves yet). **This is not the same as core
  Plane's sub-issue progress**, which counts completed vs. total *issues* regardless of
  estimate — a parent can show 60% workload progress while its core sub-issue progress
  bar shows a different percentage, because workload progress is weighted by hours, not
  issue count.
- **`due_date`** — the latest `target_date` across **all** countable descendants (not just
  leaves — an intermediate node with a due date but no estimate still counts here). `null`
  if no descendant has a due date.
- **`leaf_count`** — number of countable leaves that have an estimate (`hours > 0`).

A non-countable node (e.g. a cancelled intermediate item) prunes its entire subtree from
the rollup — descendants below it are excluded even if they'd otherwise be countable.

## Where rollups do — and don't — show up

- **Sidebar / spreadsheet estimate cell** — shows the rollup (`Σ 10h · 60%`) read-only for
  a parent, instead of an editable input.
- **Workload matrix** (`get_workload`) — counts **leaf issues only**. Parents never appear
  as their own row, and a parent's legacy/ignored estimate is never double-counted against
  its children's estimates.
- **Bulk estimates endpoint** — parent rows are **omitted** entirely (a parent looks like
  "no estimate" here, same treatment as the matrix), so a bulk export/import won't leak a
  stale parent estimate.

## Reading rollups

- **REST**: `GET /api/v1/workspaces/<slug>/workload-rollups/?issue_ids=<id1>,<id2>,…`
  returns a map of `{issue_id: rollup}` for the ids that are parents (leaf ids in the
  request are silently omitted from the response, not an error). Capped at 500 ids per
  request. The same endpoint is also mounted, unprefixed, under the app-internal API for
  the web client.
- **MCP**: use the `get_workload_rollups` tool exposed by `plane-mcp-server` (The1Studio's
  self-hosted MCP server) — this plugin's default hosted MCP does not expose it.

## v1 limitations to set expectations on

- **Stale until reload**: adding or removing a sub-item updates the parent's rollup on the
  backend immediately, but the web UI's displayed rollup only refreshes on the next page
  load — it does not live-update in the current tab.
- **Restricted guests see partial rollups by design**: a guest whose project scope doesn't
  include one of the parent's sub-items will see a rollup computed only over what they can
  see, which may under-count the true total. This is intentional (no cross-project data
  leak) — don't report it as a bug.
