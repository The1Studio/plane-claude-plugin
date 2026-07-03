# Plane plugin for Claude Code

Integrate [Plane](https://plane.so) — the open-source project management tool — directly into Claude Code. Create work items, triage intake, generate standups, and track cycles without leaving your terminal.

## What's included

- **MCP server** — connects Claude Code to Plane over OAuth (Plane Cloud). All Plane tools (work items, cycles, modules, projects, intake, pages, and more) are available to Claude automatically.
- **Skills** — bundled guidance that teaches Claude the semantics behind Plane features that aren't obvious from the API surface alone. See [`skills/`](./skills):
  - [`workload-rollup`](./skills/workload-rollup/SKILL.md) — how estimated hours, due dates, and progress % roll up from sub-items to a parent work item (self-hosted fork only).

That's it. No wrappers, no proxy scripts — just a thin plugin that wires Claude Code into Plane's hosted MCP server, plus the skill docs Claude needs to use it well.

## Install

```bash
claude plugin install plane
```

Or, for local development:

```bash
claude --plugin-dir /path/to/plane-claude-plugin
```

On first use, Claude will open a browser window to authenticate with Plane. Sign in with the workspace you want to work against.

## Requirements

- Claude Code v2.1 or later
- A [Plane Cloud](https://app.plane.so) workspace (self-hosted Plane is not supported in this version — see roadmap below)
- Node.js isn't required; the plugin uses Claude Code's native HTTP MCP transport

## Usage

Once installed and authenticated, ask Claude anything about your Plane workspace in natural language:

- "What's assigned to me in the Frontend project?"
- "Create a bug: login flash on mobile Safari, blocking sign-in"
- "Summarize my activity in Plane since yesterday for standup"
- "Walk me through the intake inbox for Backend"
- "How's the current cycle for Mobile looking?"

Claude picks the right Plane MCP tools automatically and asks for confirmation before writes.

## Roadmap

- Bundled skills for common workflows (new-issue, standup, triage, cycle-status)
- PAT / API-key auth (for CI and automation)
- Self-hosted Plane support
- Subagent for feature decomposition (initiative → epics → work items)

## Feedback

File issues at [github.com/makeplane/plane-claude-plugin](https://github.com/makeplane/plane-claude-plugin/issues).
