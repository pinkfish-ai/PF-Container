# CLAUDE.md

This is the **PinkConnect + MCPfarm self-host install repo**. When the user asks you to install, deploy, set up, or stand up Pinkfish in their AWS account, follow [`claude-setup.md`](./claude-setup.md) — it's your orchestrator playbook (ask the human 6 things up front, then drive [`install.md`](./install.md) step by step).

Treat `install.md` as the single source of truth for the install procedure. `claude-setup.md` tells you *how to behave* (which questions, what order, what to verify per phase); `install.md` is the actual *runbook* of commands. Don't merge content from other files unless `claude-setup.md` tells you to.

When the user asks to **tear down** what was installed, follow [`teardown.md`](./teardown.md) — smoke section by default.

Reference docs (don't read in full unless you need to look something up):
- [`docs/gotchas.md`](./docs/gotchas.md) — non-obvious behaviors to know about. Read in full once before starting your first install.
- [`docs/troubleshooting.md`](./docs/troubleshooting.md) — symptom → cause + fix when something breaks.
- [`docs/parameter-reference.md`](./docs/parameter-reference.md) — CFN parameter meanings.
- [`docs/alternate-components.md`](./docs/alternate-components.md) — swap-out playbook (Atlas, EKS, Cloudflare, etc.).

The [`wip/`](./wip/) directory contains the production install profile, which is **work in progress and not customer-ready against bundle v0.2.0**. Don't drive it unless the user explicitly asks for production AND acknowledges the WIP status.

Current bundle version is in [`VERSION`](./VERSION); changelog in [`RELEASE-NOTES.md`](./RELEASE-NOTES.md).
