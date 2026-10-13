# Exercise 2 — Services and the running app

**Goal:** expose the domain model through a **service** shaped for consumers, run the app, and
query it live — again by directing Claude Code.

A domain model on its own isn't reachable. In CAP you expose it through a **service** that defines
projections over your entities. That service becomes an OData API (and, with Fiori annotations,
drives a UI). This is the `cap-guided-dev` **services** stage.

## Add the service

👉 Continue the guided flow in Claude Code:

```text
Continue to the services stage. Expose the travels domain for a UI and API consumers.
```

### What the skill should do

- Move to the **services** stage and delegate to **`cap-developer`**.
- Create [`srv/travel-service.cds`](../../xstravels/srv/travel-service.cds) defining a
  `TravelService` that projects the domain entities, following CAP conventions:
  - **never expose db entities directly** — always via projections in the service;
  - expose `Travels` and its `Bookings` as writable projections;
  - expose the reference data (`TravelAgencies`, `Customers`, `Flights`, `TravelStatus`) as
    **`@readonly`** projections.
- Summarize the change and point you at the new file.

### Review what good looks like

👉 Open the service and read it. It should look like this:

```cds
using { sap.capire.travels as my } from '../db/schema';

service TravelService {

  entity Travels  as projection on my.Travels;
  entity Bookings as projection on my.Bookings;

  @readonly entity TravelAgencies as projection on my.TravelAgencies;
  @readonly entity Customers      as projection on my.Customers;
  @readonly entity Flights        as projection on my.Flights;
  @readonly entity TravelStatus   as projection on my.TravelStatus;
}
```

> [!TIP]
> **Why projections, not direct exposure.** Projections let you shape what each consumer sees —
> trim fields, flatten associations, mark reference data read-only — without touching the domain
> model. The db layer stays clean; the service layer is tailored to its clients.

## Verify it

👉 Run the app:

```bash
cds watch
```

This time the log shows the service being served:

```text
[cds] - serving TravelService {
  at: [ '/odata/v4/travel' ],
  decl: 'srv/travel-service.cds:3'
}
[cds] - server listening on { url: 'http://localhost:4004' }
```

👉 Query it live. In a second terminal (or your REST client), fetch a few travels:

```bash
curl -s -u alice: "http://localhost:4004/odata/v4/travel/Travels?\$select=description,totalPrice,status_code&\$top=3"
```

You should get your sample rows back as JSON — real descriptions, totals, and status codes:

```json
{"@odata.context":"$metadata#Travels","value":[
  {"description":"Summer trip to New York","totalPrice":"1610.00","status_code":"A","ID":"..."},
  {"description":"London business travel","totalPrice":"650.00","status_code":"O","ID":"..."},
  {"description":"Transatlantic conference","totalPrice":"920.00","status_code":"P","ID":"..."}
]}
```

> [!NOTE]
> **Mocked auth.** Local `cds watch` uses a mocked auth strategy. `-u alice:` logs in as the
> pre-defined user `alice` with an empty password — enough to exercise the API locally.

👉 Open <http://localhost:4004> in a browser to see the service's index page and metadata links.

### Let the AI explore the running app

You don't have to hand-craft every request.

👉 Ask Claude Code:

```text
Which services are running, and show me a few travels with their customer and agency.
```

> [!TIP]
> This uses the **`cap-inner-loop`** skill to discover the running service and query it for you —
> handy whenever you want to poke at live data without writing URLs.

## Summary

You exposed the domain through `TravelService`, ran the app, and queried real data over OData —
all by directing the AI and reviewing the result.

Key file created:

- [`srv/travel-service.cds`](../../xstravels/srv/travel-service.cds) — the service with
  projections over the domain

---

[Next exercise](../ex3/README.md)
