# Exercise 4 — Status flows

**Goal:** model the travel **lifecycle** — Open → Accepted / Rejected, with the ability to reopen
— as a declarative **state machine**. CAP enforces the allowed transitions, updates the status
automatically, and even generates the action buttons in a Fiori UI. Again: no handler code.

The spec described a lifecycle: a travel moves through states, and only certain transitions are
allowed (you can't accept a travel that was never open). The procedural approach is a web of
`if (status === …)` checks in handlers. CAP's declarative approach is the **`@flow.status`**
annotation: you declare the status element and, per action, which states it may run **`@from`**
and which state it moves the travel **`@to`**. This is the climax of the `cap-guided-dev`
declarative arc.

## Add the status flow

👉 Continue the guided flow in Claude Code:

```text
Continue. Model the travel lifecycle from the spec as a status flow with accept, reject,
and reopen actions.
```

### What the skill should do

- Recognize this as a status-transition workflow and reach for **`@flow.status`** (verifying the
  annotation shape against the CAP docs via the `cap` MCP tools).
- Confirm the prerequisite from Exercise 1: the status element is an **enum-coded `CodeList`** so
  states can be referenced as `#Open`, `#Accepted`, etc.
- Declare the three **actions** on the `Travels` projection in
  [`srv/travel-service.cds`](../../xstravels/srv/travel-service.cds), and create
  [`srv/travel-flows.cds`](../../xstravels/srv/travel-flows.cds) wiring the state machine.

> [!IMPORTANT]
> If the AI skipped the enum on `TravelStatus` back in Exercise 1, `@flow.status` and the `#Open`
> symbols won't resolve. Fix the code list first — `key code : String(1) enum { Open='O'; … }`.

### Review what good looks like

👉 Check the actions on the `Travels` projection:

```cds
entity Travels as projection on my.Travels actions {
  action acceptTravel();
  action rejectTravel();
  action reopenTravel();
};
```

👉 And the state machine in `srv/travel-flows.cds`:

```cds
using { TravelService } from './travel-service';

annotate TravelService.Travels with @flow.status: status actions {
  acceptTravel  @from: [ #Open ]                 @to: #Accepted;
  rejectTravel  @from: [ #Open ]                 @to: #Rejected;
  reopenTravel  @from: [ #Rejected, #Accepted ]  @to: #Open;
}
```

> [!TIP]
> **What `@flow.status` buys you.** `@flow.status: status` designates the status element. `@from`
> lists the states an action is allowed to run in; `@to` is the state it moves the entity to. At
> runtime CAP **guards** every call (rejecting invalid transitions), **updates the status
> automatically** on success, and — in a Fiori Elements UI — **generates the action buttons** with
> the right enablement. You declare the rules once; the framework does the rest.

## Verify it

👉 Run `cds watch`, then pick an **Open** travel (`status_code` `O`) and accept it:

```bash
curl -s -o /dev/null -w "HTTP %{http_code}\n" -u alice: -X POST \
  "http://localhost:4004/odata/v4/travel/Travels(<open-travel-ID>)/TravelService.acceptTravel" \
  -H "Content-Type: application/json" -d '{}'
```

You get **`HTTP 204`**, and the status flips automatically:

```bash
curl -s -u alice: \
  "http://localhost:4004/odata/v4/travel/Travels(<same-ID>)?\$select=description,status_code"
# → "status_code":"A"
```

👉 Now try an **illegal** transition — accept the same travel again (it's no longer Open):

```bash
curl -s -u alice: -X POST \
  "http://localhost:4004/odata/v4/travel/Travels(<same-ID>)/TravelService.acceptTravel" \
  -H "Content-Type: application/json" -d '{}'
```

CAP rejects it with a precise, framework-generated error:

```json
{"error":{"message":"Action \"acceptTravel\" requires \"status_code\" to be \"[\"Open\"]\".","code":"INVALID_FLOW_TRANSITION_SINGLE"}}
```

👉 Finally, `reopenTravel` on the accepted travel brings it back to Open — and now `acceptTravel`
works again. The state machine is doing exactly what you declared.

> [!NOTE]
> **Again, notice what you did *not* write:** no status checks, no manual status updates, no
> transition guards in JavaScript. The lifecycle is entirely declarative.

## Extend it yourself

The spec's lifecycle also mentions an **InReview** state (`P`).

👉 Ask the AI to add a `submitForReview` action that moves a travel `@from: [#Open] @to:
#InReview`, and adjust `acceptTravel`/`rejectTravel` to run `@from: [#InReview]` instead of
`#Open`. This is a **model-only** change — no handler. Prove the new path with a few requests.

> [!TIP]
> Want a rule `@flow.status` *can't* express — say, *reject an empty travel from being submitted
> for review*? That's the signal to add a small `before` handler. The AI writes the test first,
> then the minimal handler, only where annotations genuinely fall short — the subject of the
> next exercise.

## Summary

You modeled the travel lifecycle as a declarative state machine. CAP guards the transitions,
updates the status, and (in Fiori) generates the buttons — from two small annotation files, no
handler code.

Key files created / changed:

- [`srv/travel-flows.cds`](../../xstravels/srv/travel-flows.cds) — the `@flow.status` state machine
- [`srv/travel-service.cds`](../../xstravels/srv/travel-service.cds) — the three lifecycle actions

You've completed the **declarative core** of xstravels: a modeled domain, a service, enforced
constraints, and a lifecycle — built by directing the AI, one reviewable stage at a time.

---

From here the exercises build out the rest of the AD162 toolkit — custom handlers (test-first),
the HCQL and AI protocols, exposing the app over **MCP** so you can drive it in natural language,
and authorization. Each stands on its own, so if one topic grabs you, you can jump ahead via the
[main exercise list](../../README.md) — but the recommended path is simply to keep going.

[Next exercise](../ex5/README.md)
