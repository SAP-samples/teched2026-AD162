# Exercise 8 — Expose the app over MCP

**Goal:** turn the app you built into an **MCP server**, then **talk to it in natural language from
Claude Code** — "list all open travels with their agency and total price" — and watch the agent
query your own service. This is the climax of the session: the thing you built becomes a tool the
AI can use.

The [Model Context Protocol](https://modelcontextprotocol.io/) lets AI agents discover and call
tools. From a CAP developer's view, `@mcp` is just another protocol — like `@odata` or `@hcql`. Add
it, and the `@cap-js/mcp` adapter exposes your service as MCP tools (`describe`, `query`,
`call_action`) that any MCP client — including the Claude Code you've been using all along — can
drive.

## Expose the service over MCP

👉 Continue the guided flow in Claude Code:

```text
Expose the travel service over MCP with @cap-js/mcp, keeping OData and HCQL. Add instructions so
an agent knows how to use it.
```

### What the skill should do

- Install the plugin:

  ```bash
  npm add @cap-js/mcp
  ```

- Add `@mcp` to the service's protocol annotation **and `@mcp.instructions`** to guide agents.

> [!WARNING]
> **Don't add a second `annotate` block.** A service's protocol annotation can be assigned only
> once — a separate `@mcp` block alongside Exercise 6's `@odata @hcql` block fails with
> `Duplicate assignment with "@hcql"/"@odata"`. **Add `@mcp` to the existing**
> [`srv/travel-protocols.cds`](../../xstravels/srv/travel-protocols.cds):
>
> ```cds
> using { TravelService } from './travel-service';
>
> annotate TravelService with @odata @hcql @mcp
>   @mcp.instructions: 'This service manages travel bookings. Use describe to explore entities like Travels, Bookings, Customers, and Flights. Use query to search travels by status, date range, or agency.';
> ```

## What to look for

👉 Run `cds watch`. The service is now served over all three protocols:

```text
[cds] - serving TravelService {
  at: [ '/odata/v4/travel', '/hcql/travel', '/mcp/travel' ],
  ...
}
```

## Drive your own app in natural language

The CAP MCP adapter **autowires** into local MCP clients during `cds watch` — including Claude
Code. That means the service you just built is now a set of tools your AI assistant can call.

👉 In Claude Code, ask a plain-English question about your data:

```text
List all open travels with their agency and total price.
```

The agent discovers the model and runs a query against your service — you'll see it call the MCP
tools:

```text
⏺ cds:TravelService - describe (MCP)()
⏺ cds:TravelService - query (MCP)(cql: "SELECT from Travels { ID, description, agency.name, totalPrice, status.name } where status.code = 'O'")
```

> [!NOTE]
> **How the `query` tool is actually called.** The `query` tool takes a single **`cql` string**
> argument (a CQL `SELECT`). The agent narrates its intent in natural language, but under the hood
> it's passing CQL — the same language you used over HCQL in Exercise 6. If you drive the tool by
> hand (see the Inspector below), supply a `cql` string, not separate `entity`/`select`/`where`
> fields.

👉 Try a few more — the agent translates each into a CQL query against your service:

```text
Which travel agencies have the most bookings?
What is the average booking fee across all open travels?
Show me travels in review, sorted by total price.
```

👉 Watch your `cds watch` terminal — each natural-language question shows up as a logged tool call
carrying the CQL the agent built:

```js
[mcp] - query { service: 'TravelService',
  cql: "SELECT from Travels { ID, description, totalPrice, status.name } where status.code = 'O'" }
```

> [!NOTE]
> **This is the whole arc closing.** You directed an AI to *build* xstravels; now that same AI
> *consumes* it, through a protocol you enabled with one annotation. No glue code, no custom tool
> definitions — the model you designed is the contract.

> [!TIP]
> **Current limitation.** The MCP adapter supports **read** (`query`) and **unbound actions**
> (`call_action`) — not direct create/update/delete. Expose any write you want an agent to perform
> as an unbound action. (Your `submitForReview`/`acceptTravel` are bound actions on `Travels`;
> add an unbound wrapper if you want to drive them from MCP.)

### Deterministic fallback — the MCP Inspector

Natural-language output varies with the model. If you want to inspect the MCP server with **no LLM
in the loop** (or if the live NL run misbehaves), use the MCP Inspector:

👉 In a new terminal:

```bash
npx @modelcontextprotocol/inspector
```

Then connect with Transport **Streamable HTTP**, URL `http://localhost:4004/mcp/travel`, and an
Authorization header `Basic YWxpY2U6` (that's `alice:` base64-encoded). **List Tools** shows
`describe` and `query` (and `call_action` **once the service has an unbound action** — see the note
below). Run `query` with a `cql` argument such as `SELECT from Travels { ID, description }` to get
rows back in the compact TOON format — fully deterministic.

> [!NOTE]
> **Why you may see only two tools.** `call_action` appears only when the service exposes an
> **unbound** action. xstravels' actions (`submitForReview`, `acceptTravel`, …) are **bound** to
> `Travels`, so as built the Inspector lists just `describe` and `query`. Add an unbound action (per
> the limitation note above) and `call_action` shows up.

## Summary

You exposed xstravels over MCP with a one-line annotation and drove it in natural language from
Claude Code — closing the loop from *building* the app with AI to *operating* it with AI.

Key change:

- `@cap-js/mcp` added; `@mcp` + `@mcp.instructions` on
  [`srv/travel-protocols.cds`](../../xstravels/srv/travel-protocols.cds)

---

[Back to the exercise list](../../README.md) · [Next exercise](../ex9/README.md)
