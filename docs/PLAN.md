---
PLAN: "chore: congelar como referencia histórica; portar a Go"
TAG: none
EXECUTOR: unassigned
REVIEWER: none
---

> Parte del esfuerzo de búsqueda semántica nativa en el navegador. Índice maestro:
> https://github.com/webtyp/agent/blob/main/docs/PLAN.md

# Plan — congelar este fork, portar sus ideas a Go

## Estado: congelado

Este repositorio es un fork de
[`nitaiaharoni1/vector-storage`](https://github.com/nitaiaharoni1/vector-storage) (MIT). Se
conserva como la **implementación de referencia** para el port a Go y **no recibe más
desarrollo en TypeScript**: sin features, sin actualizaciones de dependencias, sin
refactors. Arreglos de bugs solo si hace falta clarificar un comportamiento durante el port.

Su valor es que es chico (217 líneas en `src/VectorStorage.ts`) y completo: demuestra el
ciclo entero —embeber, persistir en IndexedDB, búsqueda por coseno, desalojo LRU— en una
forma que se puede leer de una sentada. Eso vale la pena preservarlo exactamente como está.

## Lo que hizo bien, y se conserva

- **La forma del documento.** `text`, `metadata`, `timestamp`, `vector`, `vectorMag`,
  `hits` cubre lo que un almacén semántico realmente necesita, sin nada de más.
- **IndexedDB como persistencia, RAM como índice.** `this.documents` es el índice de
  búsqueda; IndexedDB es donde sobrevive a un recargar. Cargar el almacén en memoria al
  arrancar y buscar en memoria es la decisión correcta, y el port a Go la mantiene.
- **Un embedder enchufable.** `IVSOptions.embedTextsFn` deja que el llamador provea los
  embeddings en vez de hardcodear OpenAI. El port a Go promueve esto de opción a puerto
  central, `embed.Embedder`.
- **LRU por hits y después por antigüedad.** `removeDocsLRU` ordena por `hits` ascendente y
  después por `timestamp` ascendente. Política sensata; el port a Go conserva el orden y
  cambia solamente cómo se mide el tamaño.
- **Precomputar la magnitud.** `vectorMag` se guarda por documento para no recalcularla por
  consulta. El port a Go va un paso más allá — ver abajo.

## Lo que deliberadamente no se porta

Cada uno de estos es una corrección, no una preferencia de estilo:

| En el TypeScript | Por qué no se porta |
|---|---|
| `saveToIndexDbStorage()` hace `clear()` y después `put` de cada documento | Reescribe el corpus entero en **cada escritura**, incluso después de una lectura (`similaritySearch` lo llama para persistir los contadores de hits). E/S O(N) por operación. El port a Go escribe solo el shard que cambió. |
| `getObjectSizeInMB()` = `JSON.stringify(obj).length` | Serializa todo el corpus solo para medirlo, y mide el tamaño del JSON en vez del almacenado. El port a Go computa el tamaño aritméticamente (`N × Dim × 4` más la sobrecarga por fila) y lo contrasta con `navigator.storage.estimate()`. |
| `calculateSimilarityScores` devuelve `Array<[doc, number]>` y después `.sort()` | Asigna un arreglo de pares y ordena los N resultados para tomar k. El port a Go puntúa hacia un heap top-k de tamaño fijo: cero allocations por consulta. |
| `getCosineSimilarityScore` divide por ambas magnitudes por documento | El port a Go guarda los vectores normalizados L2, así que el coseno **es** el producto punto — un `sqrt` y una división ahorrados por documento por consulta. |
| `constants.DEFAULT_MAX_SIZE_IN_MB: 2048` | Contradice su propia documentación (`IVSOptions` dice "Defaults to 4.8. Cannot exceed 5" — un límite de la era de localStorage que quedó después de pasar a IndexedDB). El port a Go deriva el presupuesto de la cuota real. |
| `debounce()` en `helpers.ts` | Exportado, no importado en ningún lado, y `debounceTime` se lee hacia un campo que nunca se usa. Código muerto. |
| `documents.filter(d => d.text === doc.text)` para deduplicar | Comparación de strings O(N) por inserción, O(N·M) para un lote. El port a Go deduplica por hash de contenido en un índice. |
| El constructor llama a `loadFromIndexDbStorage()` sin await | Todo método público compite con la carga. Un `VectorStorage` usado inmediatamente después de construirlo busca sobre un corpus vacío y devuelve nada en silencio. El port a Go hace la carga explícita y devuelve error. |
| `console.error` cuando no se da API key, y sigue igual | Construye un objeto que no puede funcionar. El port a Go devuelve error desde su constructor. |
| Embeddings por HTTP a OpenAI por defecto | Requiere un viaje de red por consulta y pone una API key en JavaScript del lado del cliente, donde es pública. El port a Go genera embeddings **en el navegador**; ver **D4** del índice maestro. |

## Mapa de port

| TypeScript | Destino en Go | Notas |
|---|---|---|
| clase `VectorStorage<T>` | `webtyp.com/vectordb`.`Store` | los genéricos se reemplazan por un payload opaco `model.RawJSON` de metadatos |
| `addText` / `addTexts` / `addDocuments` | `Store.Add` | primero el lote; la forma de un solo documento lo envuelve |
| `similaritySearch` | `Store.Search` | devuelve `[]Match`; el embedding de la consulta no se devuelve |
| `IVSDocument<T>` | `vectordb.Doc` | `vectorMag` desaparece — los vectores se guardan normalizados |
| `IVSSimilaritySearchParams` | `vectordb.Query` | |
| `IVSFilterOptions` / `filterDocuments` | `vectordb.Query.IncludeTags`/`ExcludeTags` | filtrado de metadatos, reformulado — ver **O3** del índice maestro |
| `calcVectorMagnitude` / `getCosineSimilarityScore` / `normalizeScore` | `webtyp.com/vector`.`Norm`, `Dot`, `Normalize` | |
| `calculateSimilarityScores` + `.sort().slice(0, k)` | `vector.TopK` | |
| `removeDocsLRU` | `Store.evict` | mismo orden, distinta medición de tamaño |
| `initDB` / `loadFromIndexDbStorage` / `saveToIndexDbStorage` | `webtyp.com/indexdb` vía `storage.Conn` | el driver es dueño de IndexedDB; el almacén es dueño de la política |
| `embedTexts` (HTTP a OpenAI) | `webtyp.com/embed`.`Embedder` | reformulado de opción a puerto |
| `ICreateEmbeddingResponse` | no se porta | no hay API HTTP de embeddings en el diseño nativo de navegador |
| `debounce` | no se porta | código muerto |

## Cambios en este repositorio

### 1. `README.md` — un cartel de estado arriba de todo

```markdown
> **Frozen — historical reference.**
> This fork is preserved as the reference implementation for the Go/WASM port.
> No further TypeScript development. See [docs/PLAN.md](docs/PLAN.md) for the port map
> and https://github.com/webtyp/agent/blob/main/docs/PLAN.md for the current architecture.
```

### 2. Atribución en los repositorios Go derivados

`webtyp/vectordb` porta la lógica de este código y **debe** llevar un archivo `NOTICE`
acreditando a Nitai Aharoni y reproduciendo el texto de la licencia MIT, junto a su propia
licencia. **D6** del índice maestro. Esto es una obligación de licencia, no una cortesía.

### 3. Después de que el port aterrice

Archivar el repositorio en GitHub (solo lectura). **No** borrarlo: el mapa de port de arriba
apunta a él, y el código archivado es el único registro de qué es lo que la versión en Go
porta.

## Checklist de aceptación

```bash
grep -q "Frozen — historical reference" README.md
git log --oneline -1          # sin cambios en src/ en este commit
```

Sin build, sin tests: nada en `src/` cambia.
