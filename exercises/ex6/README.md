# Exercise 6 — HCQL: CQL over HTTP

**Goal:** expose your service over **HCQL** — CAP's native query protocol — and run powerful
queries (expands, filters, ordering) straight against it, with the AI driving the queries.

OData is great for Fiori. But when a client is *another CAP service* or a data-federation
scenario, the most natural, efficient query language is CAP's own **CQL**. **HCQL** is CQL over
HTTP: you POST a CQL statement and get `{"data":[…]}` back. This is the `cap-guided-dev` **plugins**
stage, and querying is handled by the **`cap-hcql`** and **`cap-cql`** skills.

## Enable HCQL

👉 Continue the guided flow in Claude Code:

```text
Continue to plugins. Expose the travel service over HCQL as well, keeping OData for the Fiori UI.
Then use the cap-hcql skill to run a few queries.
```

### What the skill should do

- Add HCQL to the service with a protocol annotation, **keeping `@odata`** so the OData/Fiori
  endpoint still works. HCQL ships with `@sap/cds` 10 — no plugin to install, just the annotation.
- Create [`srv/travel-protocols.cds`](../../xstravels/srv/travel-protocols.cds):

  ```cds
  using { TravelService } from './travel-service';

  annotate TravelService with @odata @hcql;
  ```

- Use the **`cap-hcql`** skill (which reads **`cap-cql`**) to send real queries and show results.

> [!NOTE]
> We name this file `travel-protocols.cds` and keep **all** protocol annotations for the service in
> this one `annotate` block. In Exercise 8 you'll add `@mcp` to the *same* block — a service's
> protocol annotation can only be assigned once (see the warning there).

## What to look for

👉 Run `cds watch`. The service is now served over both protocols:

```text
[cds] - serving TravelService {
  at: [ '/odata/v4/travel', '/hcql/travel' ],
  ...
}
```

👉 Send a CQL query as a plain-text POST body:

```bash
curl -s -u alice: "http://localhost:4004/hcql/travel" \
  -H "Content-Type: text/plain" \
  --data 'SELECT from Travels { ID, description, totalPrice, currency.code as currency } limit 3'
```

```json
{"data":[
  {"ID":"...","description":"Summer trip to New York","totalPrice":"1610.00","currency":"EUR"},
  {"ID":"...","description":"London business travel","totalPrice":"650.00","currency":"GBP"},
  {"ID":"...","description":"Transatlantic conference","totalPrice":"920.00","currency":"USD"}
]}
```

👉 Now the thing OData makes verbose — a **deep read with an expand** — in one natural query:

```bash
curl -s -u alice: "http://localhost:4004/hcql/travel" \
  -H "Content-Type: text/plain" \
  --data 'SELECT from Travels { ID, description, bookings { pos, flightPrice, currency.code as currency } } limit 2'
```

```json
{"data":[
  {"ID":"...","description":"Summer trip to New York","bookings":[
    {"pos":1,"flightPrice":"780.00","currency":"EUR"},
    {"pos":2,"flightPrice":"810.00","currency":"EUR"}]},
  {"ID":"...","description":"London business travel","bookings":[
    {"pos":1,"flightPrice":"650.00","currency":"GBP"}]}
]}
```

👉 And a filtered, ordered query:

```bash
curl -s -u alice: "http://localhost:4004/hcql/travel" \
  -H "Content-Type: text/plain" \
  --data "SELECT from Travels { ID, description, beginDate } where status.code = 'O' order by beginDate desc limit 5"
```

> [!TIP]
> **Let the AI write the CQL.** Ask Claude Code *"over HCQL, show me open travels with their agency
> name and the total of their bookings"* — the `cap-hcql`/`cap-cql` skills know CQL's path
> expressions, expands, and infix filters, and will build and run the query for you.

> [!NOTE]
> **CQL text is Node.js-only and experimental.** For a cross-runtime client (including CAP Java),
> send the same query as **CQN JSON** to the same endpoint with `Content-Type: application/json`.
> `cap-hcql` can show you both forms.

## Why it matters

> [!TIP]
> HCQL is the preferred protocol for **CAP-to-CAP integration** and **data federation**. When a CAP
> client connects to a service that exposes `@hcql`, CAP automatically prefers it over OData — so
> `const srv = await cds.connect.to('TravelService')` then `srv.read\`Travels { … }\`` flows across
> process boundaries natively, no hand-written REST glue.

## Summary

You exposed the service over HCQL alongside OData and ran expand, filter, and order queries — the
AI writing the CQL for you.

Key file created:

- [`srv/travel-protocols.cds`](../../xstravels/srv/travel-protocols.cds) — the `@odata @hcql`
  protocol annotation

---

[Next exercise](../ex7/README.md)
