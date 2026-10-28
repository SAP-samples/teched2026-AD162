# Exercise 9 — Authorization

**Goal:** lock the service down **declaratively** — who may read, who may run which lifecycle
action — with `@requires` / `@restrict` and role-based mocked users. No authorization code.

So far anyone (as `alice`) could do anything. Real apps need access control: everyone authenticated
may *read* travels, only *reviewers* may accept or reject them, and *admins* may do everything. CAP
expresses this as **annotations**, enforced across every protocol — the `cap-guided-dev`
**authentication / authorization** stage, which (like constraints) is declarative-first.

## Add access control

👉 Continue the guided flow in Claude Code:

```text
Continue to authorization. Everyone authenticated may read travels; reviewers may accept/reject;
admins may do everything. Add local mocked users with those roles too.
```

### What the skill should do

- Protect the service **declaratively** with `@restrict` rules — reaching for a handler only for
  instance-level checks that can't be expressed as annotations (none are needed here).
- Create [`srv/travel-auth.cds`](../../xstravels/srv/travel-auth.cds):

  ```cds
  using { TravelService.Travels } from './travel-service';

  annotate Travels with @(restrict: [
    { grant: 'READ', to: 'authenticated-user' },
    { grant: ['acceptTravel','rejectTravel'], to: 'reviewer' },
    { grant: ['*'], to: 'admin' }
  ]);
  ```

- Define local **mocked users** with roles so you can test each role, in `.cdsrc.json`:

  ```json
  {
    "requires": {
      "auth": {
        "kind": "mocked",
        "users": {
          "alice":  { "roles": ["admin"] },
          "rachel": { "roles": ["reviewer"] },
          "viewer": { "roles": [] }
        }
      }
    }
  }
  ```

> [!TIP]
> **`authenticated-user`** is the built-in pseudo-role matching any logged-in user. For row-level
> rules you'd add `where` clauses (e.g. `where: 'createdBy = $user'`) — still declarative, still no
> handler.

## What to look for

👉 Run `cds watch`, then test each role. An **anonymous** read is rejected:

```bash
curl -s -o /dev/null -w "anon: HTTP %{http_code}\n" \
  "http://localhost:4004/odata/v4/travel/Travels?\$top=1"
# → anon: HTTP 401
```

> [!TIP]
> **Prefer clicks to shell?** In `xstravels`, open
> [`requests/ex9.http`](../../xstravels/requests/ex9.http) and click **Send Request** top to
> bottom — the [REST Client](https://marketplace.visualstudio.com/items?itemName=humao.rest-client)
> extension runs each role (anonymous, `viewer`, `rachel`, `alice`) and reuses a real InReview
> travel ID for the accept calls, so there's no `<an-InReview-travel-ID>` to paste.
> Set `@acceptFrom = O` in the file if you kept Exercise 4's basic flow; the default `P` is for
> its optional InReview extension. The selected state must match `acceptTravel`'s `@from`.

👉 An authenticated user **without** the reviewer role can read but not accept:

```bash
curl -s -o /dev/null -w "viewer read: HTTP %{http_code}\n" -u viewer: \
  "http://localhost:4004/odata/v4/travel/Travels?\$top=1"
# → viewer read: HTTP 200

curl -s -o /dev/null -w "viewer accept: HTTP %{http_code}\n" -u viewer: -X POST \
  "http://localhost:4004/odata/v4/travel/Travels(<an-InReview-travel-ID>)/TravelService.acceptTravel" \
  -H "Content-Type: application/json" -d '{}'
# → viewer accept: HTTP 403
```

👉 `rachel` (reviewer) *can* accept; `alice` (admin) can do everything.

> [!NOTE]
> **Why declarative wins again.** `@restrict` rules are enforced consistently across OData, HCQL,
> and MCP, show up in metadata, and can't be bypassed by another handler. A hand-written auth check
> in one handler would protect only that one path — these protect them all.

> [!IMPORTANT]
> These are **mocked** users for local development only. For production you'd switch the auth
> strategy to XSUAA or IAS (`cds add xsuaa`) — the `@restrict` annotations stay exactly the same;
> only the identity provider changes.

## Summary

You secured the service with role-based access control — read for everyone authenticated, lifecycle
actions for reviewers, everything for admins — entirely in annotations, verified per role.

Key files created / changed:

- [`srv/travel-auth.cds`](../../xstravels/srv/travel-auth.cds) — the `@restrict` rules
- `.cdsrc.json` — local mocked users with roles

---

You've now been through the full AD162 toolkit — from an idea to a modeled, constrained, stateful,
secured CAP app that talks to AI over MCP, built by *directing* the AI the whole way.

[Back to the exercise list](../../README.md)
