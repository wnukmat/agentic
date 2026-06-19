# agentic

Personal, **project-agnostic** knowledge base for agentic (Claude Code) workflows — reusable
patterns, playbooks, configs, and templates that apply across projects. When something proves
useful in one project and isn't specific to it, it gets generalized and lands here so the next
project can reuse it.

Clone on any machine; the docs are meant to be read by a human or pulled into a Claude session
as reference.

## Layout
```
patterns/    reusable architecture/design patterns (the "how it's shaped" docs)
playbooks/   step-by-step operational guides (the "how to run it" docs)
configs/     reusable settings — permission allowlists, hooks, settings.json snippets
templates/   copy-paste scaffolding to drop into a new project
```

## Index
### patterns/
- [fleet-orchestration.md](patterns/fleet-orchestration.md) — operate multiple machines through
  Claude Code from a single point of management (manager/worker via SSH + headless `claude -p`,
  optional Telegram layer, shared-memory substrate). Includes a per-project bindings template.

## Conventions
- One concept per file; generic terms (Box A/B, manager/worker), no project specifics in the doc body.
- Each pattern ends with a *Per-project bindings* block — copy that into the consuming project's repo
  and fill in the concrete hostnames/paths there, keeping this file reusable.
- Project-specific deployment notes live in the **project's** repo, linking back here for the pattern.
