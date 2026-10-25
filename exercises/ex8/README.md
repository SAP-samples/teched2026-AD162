# Exercise 8 — Expose the app over MCP and add an agent

**Goal:** turn the app you built into an **MCP server**, talk to it in natural language from
**Claude Code**, then add a **CAP-level agent** with its own browser chat. This is the climax of
the session: the thing you built becomes a tool the AI can use and hosts an agent of its own.

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

## Add a CAP-level agent

So far **Claude Code hosts the agent** and your app supplies MCP tools. With
[`@cap-js/agents`](https://cap.cloud.sap/docs/guides/ai/cap-agents), the **CAP app hosts its own
agent**, served over the Agent-to-Agent (**A2A**) protocol. Let's add a small chat over the same
travel data.

👉 In Claude Code, ask:

```text
Add a CAP-level agent to TravelService using @cap-js/agents and @agent.
Keep OData, HCQL, MCP, and the existing MCP instructions in travel-protocols.cds.
Use the plugin's built-in agent and tools to answer travel questions. Check the CAP docs
for local LLM configuration and use the existing Claude Code setup where supported.
```

### What good looks like

The skill installs the plugin with `npm add @cap-js/agents` and extends the **existing** protocol
annotation in [`srv/travel-protocols.cds`](../../xstravels/srv/travel-protocols.cds):

```cds
using { TravelService } from './travel-service';

annotate TravelService with @odata @hcql @mcp @agent
  @mcp.instructions: 'This service manages travel bookings. Use describe to explore entities like Travels, Bookings, Customers, and Flights. Use query to search travels by status, date range, or agency.';
```

The plugin supplies the agent loop and service tools — no custom agent handler needed. In local
Node.js development it can pick up a supported Claude Code LLM configuration automatically; if
that is unavailable, follow the [LLM configuration guide](https://cap.cloud.sap/docs/guides/ai/cap-agents#configuration).

👉 Review the diff, restart `cds watch`, and open the **Preview** link beside the new A2A endpoint
on <http://localhost:4004> — normally <http://localhost:4004/a2a/travel/preview/>.

👉 In that browser chat, ask:

```text
List the open travels with their agency and total price, then summarize which is most expensive.
```

Check that the answer matches your sample data and watch the server log for the agent's service
tool calls. The request now goes to the **app's agent**, rather than Claude Code's agent.

> [!TIP]
> **Go deeper in sibling hands-on AD-262.** [Exercise 5 — Add Agent with Skills](https://github.com/SAP-samples/teched2026-AD262/tree/main/exercises/ex5)
> builds an invoice-matching agent with domain policies and human-in-the-loop approval. This small
> travel chat is the starting point; AD-262 explores the richer workflow.

## Summary

You exposed xstravels over MCP and drove it from Claude Code, then added a CAP-level agent with
its own browser chat — closing the loop from *building* the app with AI to *operating* it with AI.

Key change:

- `@cap-js/mcp` and `@cap-js/agents` added; `@mcp`, `@mcp.instructions`, and `@agent` on
  [`srv/travel-protocols.cds`](../../xstravels/srv/travel-protocols.cds)

---

[Next exercise](../ex9/README.md)
