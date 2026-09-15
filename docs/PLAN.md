---
PLAN: "chore: freeze as historical reference; port to Go"
TAG: none
EXECUTOR: unassigned
REVIEWER: none
---

> Part of the browser-native semantic search effort. Master index:
> https://github.com/webtyp/agent/blob/main/docs/PLAN.md

# Plan — freeze this fork, port its ideas to Go

## Status: frozen

This repository is a fork of
[`nitaiaharoni1/vector-storage`](https://github.com/nitaiaharoni1/vector-storage) (MIT).
It is kept as the **reference implementation** for the Go port and receives **no further
TypeScript development**: no features, no dependency bumps, no refactors. Bug fixes only
if a behaviour needs clarifying while porting.

Its value is that it is small (217 lines in `src/VectorStorage.ts`) and complete: it
proves the whole loop — embed, persist to IndexedDB, cosine search, LRU evict — in a form
you can read in one sitting. That is worth preserving exactly as it is.

## What was right, and is kept

- **The document shape.** `text`, `metadata`, `timestamp`, `vector`, `vectorMag`, `hits`
  covers what a semantic store actually needs, with nothing spare.
- **IndexedDB as persistence, RAM as the index.** `this.documents` is the search index;
  IndexedDB is where it survives a reload. Reloading the store into memory at startup and
  searching in memory is the correct call, and the Go port keeps it.
- **A pluggable embedder.** `IVSOptions.embedTextsFn` lets the caller supply embeddings
  instead of hardcoding OpenAI. The Go port promotes this from an option to the central
  port, `embed.Embedder`.
- **LRU by hits then age.** `removeDocsLRU` sorts by `hits` ascending, then `timestamp`
  ascending. Sound policy; the Go port keeps the ordering and changes only how the size
  is measured.
- **Precomputing the magnitude.** `vectorMag` is stored per document to avoid
  recomputing it per query. The Go port goes one step further — see below.

## What is deliberately not ported

Each of these is a correction, not a stylistic preference:

| In the TypeScript | Why it is not ported |
|---|---|
| `saveToIndexDbStorage()` does `clear()` then `put` every document | Rewrites the entire corpus on **every single write**, including after a read (`similaritySearch` calls it to persist hit counters). O(N) I/O per operation. The Go port writes only the shard that changed. |
| `getObjectSizeInMB()` = `JSON.stringify(obj).length` | Serialises the whole corpus just to measure it, and measures the JSON size rather than the stored size. The Go port computes the size arithmetically (`N × Dim × 4` plus row overhead) and cross-checks against `navigator.storage.estimate()`. |
| `calculateSimilarityScores` returns `Array<[doc, number]>` then `.sort()` | Allocates a pair array and sorts all N results to take k. The Go port scores into a fixed-size top-k heap: zero allocations per query. |
| `getCosineSimilarityScore` divides by both magnitudes per document | The Go port stores vectors L2-normalised, so cosine **is** the dot product — one `sqrt` and one division saved per document per query. |
| `constants.DEFAULT_MAX_SIZE_IN_MB: 2048` | Contradicts its own documentation (`IVSOptions` says "Defaults to 4.8. Cannot exceed 5" — a localStorage-era limit left behind after the move to IndexedDB). The Go port derives the budget from the actual quota. |
| `debounce()` in `helpers.ts` | Exported, imported nowhere, and `debounceTime` is read into a field that is never used. Dead code. |
| `documents.filter(d => d.text === doc.text)` for dedup | O(N) string comparison per insert, O(N·M) for a batch. The Go port dedups on a content hash in an index. |
| The constructor calls `loadFromIndexDbStorage()` without awaiting | Every public method races the load. A `VectorStorage` used immediately after construction searches an empty corpus and silently returns nothing. The Go port makes the load explicit and returns an error. |
| `console.error` when no API key is given, then continuing | Constructs an object that cannot work. The Go port returns an error from its constructor. |
| OpenAI HTTP embedding as the default | Requires a network round-trip per query and puts an API key in client-side JavaScript, where it is public. The Go port generates embeddings **in the browser**; see master index **D4**. |

## Port map

| TypeScript | Go destination | Notes |
|---|---|---|
| `VectorStorage<T>` class | `webtyp.com/vectordb`.`Store` | generics replaced by an opaque `model.RawJSON` metadata payload |
| `addText` / `addTexts` / `addDocuments` | `Store.Add` | batch-first; the single-document form wraps it |
| `similaritySearch` | `Store.Search` | returns `[]Match`; the query embedding is not echoed back |
| `IVSDocument<T>` | `vectordb.Doc` | `vectorMag` disappears — vectors are stored normalised |
| `IVSSimilaritySearchParams` | `vectordb.Query` | |
| `IVSFilterOptions` / `filterDocuments` | `vectordb.Query.IncludeTags`/`ExcludeTags` | metadata filtering, reshaped — see master index **O3** |
| `calcVectorMagnitude` / `getCosineSimilarityScore` / `normalizeScore` | `webtyp.com/vector`.`Norm`, `Dot`, `Normalize` | |
| `calculateSimilarityScores` + `.sort().slice(0, k)` | `vector.TopK` | |
| `removeDocsLRU` | `Store.evict` | same ordering, different size measurement |
| `initDB` / `loadFromIndexDbStorage` / `saveToIndexDbStorage` | `webtyp.com/indexdb` via `storage.Conn` | the driver owns IndexedDB; the store owns policy |
| `embedTexts` (OpenAI HTTP) | `webtyp.com/embed`.`Embedder` | reshaped from an option into the port |
| `ICreateEmbeddingResponse` | not ported | no HTTP embedding API in the browser-native design |
| `debounce` | not ported | dead code |

## Changes to this repository

### 1. `README.md` — a status banner at the top

```markdown
> **Frozen — historical reference.**
> This fork is preserved as the reference implementation for the Go/WASM port.
> No further TypeScript development. See [docs/PLAN.md](docs/PLAN.md) for the port map
> and https://github.com/webtyp/agent/blob/main/docs/PLAN.md for the current architecture.
```

### 2. Attribution in the derived Go repositories

`webtyp/vectordb` ports this code's logic and **must** carry a `NOTICE` file crediting
Nitai Aharoni and reproducing the MIT licence text, alongside its own licence. Master
index **D6**. This is a licence obligation, not a courtesy.

### 3. After the port lands

Archive the repository on GitHub (read-only). Do **not** delete it: the port map above
points at it, and the archived code is the only record of what the Go version is a port
of.

## Acceptance checklist

```bash
grep -q "Frozen — historical reference" README.md
git log --oneline -1          # no src/ changes in this commit
```

No build, no tests: nothing in `src/` changes.
