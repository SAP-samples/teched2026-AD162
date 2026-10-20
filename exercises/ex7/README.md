# Exercise 7 — Semantic search with vector embeddings

**Goal:** add a **vector embedding** to travels and run a **semantic search** — find travels by
*meaning*, not keywords — straight over the API, with no UI and no external AI service. A search for
*"skiing in snowy mountains"* finds *"Alpine getaway"* even though they share no words.

> [!NOTE]
> This exercise stands on its own — it builds only on the domain model and service from
> Exercises 1–2 — so you can reach it directly if you're short on time.

Classic search matches *characters*: "beach" finds rows containing "beach". **Vector embeddings**
match *meaning*: text is turned into a numeric vector that captures semantics, so two phrases about
the same idea land close together even with no shared words. CAP makes this a first-class modelling
feature — a `Vector` element plus the `cosine_similarity` function — and it runs **locally on
SQLite** for development. This is a `cap-guided-dev` **plugins** stage step.

> [!IMPORTANT]
> **Setup — the local embedding model is a large download (~290 MB).** To compute embeddings
> offline, `@cap-js/ai` pulls native packages (`onnxruntime-node` ~180 MB) and, on first `cds watch`,
> an embedding model (~97 MB). **The server waits for the model on first start.** If the TechEd
> notebook has these pre-installed, startup is instant — your facilitator will confirm. Otherwise
> install the packages **once, up front, ideally before the session** (don't pull ~290 MB live on
> shared wifi):
>
> ```bash
> npm add -D @cap-js/ai @cap-js/sqlite@^3.1 @huggingface/hub@^2.15.0 @huggingface/tokenizers@0.1.3 onnxruntime-node@1.20.1
> ```
>
> The model itself downloads automatically on the first `cds watch` — no separate install step.

> [!WARNING]
> The exact peer set is tied to the `@cap-js/ai` version shipped for the event (verified against
> **`@sap/cds` 10.1** / `@cap-js/sqlite` 3.1). If a start-up error names another missing package, add
> it as a `-D` dependency and retry — the message tells you which one.

## Add an embedding to the model

👉 Continue the guided flow in Claude Code:

```text
Add semantic search over travels. Give Travels a vector embedding computed from the description,
and add a searchTravels(q) function that returns the best matches by cosine similarity. Keep it
local — use @cap-js/ai on SQLite, no SAP AI Core.
```

### What the skill should do

- Recognize this as a **vector embeddings** task and reach for the built-in `Vector` type and the
  `vector_embedding` / `cosine_similarity` functions — verifying their shape against the CAP docs
  (the `cap` MCP tools) rather than guessing.
- Install `@cap-js/ai` and peers so the standard `sqlite` database computes embeddings locally.
- Add a **stored calculated element** `embedding` on `Travels` in
  [`db/schema.cds`](../../xstravels/db/schema.cds), derived from `description`.
- Add a **`searchTravels(q)`** function to the service and a small handler that ranks travels by
  `cosine_similarity`.

### Review what good looks like

👉 The embedding on `Travels` — a `stored` calculated element, so it's persisted and recomputed on
every write:

```cds
entity Travels : cuid, managed {
  description : String(1024);
  // ... the rest of the elements ...
  embedding   : Vector = vector_embedding(description, 'DOCUMENT', 'SAP_GXY.20250407') stored;
}
```

👉 The search function in [`srv/travel-service.cds`](../../xstravels/srv/travel-service.cds):

```cds
extend service TravelService with {
  function searchTravels(q: String) returns many {
    ID          : UUID;
    description : String;
    similarity  : Double;
  };
}
```

👉 And its handler in [`srv/travel-service.js`](../../xstravels/srv/travel-service.js):

```js
this.on('searchTravels', async req => {
  const { q } = req.data
  return SELECT.from('sap.capire.travels.Travels')
    .columns('ID', 'description',
      `cosine_similarity(embedding, vector_embedding('${q}', 'QUERY', 'SAP_GXY.20250407')) as similarity`)
    .orderBy('similarity desc')
    .limit(5)
})
```

> [!TIP]
> **One query, two embeddings.** `vector_embedding(description, 'DOCUMENT', …)` embeds the stored
> text; `vector_embedding(q, 'QUERY', …)` embeds the user's search phrase. `cosine_similarity`
> scores how close the two vectors point — 1.0 is a perfect semantic match, 0 is unrelated.

> [!IMPORTANT]
> Use the **same embedding model** for both the stored `DOCUMENT` embedding and the `QUERY`
> embedding — similarity is only meaningful within one model's vector space. On SQLite the
> model-name argument is ignored (the local model is used); the *same* query runs unchanged on SAP
> HANA, where the name selects a HANA NLP model.

## Verify it

👉 Run the app (first start downloads the model; later starts are instant):

```bash
cds watch
```

You'll see the plugin initialize the local model, then the usual deploy:

```text
[@cap-js/ai] - Vector embedding ONNX model initialized
/> successfully deployed to in-memory database.
[cds] - serving TravelService { at: [ '/odata/v4/travel' ] }
```

👉 Now search by **meaning**. Ask for mountains and skiing — note the top hit shares **no words**
with the query:

```bash
curl -s -u alice: "http://localhost:4004/odata/v4/travel/searchTravels(q='skiing in snowy mountains')"
```

> [!TIP]
> **On Windows / prefer not to use a shell?** Open [`requests.http`](requests.http) and click
> **Send Request** above either search — the [REST Client](https://marketplace.visualstudio.com/items?itemName=humao.rest-client)
> extension runs it with no `curl` and no shell quirks.

```json
{"value":[
  {"ID":"...","description":"Alpine getaway","similarity":0.398},
  {"ID":"...","description":"Summer trip to New York","similarity":0.292},
  {"ID":"...","description":"Barcelona city break","similarity":0.075},
  {"ID":"...","description":"London business travel","similarity":0.041},
  {"ID":"...","description":"Transatlantic conference","similarity":0.027}
]}
```

**"Alpine getaway" wins** — there's no "ski", "snow", or "mountain" in it, yet the embedding knows
it's the closest in meaning. A keyword search would have returned nothing.

👉 Try a work-trip query — the conference and the business travel rise to the top:

```bash
curl -s -u alice: "http://localhost:4004/odata/v4/travel/searchTravels(q='corporate work conference meeting')"
```

```json
{"value":[
  {"ID":"...","description":"Transatlantic conference","similarity":0.517},
  {"ID":"...","description":"London business travel","similarity":0.159},
  {"ID":"...","description":"Alpine getaway","similarity":0.128},
  {"ID":"...","description":"Barcelona city break","similarity":0.126},
  {"ID":"...","description":"Summer trip to New York","similarity":0.095}
]}
```

> [!NOTE]
> **The scores are modest and that's expected.** The local `all-MiniLM-L6-v2` model embedding very
> short descriptions gives smaller absolute similarities than a production HANA model would — what
> matters is the **ranking**, and the right travels come out on top. With richer text (adding the
> agency, destinations, or booking notes to the embedded string) the separation sharpens.

> [!TIP]
> **Let the AI drive the searches.** Ask Claude Code *"find travels about relaxing by the sea"* or
> *"which travels look like business trips?"* — it calls `searchTravels` for you and reads back the
> ranked matches, so you can explore semantic search conversationally.

## Under the hood

- **`embedding : Vector = … stored`** is a [calculated element](https://cap.cloud.sap/docs/cds/cdl#calculated-elements):
  CAP recomputes and persists it on every `CREATE`/`UPDATE`, so embeddings never drift from the
  source text. `Vector` elements are excluded from default reads — you won't see a wall of numbers
  in `GET /Travels`.
- **`@cap-js/ai` emulates `vector_embedding` on SQLite** with a local ONNX model, so the *same* model
  and the *same* queries you write for SAP HANA run offline in your inner loop — no AI Core, no
  network. In production on HANA, the model-name argument selects a HANA NLP embedding model and the
  vectors are stored in HANA's native vector engine.

> [!NOTE]
> **What you did *not* write:** no embedding pipeline, no vector math, no similarity algorithm, no
> external AI calls. One `Vector` calculated element and one `cosine_similarity` query — the database
> does the rest.

## Summary

You added a vector embedding to `Travels` and a `searchTravels` function, then found travels by
meaning — ranking "Alpine getaway" first for a skiing query with no shared words — entirely offline
on SQLite.

Key changes:

- `@cap-js/ai` added as a dev dependency (local embeddings on SQLite)
- [`db/schema.cds`](../../xstravels/db/schema.cds) — the `embedding : Vector = vector_embedding(…) stored` element
- [`srv/travel-service.cds`](../../xstravels/srv/travel-service.cds) / [`srv/travel-service.js`](../../xstravels/srv/travel-service.js) — the `searchTravels` function and its `cosine_similarity` handler

---

[Next exercise](../ex8/README.md)
