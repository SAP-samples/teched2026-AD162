# Exercise 1 — From idea to a domain model

**Goal:** turn the xstravels project brief into a CDS **domain model** with realistic **sample
data** — by directing Claude Code, not by hand-writing CDS.

In Exercise 0 you installed the CAP skills and got the seed ready. The seed's
[`readme.md`](../../xstravels/readme.md) already holds the agreed **project brief** — purpose,
actors, core entities, lifecycle, and hard rules. That brief is the first artifact of the
`cap-guided-dev` flow (its "rough idea" stage). Here you pick up at the next stage: **the domain
model**.

## Start the guided flow

👉 Open Claude Code in the `xstravels` folder and start the flow:

```text
Let's build this CAP app with the cap-guided-dev skill. The project brief is in readme.md.
Start with the domain model.
```

### What the skill should do

- Activate **`cap-guided-dev`** and establish its documentation source — it should use the **CAP
  MCP server** (the `cap` tools) to ground every annotation and type in current CAP docs rather
  than guessing.
- Show you a short **stage menu** and, because the brief is already written, propose jumping
  straight to the **domain model** stage. 👉 Confirm.
- Delegate modeling to **`cap-developer`** and create [`db/schema.cds`](../../xstravels/db/schema.cds)
  from the brief's core entities — `Travels`, `Bookings`, `Customers`, `TravelAgencies`,
  `Flights`, and a `TravelStatus` code list — following CAP conventions:
  - reuse the `cuid` and `managed` aspects from `@sap/cds/common`,
  - model **Travel → Bookings** as a `Composition of many`,
  - model references (Agency, Customer, Flight, Status) as `Association to`,
  - put a `namespace` at the top of the file.
- Then **generate sample data** (next section).
- Finish with a summary listing the files it created as clickable `file:line` references.

### Review what good looks like

👉 When the skill reports the files it changed, open
[`db/schema.cds`](../../xstravels/db/schema.cds) and read it. It should look essentially like
this:

```cds
namespace sap.capire.travels;

using { cuid, managed, Currency, sap.common.CodeList } from '@sap/cds/common';

entity Travels : cuid, managed {
  description : String(1024);
  beginDate   : Date;
  endDate     : Date;
  bookingFee  : Decimal(9,2);
  totalPrice  : Decimal(9,2) @readonly;
  currency    : Currency;
  status      : Association to TravelStatus default 'O';
  agency      : Association to TravelAgencies;
  customer    : Association to Customers;
  bookings    : Composition of many Bookings on bookings.travel = $self;
}

entity Bookings : cuid {
  travel      : Association to Travels;
  pos         : Integer @readonly;
  flight      : Association to Flights;
  flightPrice : Decimal(9,2);
  currency    : Currency;
  bookingDate : Date;
}

entity TravelAgencies : cuid, managed { name: String(80); street: String(60); city: String(40); country: String(3); }
entity Customers      : cuid, managed { firstName: String(40); lastName: String(40); email: String(255); }
entity Flights        : cuid { airline: String(3); flightNo: String(4); date: Date; origin: String(3); destination: String(3); price: Decimal(9,2); currency: Currency; }

entity TravelStatus : CodeList {
  key code : String(1) enum { Open='O'; InReview='P'; Blocked='B'; Accepted='A'; Rejected='X'; }
}
```

> [!TIP]
> This is the moment to catch a wrong modeling decision. Does `Travels` really *compose* its
> `bookings` (parent-child)? Are references associations, not compositions? Everything later
> builds on this — a wrong choice here makes the services, constraints, and flows wrong too.

> [!IMPORTANT]
> **`TravelStatus` must be an enum-coded `CodeList`.** The `code` key carries an `enum` with named
> symbols (`Open`, `InReview`, …). This is a **prerequisite** for the status-transition flow in
> Exercise 4, where actions refer to states as `#Open`, `#Accepted`, and so on. If the AI models a
> plain code list without the enum, ask it to check the CAP docs for `@flow.status` and fix it
> now.

## Generate and fill the sample data

The guided flow generates CSV skeletons with the CLI, then the AI fills them with realistic
content. The command is:

```bash
cds add data --records 5 --keys-only
```

This writes one CSV per entity under `db/data/`, pre-filling the **primary-key** columns with
generated UUIDs and leaving every other column — including association **foreign keys** — empty for
the AI to complete.

> [!IMPORTANT]
> **`--keys-only` only works together with `--records`.** Run `cds add data --keys-only` on its own
> and the CLI prints `ignoring --keys-only, use --records to create entries` and writes
> **header-only** files (no rows at all). Always pass `--records <n>` as well.

👉 After the AI has generated **and filled** `db/data/`, open a couple of the CSVs and sanity-check
them.

> [!NOTE]
> **What the CLI fills vs. what the AI fills.** The CLI generates the primary keys (the `ID`
> columns). It does **not** populate association foreign keys (e.g. `agency_ID`, `customer_ID`,
> `travel_ID`) or any content — those are left empty. The AI fills the descriptive columns *and*
> wires the foreign keys so travels point at real agencies/customers and bookings point at their
> travel. Keep the generated primary-key UUIDs stable, and make every foreign key reference an `ID`
> that actually exists in the related file.

> [!WARNING]
> **Code-list codes are domain values.** In `sap.capire.travels-TravelStatus.csv` the codes must be
> the real lifecycle codes (`O`, `P`, `B`, `A`, `X`) — not placeholder values. Make sure the AI set
> them (and the matching `.texts` rows) correctly.

## Verify it

👉 Start the app and confirm the model and data load cleanly:

```bash
cds watch
```

In the log you should see every CSV deploy to the in-memory database with **no foreign-key
errors**, ending with:

```text
/> successfully deployed to in-memory database.
[cds] - server listening on { url: 'http://localhost:4004' }
```

You'll also see **"No service definitions found … Waiting for some to arrive"** — expected. You
have a model and data, but no service yet. That's Exercise 2.

👉 Stop the server with `Ctrl+C`.

## Summary

You turned the brief into a CDS domain model and realistic sample data — without typing CDS by
hand. You directed the AI, reviewed each decision, and verified the model deploys.

Key files created:

- [`db/schema.cds`](../../xstravels/db/schema.cds) — the domain model
- `db/data/*.csv` — sample data (keys/FKs from the CLI, content filled by AI)

---

[Next exercise](../ex2/README.md)
