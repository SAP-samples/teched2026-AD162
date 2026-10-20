# Exercise 3 — Declarative constraints

**Goal:** make the service enforce the spec's **hard rules** — required fields, cross-field
checks, referential checks, even a "no overlapping travels" uniqueness rule — **declaratively**,
with no handler code.

The spec listed the rules the system must guarantee on every write. The instinct is to write
JavaScript validation handlers. CAP's answer is better: express most rules as **annotations** on
the model. Declarative constraints are enforced consistently across OData, drafts, and batch
requests, show up in metadata, and can't be bypassed by another handler — none of which a
hand-written check gets for free. This is the `cap-guided-dev` **constraints** stage, and its rule
is explicit: *before writing any logic, ask whether it can be expressed declaratively.*

## Add the constraints

👉 Continue the guided flow in Claude Code:

```text
Continue to the constraints stage. Enforce the hard rules from the spec — declaratively.
```

### What the skill should do

- Move to the **constraints** stage and work through the spec's hard rules one by one.
- For **each** rule, ask "can this be an annotation?" and reach for the declarative form — it
  should *not* write a `before` handler for these. It verifies the exact `@assert` syntax against
  the CAP docs (the `cap` MCP tools) rather than guessing.
- Create [`srv/travel-constraints.cds`](../../xstravels/srv/travel-constraints.cds) mapping rules
  to annotations: `@mandatory` for required references; `@assert: (case when … end)` for field,
  cross-field, referential, booking-period, and currency-match rules; and a **uniqueness** rule
  using an `exists` predicate over a self-association.
- Put user-facing messages in [`_i18n/messages.properties`](../../xstravels/_i18n/messages.properties)
  as plain keys.

### Review what good looks like

👉 Open the constraints file. It should look like this:

```cds
using { TravelService } from './travel-service';
using { sap.capire.travels as my } from '../db/schema';

// Helper to reach all OTHER travel rows from within the overlap constraint.
// Added on the DB entity so the service projection inherits it.
extend my.Travels with {
  others : Association to many my.Travels on 1 = 1;
}

annotate TravelService.Travels with {

  description @assert: (case
    when description is null then 'Description is required'
    when length(trim(description)) < 3 then 'Description is too short'
  end);

  agency @mandatory @assert: (case
    when not exists agency then 'ASSERT_AGENCY_UNKNOWN'
  end);

  customer @mandatory @assert: (case
    when not exists customer then 'ASSERT_CUSTOMER_UNKNOWN'
  end);

  beginDate @mandatory @assert: (case
    when endDate < beginDate then 'ASSERT_BEGINDATE_BEFORE_ENDDATE'
    when exists bookings[flight.date < $self.beginDate] then 'ASSERT_BOOKING_IN_TRAVEL_PERIOD'
    // no second travel of the same customer may overlap this one
    when exists others[
         customer.ID = $self.customer.ID
         and ID != $self.ID
         and beginDate <= $self.endDate
         and endDate   >= $self.beginDate
       ] then 'ASSERT_NO_OVERLAPPING_TRAVELS'
  end);

  endDate @mandatory @assert: (case
    when endDate < beginDate then 'ASSERT_ENDDATE_AFTER_BEGINDATE'
    when exists bookings[flight.date > $self.endDate] then 'ASSERT_BOOKING_IN_TRAVEL_PERIOD'
  end);

  bookingFee @assert: (case
    when bookingFee < 0 then 'ASSERT_BOOKING_FEE_NON_NEGATIVE'
  end);
}

annotate TravelService.Bookings with {
  travel @mandatory;
  flight @mandatory;

  flightPrice @assert: (case
    when flightPrice < 0 then 'ASSERT_FLIGHT_PRICE_NON_NEGATIVE'
  end);

  currency @assert: (case
    when currency.code != travel.currency.code then 'ASSERT_BOOKING_CURRENCY_MATCHES_TRAVEL'
  end);

  bookingDate @assert: (case
    when bookingDate > travel.endDate then 'ASSERT_NO_BOOKINGS_AFTER_TRAVEL'
  end);
}
```

And the messages, in `_i18n/messages.properties`:

```properties
ASSERT_AGENCY_UNKNOWN = The selected travel agency does not exist.
ASSERT_CUSTOMER_UNKNOWN = The selected customer does not exist.
ASSERT_BEGINDATE_BEFORE_ENDDATE = The begin date must not be after the end date.
ASSERT_ENDDATE_AFTER_BEGINDATE = The end date must not be before the begin date.
ASSERT_NO_OVERLAPPING_TRAVELS = This customer already has a travel that overlaps this period.
ASSERT_BOOKING_FEE_NON_NEGATIVE = The booking fee must not be negative.
ASSERT_FLIGHT_PRICE_NON_NEGATIVE = The flight price must not be negative.
ASSERT_BOOKING_IN_TRAVEL_PERIOD = The flight date must fall within the travel period.
ASSERT_BOOKING_CURRENCY_MATCHES_TRAVEL = The booking currency must match the travel currency.
ASSERT_NO_BOOKINGS_AFTER_TRAVEL = A booking date must not be after the travel end date.
```

> [!TIP]
> **`@assert: (case when … end)` is the workhorse.** It's an expression evaluated per row that
> rejects the row when it yields a message. It handles conditional-required, field-vs-field, and
> (via associations) cross-row rules — most things people reach for handlers to do. Messages are
> plain `i18n` keys (no `{i18n>…}` wrapper); they resolve from `_i18n/messages.properties` and
> also surface as the error `code`.

> [!WARNING]
> **Two things that are easy to get subtly wrong** — have the AI verify both against the CAP docs:
> - The uniqueness **`others` helper must `extend` the DB entity** (`my.Travels`), not the service
>   projection — CDS can't add elements to a projection that way, and will error.
> - The **booking-in-period check is expressed from the Travel** via
>   `exists bookings[flight.date …]`, not as an annotation on the Booking's `flight.date` (which
>   isn't a foreign key and would only produce a warning).

## Verify it

👉 Run `cds watch`, then try to create a travel with the end date before the begin date (use real
agency/customer IDs from `GET …/TravelAgencies` and `GET …/Customers`):

```bash
curl -s -u alice: -X POST "http://localhost:4004/odata/v4/travel/Travels" \
  -H "Content-Type: application/json" \
  -d '{"description":"Bad dates","beginDate":"2026-08-10","endDate":"2026-08-01","currency_code":"EUR","agency_ID":"<an agency ID>","customer_ID":"<a customer ID>"}'
```

> [!TIP]
> **Skip the shell and the ID hunting.** Open [`requests.http`](requests.http) and click
> **Send Request** top to bottom — the [REST Client](https://marketplace.visualstudio.com/items?itemName=humao.rest-client)
> extension grabs a real agency and customer ID from the first two requests and fills them into
> the create bodies automatically, so there are no UUIDs to copy and it adapts to this run's data.

It's **rejected** with your i18n messages, wrapped as multiple errors:

```json
{"error":{"message":"Multiple errors occurred, see details below.","code":"MULTIPLE_ERRORS","details":[
  {"message":"The begin date must not be after the end date.","code":"ASSERT_BEGINDATE_BEFORE_ENDDATE","target":"beginDate"},
  {"message":"The end date must not be before the begin date.","code":"ASSERT_ENDDATE_AFTER_BEGINDATE","target":"endDate"}
]}}
```

👉 Try a negative booking fee — rejected with a single, readable message:

```json
{"error":{"message":"The booking fee must not be negative.","code":"ASSERT_BOOKING_FEE_NON_NEGATIVE","target":"bookingFee"}}
```

👉 Submit a **valid** travel and confirm you get **HTTP 201 Created**.

> [!NOTE]
> **Notice what you did *not* write:** there is no JavaScript handler for any of these rules. The
> entire validation layer is model annotations — enforced on every write path.

## Extend it yourself

👉 Pick one more rule from the spec (or invent one) and ask the AI to add it declaratively — for
example, *"a travel's booking fee may not exceed 100"*. Watch it reach for `@assert.range` or a
`case` expression rather than a handler, then prove it with a failing and a passing request.

## Summary

You turned the spec's hard rules into a declarative validation layer — required fields,
cross-field and cross-entity checks, and a uniqueness rule — with zero handler code, verified live.

Key files created:

- [`srv/travel-constraints.cds`](../../xstravels/srv/travel-constraints.cds) — the declarative
  constraints
- [`_i18n/messages.properties`](../../xstravels/_i18n/messages.properties) — the user-facing
  messages

---

[Next exercise](../ex4/README.md)
