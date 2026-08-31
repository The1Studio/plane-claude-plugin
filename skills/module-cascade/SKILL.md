---
name: module-cascade
description: Explains how completing or cancelling a Plane module can cascade that terminal status to every live member work item plus each member's full sub-item subtree, why a plain module status change never cascades from any client, the 100-item cap that is a refusal rather than a truncation, and how a work item already in a terminal group prunes its whole subtree instead of being traversed through. Use when updating a module to completed or cancelled, when a module status change leaves work items live and the user expected them to close, when a cascade preview returns over_cap true or a cascade-apply returns HTTP 400, or when a work item inside a terminal group appears to have live descendants left behind.
---

# Plane Module Cascade — Terminal Status Propagation

> **Scope:** this capability ships in The1Studio's self-hosted Plane fork (the
> `cascade_ext` Django app, backend branch `feat/module-cascade-terminal-status`,
> plan `plans/260828-module-cascade-terminal-status/`). Plane Cloud — the hosted
> MCP server this plugin wires to by default — does not have it. This skill
> applies to deployments pointed at a self-hosted instance running the fork;
> self-hosted support in this plugin itself is still on the
> [Roadmap](../../README.md#roadmap).

## Core rule: a plain module status change never cascades

Completing or cancelling a module can cascade that terminal status to **every
live member work item plus each member's full sub-item subtree** — but only as
an explicit, confirmed action. **A plain module status change never cascades,
from any client:**

- **Web UI** — the summary modal appears and **defaults to "only change this
  module"**. The user must opt into cascading in the modal.
- **MCP / API** — `update_module` defaults to `cascade=False`, which is a plain
  PATCH, byte-for-byte what the tool did before cascade existed. Work items in
  the module are left untouched.

So a user asking "complete the module" and expecting every open work item to
close is describing the cascade, not the plain update. When the intent is a
plain status change, the item's live members staying live is **correct
behaviour**, not a bug — only the terminal status moves.

## Only terminal groups cascade

The cascade is monodirectional in one direction: `completed` and `cancelled`
are the only statuses that cascade. `backlog`, `planned`, `in-progress` and
`paused` — or any unrecognized status string this tool coerces to `None` — are
plain PATCHes even with `cascade=True`; the flag never raises, so a caller may
set `cascade=True` once and reuse it across calls without surprise-cascading.

The doorway is the **validated** module status literal, never the raw
argument: `status="Completed"` coerces to `None` and cannot fire the cascade
branch. Cascading is keyed on the module-status literal (`completed` /
`cancelled`), not on a per-project renameable state group.

## MCP: preview first, then apply

Two MCP tools mirror the per-issue cascade pattern one level up:

- **`preview_module_cascade(workspace_slug, project_id, module_id, status)`** —
  read-only. `GET /api/cascade-ext/workspaces/<slug>/projects/<project_id>/modules/<module_id>/cascade-preview/?status=<completed|cancelled>`.
  Returns `{target_group, depth_capped, over_cap, cap, summary, items}`.
  `items` are the currently-eligible members plus descendants a matching apply
  would update, each with an `eligible` flag and a `reason` when disabled
  (`no_matching_state` / `no_permission` / `already_terminal` /
  `under_terminal_ancestor` / `not_in_module_tree` / `not_eligible`). An empty
  `items` list means every member plus descendant is already terminal (or there
  are none) — a plain `update_module(..., cascade=False)` is equivalent and
  there is nothing to cascade.

  Note the query parameter: the module preview takes **`status`** (a MODULE
  status), not the `group` the work-item preview takes. The two are
  deliberately not unified server-side.

- **`update_module(project_id, module_id, status=..., cascade=False)`** — with
  `cascade=True` **and** a terminal validated status, calls
  `POST /api/cascade-ext/workspaces/<slug>/projects/<project_id>/modules/<module_id>/cascade-apply/`
  with body `{"status": <terminal>}`. The module's new status and every
  currently-eligible member plus descendant are applied **in one transaction** —
  apply does not expect the module to have already been PATCHed. Omit
  `item_ids` (the documented headless path) and the server takes every
  currently-eligible item.

  The return value is the cascade-apply response:
  `{"module": "<id>", "status": "<terminal>", "updated": ["<uuid>", ...], "rejected": [{"id": "<uuid>", "reason": "..."}, ...]}`.
  `rejected` items are surfaced, not swallowed. Call `retrieve_module` afterward
  for the full module object.

**Set `cascade=True` only after the preview and a confirmation.** A headless
caller with no modal is exactly the population this opt-in protects: without
`cascade=True` the status change is safe and isolated; with it, one call moves
the module and every eligible descendant. Preview-then-confirm is the
headless analogue of the web modal.

Other fields set alongside a cascade still land: a rename like
`update_module(..., status="completed", name="Q3 (shipped)", cascade=True)`
applies the rename via the ordinary PATCH (status excluded — cascade-apply owns
the status move) and still returns the cascade-apply response.

## The 100-item cap is a refusal, not a truncation

`MAX_MODULE_CASCADE_ITEMS = 100` is a **refusal**, not a truncation:

- **Preview** — a module over the cap returns `over_cap: true` with an **empty
  `items` list** (the summary still reports `total_live` — e.g. 240).
- **Apply** — returns **HTTP 400** having written **nothing**: no work item
  states, and the **module's own status is not changed either**. The error
  payload names the cap:
  ```json
  { "error": "module exceeds cascade cap of 100 live items", "total_live": 240, "cap": 100 }
  ```

A headless caller reading that 400 as a transport failure will retry it — the
same apply, the same refusal. **Do not retry.** Treat the 400 as a decision
point: either cascade in batches out-of-band, or fall back to a plain
`update_module(..., cascade=False)` status change, which still works — the cap
refuses the **cascade**, not the module status update. The web UI's modal never
offers the cascade for an over-cap module, so this is the headless-only trap
this section exists to close.

## Behaviour change to the per-issue cascade: terminal groups prune

> Applies to the **existing per-issue cascade too** — both subjects share the
> one `cascade_ext` app.

A work item that is **already in a terminal group** now **prunes its entire
subtree** instead of being traversed through: nothing beneath it is listed,
walked, or changed. So a live sub-item under a cancelled parent is **left
live**, where it used to be swept into the parent's terminal group.

A preview therefore never lists `already_terminal` rows, and everything beneath
them is excluded. A caller previewing a parent whose children are terminal will
see the subtree under them disappear from `items`; the user-facing consequence
is that closing a terminal parent no longer closes its grandchildren. Reconcile
those grandchildren **individually**, one terminal work item at a time.

## Where the cascade does NOT apply

- **Plain status PATCHes** — any `cascade=False` call, and any web-UI "only
  change this module" selection. Nothing below the module moves.
- **Non-terminal statuses** — `backlog`, `planned`, `in-progress`, `paused`.
- **Servers without the fork** — the `cascade_ext` app 404s on upstream Plane /
  Plane Cloud, or a server predating the backend branch.