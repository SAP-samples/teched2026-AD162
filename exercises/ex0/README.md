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
| `@sap/cds-dk` (global CLI) | 10.2 |
| `@sap/cds` (runtime) | ^10 |
| `@cap-js/sqlite` | ^3 |
| `@cap-js/cds-test` | ^1 |
| CAP skills | public `github.com/capire/skills` (`teched2026-ad162` branch) |
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

👉 When VS Code opens the project, a notification appears asking *"Do you want to install the
recommended extensions from Microsoft, mechatroner and others for this repository?"* — click
**Install**. This repository recommends a handful of extensions (CDS language support, ESLint, a
CSV viewer, and a SQLite viewer) that make the following exercises smoother.

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
> **Use the latest `@sap/cds-dk` (10.2 or later).** The guided-dev flow relies on current CLI features (e.g.
> `cds add data --keys-only`). An outdated kit generates stale project layouts and misses flags,
> which quietly derails later stages.

## Install the CAP documentation MCP server

The CAP skills work best when they can look up current CAP docs instead of guessing. The
**CAP MCP docs server** (`@cap-js/mcp-server`, binary `cds-mcp`) provides exactly that — tools to
search CAP documentation and inspect your project's effective model. It is published at
[`github.com/cap-js/mcp-server`](https://github.com/cap-js/mcp-server). On the TechEd notebook it's
**pre-configured**; otherwise add it yourself.

👉 In the terminal, run:

```text
claude mcp add cds-mcp -- npx -y @cap-js/mcp-server
```

> [!NOTE]
> The first run downloads a small embedding model, so allow a moment and a network connection.

## Install the CAP skills into Claude Code

The CAP skills are published as a plugin marketplace at
[`github.com/capire/skills`](https://github.com/capire/skills). This session uses the
**`teched2026-ad162`** branch — a frozen snapshot of the full skill set (including `cap-guided-dev`)
pinned for this workshop.

👉 In Claude Code, add the marketplace (pinned to the `teched2026-ad162` branch) and install the
`cap-developer` plugin:

```text
/plugin marketplace add capire/skills#teched2026-ad162
/plugin install cap-developer@cap
```

This gives Claude Code the CAP skill library, including:

- **`cap-guided-dev`** — the conductor that walks you through building the app stage by stage.
- **`cap-developer`** — expert CAP modeling and handler guidance the conductor delegates to.
- Specialists such as **`cap-check-setup`**, **`cap-inner-loop`**, **`cap-hcql`**, and
  **`cap-troubleshooting`** that other exercises use.


## Restart Claude Code and verify your setup

Both the MCP server and the skills are picked up when Claude Code starts.

👉 **Restart Claude Code** now that you've installed both, so it loads them.

👉 **Verify the skills** — list the installed plugins in Claude Code:

```text
/plugin
```

You should see the `cap-developer` plugin from the `cap` marketplace, exposing `cap-guided-dev`,
`cap-developer`, `cap-check-setup`, `cap-inner-loop`, `cap-hcql`, `cap-troubleshooting`, and more.

👉 **Verify the CAP MCP docs server** — ask Claude Code a CAP documentation question:

```text
Using the CAP docs, how does the @flow.status annotation work?
```

You should get an answer grounded in the official CAP docs (with source links), not a guess.

👉 **Verify the rest of your environment** — rather than hand-checking tool versions, ask Claude
Code to do it:

```text
Check my CAP development environment.
```

This activates the **`cap-check-setup`** skill, which inspects Node.js, npm, `@sap/cds-dk`, and
other tooling, then reports what's installed, outdated, or missing.

👉 Fix anything it flags before continuing.

> [!TIP]
> **Cut down the permission prompts.** By default Claude Code asks before each edit or command,
> which gets tedious across a whole app build. Press **`Shift+Tab`** in the Claude Code prompt to
> cycle the **permission mode** — one press gives you **accept edits** (`⏵⏵ accept edits on`), and
> another, where available, gives you **auto mode** (`⏵⏵ auto mode on`) so it runs without stopping
> to ask. In the VS Code extension you can also pick the mode from the indicator under the prompt
> box (**Edit automatically** / **Auto**). You'll still see the diffs — keep reviewing what the AI
> changes, which is the whole point of this session.

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

> [!TIP]
> **Port 4004 still in use?** If a previous `cds watch` didn't shut down cleanly, the next start
> fails with `EADDRINUSE … :4004`. Kill whatever holds the port:
>
> ```powershell
> # Windows PowerShell
> Get-NetTCPConnection -LocalPort 4004 | ForEach-Object { Stop-Process -Id $_.OwningProcess -Force }
> ```
>
> ```bash
> # macOS / Linux / Git Bash
> lsof -ti tcp:4004 | xargs kill -9
> ```
>
> Then run `cds watch` again. (Or just ask Claude Code *"kill whatever is running on port 4004"* —
> the `cap-troubleshooting` skill handles it cross-platform.)

## Read the spec

👉 Open [`xstravels/readme.md`](../../xstravels/readme.md) and skim the project spec: the
purpose, the actors, the core entities, the lifecycle, and the **hard rules** the system must
enforce.

> [!TIP]
> These are the decisions the AI will build on — and the hard rules become the declarative
> constraints you add in Exercise 3. A couple of minutes reading the spec now makes every later
> stage click into place.

## Summary

You now have the latest `@sap/cds-dk`, the CAP MCP docs server (`cds-mcp`), the CAP skills installed
in Claude Code (`cap-developer` plugin from `capire/skills#teched2026-ad162`), a verified
environment (via `cap-check-setup`), and the xstravels seed project ready to grow — with its spec in
[`xstravels/readme.md`](../../xstravels/readme.md).

---

[Next exercise](../ex1/README.md)
