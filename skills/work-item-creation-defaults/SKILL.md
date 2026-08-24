---
name: work-item-creation-defaults
description: Explains how a newly created Plane work item gets an assignee and a due date when the caller did not supply them, why an omitted field and an explicitly empty one behave differently, and how to create a work item that is deliberately unassigned or deliberately undated. Use when creating a work item, when a user asks why a new item came out assigned to them or due today, or when a caller wants a work item with no assignee or no due date.
---

# Plane Work-Item Creation Defaults

> **Scope:** this behaviour ships in The1Studio's self-hosted Plane fork (the
> `issue_defaults_ext` Django app). Plane Cloud — the hosted MCP server this plugin wires
> to by default — does not have it, and a work item created there comes out unassigned and
> undated as before. This skill applies to deployments pointed at a self-hosted instance
> running the fork; self-hosted support in this plugin itself is still on the
> [Roadmap](../../README.md#roadmap).

## Core rule: absent and empty are different

When a create request **does not carry** a field, the server fills it:

- **no assignee field** → assigned to the authenticated caller
- **no `target_date`** → today, in the **caller's own timezone**

When a create request carries the field **explicitly empty**, the server leaves it alone:

- **`assignees: []`** → deliberately nobody; the work item is created unassigned
- **`target_date: null`** → deliberately no due date

This is the one thing to get right, because getting it wrong is silent. Filling in an
empty value for an argument the user did not mention does not produce the same result as
leaving the argument out — it opts them out of the defaults, with no error anywhere.

**So: only send a field the user actually specified.** Do not helpfully initialise
`assignees` to `[]` or `target_date` to `null` when building a create call.

## Answering "why is this assigned to me / due today?"

Because the user did not say otherwise, and the fork fills both. That is intended
behaviour, not a bug, and it is undone the same way it was applied — set the field.

To create a work item **with no assignee**, pass an empty assignee list. To create one
**with no due date**, the answer depends on the transport:

- **MCP `create_work_item`** — there is no way to say it on the create call. The SDK
  behind the tool strips `None`-valued arguments before the request, so
  `target_date=None` is indistinguishable from omitting it and you still get today.
  Create the item, then clear the field:
  `update_work_item(..., clear=["target_date"])`.
- **Node SDK** — pass `target_date: null` on create; `null` survives serialisation there.

`assignees: []` works on every transport, because an empty list is not a null.

## Precedence and the cases where nothing is assigned

The creator is a **fallback**, not an override:

1. If the project has a **`default_assignee`** configured in project settings, that person
   is assigned — unchanged from before this feature, and it applies even when the caller
   sent an explicitly empty assignee list.
2. Otherwise the **creator**, but only when the assignee field was absent.
3. If the creator is **not an active project member** at `role >= 15`, nobody is assigned.
   The item is created unassigned rather than assigned to someone who cannot see it.

## Due-date detail worth knowing

- **"Today" is the caller's day**, resolved from their Plane profile timezone — not the
  server's UTC date. A user at UTC+7 creating an item at 06:00 local gets their own date,
  not yesterday's.
- **A future `start_date` wins.** If the caller sets a start date in the future and no due
  date, the due date becomes the **start date**, not today. This exists so a request that
  used to succeed cannot start failing on the server's "Start date cannot exceed target
  date" validation.

## Where the defaults do NOT apply

- **Updates.** Editing an existing work item never re-fills anything. Clearing a due date
  makes it stay cleared.
- **Intake submissions.** Items submitted through a project's intake queue get neither
  default — a submitter is frequently not a project member, and dating a triage queue
  would misrepresent it.
- **Bulk import.** Anything written directly to the database, such as a migration script,
  bypasses this entirely.

Drafts, sub-work-items and epics **do** get the defaults; they go through the same create
path as a normal work item.
