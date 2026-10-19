# Exercise 0 — Getting started: your AI-native CAP setup

In this exercise you set up everything you need to build a CAP application **by directing an AI**
rather than typing it by hand. You'll install the CAP skills into **Claude Code**, verify your
development environment with an AI skill, and get the **xstravels seed** project ready to grow.

> [!NOTE]
> **The idea of this session.** You won't copy-paste a finished app. You'll *direct* Claude Code —
> grounded in the CAP team's own skills — to build xstravels one small, reviewable stage at a
> time. The "latest and greatest" CAP features (`@assert` constraints, `@flow.status` state
> machines, HCQL, vector-embedding search, MCP) emerge naturally as you reach each stage. You stay in
> control: every step is a diff you review before moving on.

Throughout the exercises, a line beginning with 👉 is something **you actually do**. Everything
else is context to read.

## Prerequisites

- **Git** and **Visual Studio Code** (pre-installed on the TechEd notebook).
- **Claude Code**, pre-configured on the TechEd notebook.
- **Node.js** and **npm**.
- The latest **`@sap/cds-dk`** (installed below).

### Versions this session was built and tested against

| Component | Version |
|---|---|
| Node.js | 24.x (LTS) |
| `@sap/cds-dk` (global CLI) | 10.1 |
| `@sap/cds` (runtime) | ^10 |
| `@cap-js/sqlite` | ^3 |
| `@cap-js/cds-test` | ^1 |
| CAP skills | public `github.com/capire/skills` (session release) |
| CAP MCP docs server | `@cap-js/mcp-server` (`cds-mcp`) |
| AI tool | Claude Code |

> [!NOTE]
> Use these as the baseline. If a newer `@sap/cds-dk` major is available the session still works;
> these are the exact versions we verified.

## Open the session repository

👉 Clone this repository and open it in VS Code:

```text
https://github.com/SAP-samples/teched2026-AD162.git
```

👉 Open a terminal in VS Code (**Terminal > New Terminal**).

## Install the latest CAP development kit

👉 Install (or update) the CAP development kit globally:

```bash
npm i -g @sap/cds-dk
```

👉 Check it:

```bash
cds --version
```

> [!IMPORTANT]
> **Use the latest `@sap/cds-dk`.** The guided-dev flow relies on current CLI features (e.g.
> `cds add data --keys-only`). An outdated kit generates stale project layouts and misses flags,
> which quietly derails later stages.

## Install the CAP MCP server
The CAP MCP server is published in github
[`github.com/cap-js/mcp-server`](https://github.com/cap-js/mcp-server).

👉 In the terminal run:

```text
claude mcp add cds-mcp -- npx -y @cap-js/mcp-server
```

## Install the CAP skills into Claude Code

The CAP skills are published as a plugin marketplace at
[`github.com/capire/skills`](https://github.com/capire/skills).

👉 In Claude Code, add the marketplace and install the `cap-developer` plugin:

```text
/plugin marketplace add capire/skills
/plugin install cap-developer@cap
```

This gives Claude Code the CAP skill library, including:

- **`cap-guided-dev`** — the conductor that walks you through building the app stage by stage.
- **`cap-developer`** — expert CAP modeling and handler guidance the conductor delegates to.
- Specialists such as **`cap-check-setup`**, **`cap-inner-loop`**, **`cap-hcql`**, and
  **`cap-troubleshooting`** that other exercises use.

> [!WARNING]
> If a skill isn't found, confirm the marketplace was added and the `cap-developer` plugin
> installed. The public skills repo is released shortly before the event — ask a facilitator if
> the plugin install reports a problem.

> [!IMPORTANT]
> **If the skills marketplace is unreachable, the session still works.** The later exercises are
> framed as "continue the guided flow," but each one also shows the exact **CDS/JS to produce** in
> its "Review what good looks like" section. If `capire/skills` can't be reached, you can still do
> every exercise by asking Claude Code (with the CAP MCP docs server from the next step) to create
> those artifacts directly, or by typing them yourself. You lose the stage-menu conductor UX, not
> the content. Flag it to a facilitator either way.

## Add the CAP documentation MCP server

The CAP skills work best when they can look up current CAP docs instead of guessing. The
**CAP MCP docs server** (`@cap-js/mcp-server`, binary `cds-mcp`) provides exactly that — tools to
search CAP documentation and inspect your project's effective model. On the TechEd notebook it's
**pre-configured**; the steps here are for reference or a non-notebook setup.

👉 Confirm Claude Code can reach it by asking a CAP documentation question, e.g.:

```text
Using the CAP docs, how does the @flow.status annotation work?
```

You should get an answer grounded in the official CAP docs (with source links), not a guess.

> [!NOTE]
> If the server isn't wired up, add it to your AI tool's MCP configuration as a local stdio server
> running `npx -y @cap-js/mcp-server@latest`, then restart the tool. The first run downloads a small
> embedding model, so allow a moment and a network connection.

## Verify your environment with AI

Rather than hand-checking tool versions, ask Claude Code to do it.

👉 In Claude Code, prompt:

```text
Check my CAP development environment.
```

This activates the **`cap-check-setup`** skill, which inspects Node.js, npm, `@sap/cds-dk`, and
other tooling, then reports what's installed, outdated, or missing.

👉 Fix anything it flags before continuing.

## Get the seed project ready

This repository ships a **seed** CAP project at [`xstravels/`](../../xstravels) — a fresh
`cds init` project whose [`readme.md`](../../xstravels/readme.md) already contains the xstravels
**project spec** (the agreed problem statement). This is your starting point: an empty app and a
clear idea. You'll build everything else *with the AI* in the following exercises.

👉 Install its dependencies:

```bash
cd xstravels
npm install
```

👉 Start it once to confirm it boots:

```bash
cds watch
```

You should see CAP report **"No models found … Waiting for some to arrive"** — that's expected.
The app has no domain model yet; you'll create it in Exercise 1.

👉 Stop the server with `Ctrl+C`.

## Read the spec

👉 Open [`xstravels/readme.md`](../../xstravels/readme.md) and skim the project spec: the
purpose, the actors, the core entities, the lifecycle, and the **hard rules** the system must
enforce.

> [!TIP]
> These are the decisions the AI will build on — and the hard rules become the declarative
> constraints you add in Exercise 3. A couple of minutes reading the spec now makes every later
> stage click into place.

## Summary

You now have the latest `@sap/cds-dk`, the CAP skills installed in Claude Code (`cap-developer`
plugin from `capire/skills`), a verified environment (via `cap-check-setup`), and the xstravels
seed project ready to grow — with its spec in [`xstravels/readme.md`](../../xstravels/readme.md).

---

[Next exercise](../ex1/README.md)
