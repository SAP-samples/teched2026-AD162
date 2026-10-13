# Exercise 5 — Custom handlers, test-first

**Goal:** add the one piece of logic that **can't** be declarative — a rule needing a database
lookup — and build it the right way: **write the test first**, watch it fail, then write the
minimal handler to make it pass.

> [!NOTE]
> This is the first of the **pick-your-adventure** exercises. From here on, each exercise is
> independent — do them in any order. This one builds on the `submitForReview` action from
> Exercise 4's "extend it yourself" step; if you skipped it, the prompt below adds it too.

In Exercises 3 and 4 you expressed every rule declaratively — and that's the goal. But some rules
genuinely can't be an annotation: they need a DB lookup, an external call, or async work. *"A
travel can only be submitted for review if it has at least one booking"* is one of them — it has to
count related rows. That's a legitimate handler. The `cap-guided-dev` **custom handlers** stage
builds these **test-first**, because handler logic is where subtle, app-specific bugs hide.

## Build it test-first

👉 Continue the guided flow in Claude Code:

```text
Continue to custom handlers. A travel may only be submitted for review if it has at least one
booking — that needs a DB lookup, so it can't be a declarative constraint. Build it test-first:
write the test, watch it fail, then the handler.
```

### What the skill should do

- Confirm this rule **can't** be declarative (it counts rows on another entity) — naming the
  reason, per the skill's "handler is the last resort" rule.
- Make sure the `submitForReview` action exists (`@from: [#Open] @to: #InReview`); add it if ex4's
  bonus wasn't done.
- **Write the test first** with `cds.test`, capturing the contract: creating an empty travel and
  submitting it must be **rejected**.
- **Run it and watch it fail** — a test that passes before the handler exists is testing nothing.
- Implement the **minimal `before` handler**, then run the test to **green**.
- Show you both the test and the handler as `file:line` references.

### The test (write this first)

The seed is an ES module project, so the test is ESM too:

```js
import cds from '@sap/cds'
const { POST, expect } = cds.test(import.meta.dirname + '/..')

describe('submitForReview', () => {
  it('rejects submitting an empty travel (no bookings)', async () => {
    const { data: travel } = await POST('/odata/v4/travel/Travels', {
      description: 'Empty travel',
      beginDate: '2026-09-01', endDate: '2026-09-10',
      currency_code: 'EUR',
      agency_ID: '<an agency ID>', customer_ID: '<a customer ID>',
    }, { auth: { username: 'alice' } })

    await expect(
      POST(`/odata/v4/travel/Travels(${travel.ID})/TravelService.submitForReview`, {},
        { auth: { username: 'alice' } })
    ).to.be.rejectedWith(/at least one booking/i)
  })
})
```

> [!IMPORTANT]
> Tests must be **ES modules** (`import …`, `import.meta.dirname`) because the project is
> `"type": "module"`. Run them with **`npm test`** (which runs `cds test`) — not a bare `npx jest`,
> which trips over ESM.

👉 Run the test and watch it fail — **for the right reason**:

```bash
npm test
```

```text
✗ rejects submitting an empty travel (no bookings)
  expected promise to be rejected … but it was fulfilled with Response { status: 204, … }
```

The empty travel submits with `204` because there's no rule yet. The red test proves it exercises
the real behaviour and pins the contract.

### The handler (write this to go green)

```js
import cds from '@sap/cds'

export default class TravelService extends cds.ApplicationService {
  init() {
    // Can't be a declarative @assert — it needs a DB lookup (count related rows).
    this.before('submitForReview', 'Travels', async req => {
      const { ID } = req.params[0]
      const bookings = await SELECT.one`count(*) as n`
        .from('sap.capire.travels.Bookings').where({ travel_ID: ID })
      if (!bookings?.n) {
        req.reject(400, 'A travel must have at least one booking before it can be submitted for review.')
      }
    })
    return super.init()
  }
}
```

👉 Run the test again — now **green**:

```bash
npm test
```

```text
✔ rejects submitting an empty travel (no bookings)
 1 passed
```

> [!TIP]
> **Mix declarative and code — don't over-reach.** Keep everything that *can* be an annotation
> declarative; use the handler only for the part that genuinely can't. Rely on CAP's intrinsic
> transactions, and don't re-implement what the generic provider already does. A good handler is
> small.

## What to look for

👉 With `cds watch` running, confirm the behaviour live: create a travel with no bookings and call
`submitForReview` — you get a **400** with your message. Add a booking, submit again — it succeeds
and the status moves to `InReview`.

> [!NOTE]
> **A booking added via a direct `POST /Bookings` isn't re-checked against the parent's period
> rule.** The booking-in-travel-period constraint from Exercise 3 lives on `Travels` (`exists
> bookings[…]`), so it runs when a *travel* is written — not when you insert a booking straight into
> the `Bookings` set. If you add bookings this way for the live check, a flight date outside the
> travel period won't be rejected. That's inherent to where the check lives; writing through the
> travel (deep update) re-evaluates it.

> [!NOTE]
> **Why test-first here specifically:** handlers are the least-generic, highest-risk code in the
> app. A red-then-green test is the cheapest way to prove the handler does exactly what you asked —
> and to keep it working as later changes land.

## Summary

You added the one rule that couldn't be declarative, and built it the disciplined way: a failing
test first, then the smallest handler to pass it.

Key files created / changed:

- `test/submitForReview.test.js` — the test (written first)
- `srv/travel-service.js` — the minimal `before` handler
- `srv/travel-flows.cds` / `srv/travel-service.cds` — the `submitForReview` action (if added here)

---

[Back to the exercise list](../../README.md) · [Next exercise](../ex6/README.md)
