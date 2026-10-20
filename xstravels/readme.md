# xstravels — a travel booking app

> This is the **project spec** for xstravels. It is the first artifact of the
> `cap-guided-dev` flow (stage 1 — *rough idea*): a short, agreed problem statement that
> every later stage builds on. You start the TechEd AD162 build from here — the domain
> model, services, constraints, and so on are what you'll create *with* the AI, stage by
> stage.

## Purpose

xstravels lets a travel agency manage **travels** (trips) for their customers and the
**flight bookings** that make up each travel. It tracks what was booked, for whom, what it
costs, and where each travel is in its approval lifecycle.

## Primary actors

- **Travel agent** — creates and edits travels and their bookings for customers.
- **Reviewer / processor** — moves a travel through its approval lifecycle (review, accept,
  reject, reopen) and may apply discounts.

## Core entities

- **Travel** — the root thing the app manages: a trip for one customer, booked through one
  agency, with a begin/end date, a booking fee, a total price, a currency, and a status.
- **Booking** — a single flight within a travel (flight, price, currency, booking date);
  a travel is composed of its bookings.
- Supporting reference data: **Customers**, **Travel Agencies**, **Flights**, and the
  **Travel status** code list.

## Naming conventions

Follow standard CAP/CDS casing when modeling these entities:

- **Entities and types** use `PascalCase`, with entities named in the plural — `Travels`,
  `Bookings`, `Customers`, `TravelAgencies`, `Flights`, `TravelStatus`.
- **Elements (fields)** use lowercase `camelCase` — `beginDate`, `endDate`, `bookingFee`,
  `totalPrice`, `flightPrice`, `bookingDate` — **not** `BeginDate` or `TotalPrice`. Element names
  are case-sensitive and are used verbatim in OData, HCQL, JSON payloads, and CSV headers.
- **Namespace** is lowercase and dotted, e.g. `sap.capire.travels`.

## Lifecycle (status flow)

A travel moves through a small state machine: **Open → InReview → Accepted / Rejected**,
with the ability to **reopen** a travel back to Open. Only certain transitions are allowed
(e.g. you can't accept a travel that was never reviewed).

## Hard rules the system must enforce

These are the business rules the app must guarantee on every write (they become the
declarative constraints in a later stage — prefer annotations over handlers):

1. **Description** is required and must be at least 3 characters.
2. **Agency** is mandatory and must reference an existing agency.
3. **Customer** is mandatory and must reference an existing customer.
4. **Begin date and end date** are both required, and the end date must not be before the
   begin date.
5. Every **booking's flight date must fall within its travel's begin/end period.**
6. **Booking fee** and **flight price** must not be negative.
7. A **booking's currency must match its travel's currency.**
8. A **booking date must not be after the travel's end date.**
9. *(reached later)* A customer must not have **two overlapping travels** at the same time.

## Where this goes next

From here, you'll build xstravels with Claude Code and the `cap-guided-dev` skill:

1. **Domain model** — turn the core entities above into CDS + sample data.
2. **Services** — expose the domain for the Fiori UI and API consumers.
3. **Constraints** — encode the hard rules declaratively.
4. **Status flows** — model the lifecycle as a declarative state machine.
5. **Protocols & AI** — HCQL, AI recommendations, and exposing the app over MCP.

## Project layout

File or Folder | Purpose
---------|----------
`app/` | content for UI frontends goes here
`db/` | your domain models and data go here
`srv/` | your service models and code go here
`readme.md` | this project spec

## Learn more

Learn more about CAP at <https://cap.cloud.sap>.
