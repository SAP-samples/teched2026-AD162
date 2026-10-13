# Exercise 7 — AI recommendations in the UI

**Goal:** add the `@cap-js/ai` plugin so draft-enabled Fiori fields get **AI-powered value
recommendations** — out of the box, no custom handler — and understand how it works under the hood.

CAP ships an AI plugin that detects fields with value helps and offers intelligent recommendations
in draft-enabled Fiori Elements UIs, powered by SAP's RPT-1 model (via the Calesi pattern). You
install it, and it wires itself in. This is another `cap-guided-dev` **plugins** stage step.

> [!IMPORTANT]
> **Setup first — this plugin needs a local embedding model (~290 MB).** Recent `@cap-js/ai`
> downloads an embedding model plus native packages (`onnxruntime-node` ~180 MB + the model ~97 MB)
> on first start, and **the server won't boot until they're present**. If the TechEd notebook has
> these pre-installed, startup is instant — your facilitator will confirm. Otherwise, install them
> **once, up front, and ideally before the session** (it's a large download — don't do it live on
> shared wifi):
>
> ```bash
> npm add -D @huggingface/hub @huggingface/tokenizers@0.1.3 onnxruntime-node@1.20.1
> npx @cap-js/ai install-model sentence-transformers/all-MiniLM-L6-v2
> ```
>
> All three dev dependencies are needed — `install-model` fails without `onnxruntime-node`, and the
> server won't boot without the model.

> [!WARNING]
> The exact dependency set is tied to the `@cap-js/ai` version shipped for the event (verified
> against **1.2.0**). If `install-model` reports another missing package, add it as a `-D`
> dependency and retry — the error message names it.

## Add the AI plugin

👉 Continue the guided flow in Claude Code:

```text
Add the @cap-js/ai plugin for Fiori field recommendations. Keep it on the local mock service —
no SAP AI Core binding.
```

### What the skill should do

- Install the plugin:

  ```bash
  npm add @cap-js/ai
  ```

- Explain that, with no SAP AI Core binding, the plugin uses a built-in **MockAICoreService** — so
  it works fully **offline**.
- Note that the plugin auto-detects **draft-enabled** entities with value-helped fields and adds the
  recommendation wiring for you (synthetic `*_Recommendations` companion entities, `@UI.Recommendations`).

## Enable drafts

The recommendation feature only activates on **draft-enabled** entities — and nothing so far enabled
drafts on `Travels`.

👉 Enable drafts on `Travels` so the plugin has something to attach to. Create
[`srv/travel-ui.cds`](../../xstravels/srv/travel-ui.cds):

```cds
using { TravelService } from './travel-service';

annotate TravelService.Travels with @odata.draft.enabled;
```

> [!NOTE]
> Draft handling is a Fiori/SAPUI5 mechanism — enabling it is what lets the AI plugin generate the
> recommendation companion entities. It also makes `Travels` ready for a Fiori Elements UI, should
> you add one.

## What to look for

👉 Run `cds watch`. You should see the plugin load the AI services in the model list and start
cleanly (thanks to the pre-installed model):

```text
[cds] - loaded model from N file(s):
  ...
  node_modules/@cap-js/ai/srv/MockAICoreService.cds
  node_modules/@cap-js/ai/srv/AICoreService.cds
```

👉 Confirm the plugin wired up recommendations — the companion entity now appears in the service
metadata (a headless check, no UI needed):

```bash
curl -s -u alice: "http://localhost:4004/odata/v4/travel/\$metadata" | grep -o "Travels_Recommendations"
# → Travels_Recommendations
```

That `Travels_Recommendations` entity is what a draft-enabled Fiori UI reads to offer suggestions.

> [!NOTE]
> **The mock isn't real AI.** `MockAICoreService` just returns an existing value from your data as
> the "prediction" — deterministic, not context-aware. It exists to prove the **integration** works
> end-to-end, offline. Real recommendations need an SAP AI Core binding (below, optional).

### Optional — see it in a Fiori UI

> [!TIP]
> The payoff is most visual in a draft-enabled Fiori Elements app: create a new travel and the
> value-helped fields (Agency, Customer) show recommended values. The base xstravels build is
> headless, so generating that UI is out of scope here — your facilitator will demo it, or you can
> add a Fiori app yourself if you finish early.

## Under the hood

For a draft read that expands the recommendations, the plugin sends up to ~2000 rows of the
**active** entity as context to RPT-1 (stripping audit and binary fields) and returns scored
predictions that a Fiori UI renders as soft-fill defaults.

👉 Keep a field out of recommendations (e.g. for privacy) declaratively:

```cds
annotate TravelService.Travels with {
  Customer @UI.RecommendationState: 0;   // 0 = excluded
}
```

> [!TIP]
> It accepts dynamic expressions too — e.g. `Agency @UI.RecommendationState: (TotalPrice > 10000 ?
> 0 : 1)` — so you can suppress recommendations conditionally without any handler.

## Optional — connect real SAP AI Core

> [!NOTE]
> Requires an SAP AI Core instance on BTP; **skip if you don't have one** — the mock covers the
> workshop. With a binding (`cds bind AICore -2 <instance>`), run `cds watch --profile hybrid` and
> the plugin returns real RPT-1 predictions learned from your data.

## Summary

You added AI-powered field recommendations with a single `npm add`, enabled drafts so the plugin
activates, verified the recommendation entity appears in metadata (offline, headless), and learned
to exclude fields declaratively — no custom AI code.

Key changes:

- `@cap-js/ai` added as a dependency
- [`srv/travel-ui.cds`](../../xstravels/srv/travel-ui.cds) — `@odata.draft.enabled` on `Travels`
- optional `@UI.RecommendationState` annotations

---

[Back to the exercise list](../../README.md) · [Next exercise](../ex8/README.md)
