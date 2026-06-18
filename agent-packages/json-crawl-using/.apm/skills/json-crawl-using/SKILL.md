---
name: json-crawl-using
description: Use when consuming json-crawl from another TypeScript project — crawling or cloning JSON trees, wiring path rules, or reading hook lifecycle semantics in APIHUB engines.
---

# Using json-crawl

Import from the package root only:

```typescript
import {
  syncCrawl, syncClone,
  getNodeRules, mergeRules,
  isObject, isArray, anyArrayKeys,
  JSON_ROOT_KEY, JsonPath, CrawlRules, CrawlPrefixRules,
  CrawlRulesContext, SyncCrawlHook, SyncCloneHook, CloneState,
} from '@netcracker/qubership-apihub-json-crawl'
```

Internal paths under the package are unstable.

Across **api-unifier**, **api-diff**, and **api-doc-viewer**, usage falls into
four roles: **type-only** (`JsonPath`), **guards** (`isObject`/`isArray`/
`anyArrayKeys`), **depth-first walks** (`syncCrawl` + hooks + rules), and
**deep copies** (`syncClone` + hooks + rules). Async `crawl`/`clone`, library
`transform`, and `breadthFirstTraverse` are not used in those three repos —
reach for them only when hooks are genuinely async or you need in-place mutation
or level-order traversal.

## `JsonPath` — shared address type

`JsonPath` is `PropertyKey[]`. Every APIHUB layer stores document locations as
json-crawl paths: diff entries, origin chains, tree node ids (`'#' +
buildPointer(path)` from api-unifier), validation errors, and UI change
highlights. Keys in a path are whatever the crawler visited — strings, numbers,
and symbols (though most hooks skip symbol keys; see below).

## Pick sync crawl vs sync clone

| API | When APIHUB uses it |
|-----|---------------------|
| `syncCrawl` | Read-only or in-place tree walks — compare/merge (api-diff), tree builders (api-doc-viewer), hash scan (api-unifier), deprecated-item scan. Mutate **source** objects only when the hook intentionally writes into `value`/`parent` (compare merge writes into `mergedJso`). |
| `syncClone` | Produce a new object graph — normalize/validate/unify/merge pipelines (api-unifier), identity-preserving clones with cycle repair (`define-ddlapi-origins`), rule-driven hash views. User hooks run **before** the built-in clone hook. Result is `root[JSON_ROOT_KEY]` (`'#'`). |

Both accept `{ state, rules }` in `params`. Pass `rules` as a single object or an
array (merged via `mergeRules`).

## Hook context and control flags

Each hook receives `{ value, path, key, state, rules? }`. At the **root**,
`key === undefined` and `path === []`.

Return `void` or a partial response. Fields that matter in APIHUB code:

| Field | Typical APIHUB use |
|-------|---------------------|
| `value` | Replace node value for later hooks / descent (tree builders after rule transformers; compare after adapters). |
| `state` | Carry parent tree node, merge caches, `dataLevel`, mapping stacks — **must** be returned when you replace it (`{ ...state, parent: newNode }`). |
| `rules` | Override rules for descendants (rare; usually rules come from the static rule tree). |
| `done: true` | **Prune subtree** — cycle guard hit, symbol key, `ignoreDifference`, hash skip, or "already built this node". |
| `terminate: true` | Abort entire walk (equality helper, early exit). |
| `exitHook` | Deferred work **after children** — compare merge-cache cleanup, cycle-clone completion in `createCycledJsoHandlerHook`. |
| `afterHooksHook` | Runs after all hooks on the node, before children — cycle clone marks partial completion so re-entrant `$ref` graphs get a stable reference. |

Run **cycle guards first**, then value transformers, then node-creation hooks
(api-doc-viewer `createTreeBuildingHooks` order). Multiple hooks merge responses
in array order.

### Symbol keys

Metadata attached via `symbol` properties is not part of the spec surface.
Hooks in compare, tree builders, and hash consistently bail out:

```typescript
if (typeof key === 'symbol') {
  return { done: true }
}
```

Exception: api-doc-viewer cycle guard may still `{ value }` for excluded
components — follow the local hook, but default to skipping symbols.

## Extending `CrawlRules<R>` — the APIHUB pattern

Reserved path keys (`/**`, `/*`, `/^`, `/keyName`) select **where** a rule
applies. Custom fields on the generic `R` select **what happens** there. Read
them from `ctx.rules` inside hooks; do not re-parse paths by hand when rules
already encode the mapping.

**api-unifier** — `CrawlRules<NormalizationRule>` (`NormalizationRules`):

- Hooks check `rules.merge`, `rules.validate`, `rules.unify`, `rules.hashOwner`,
  `rules.referenceHandler`, etc.
- OpenAPI `x-*` extensions: `'/^': { 'x-': { isExtension: true, validate: …, merge: … } }`
  using `CrawlPrefixRules<NormalizationRule>`.

**api-diff** — `CrawlRules<CompareRule>`:

- Classifier at **`rules.$`** (constant `CLASSIFIER_RULE`).
- Other fields: `compare`, `mapping`, `adapter`, `ignoreDifference`,
  `ignoreKeyDifference`, `description`, `newCompareScope`, `syntheticDiffs`.
- Same `x-` prefix pattern under `'/^'`.
- `ignoreDifference: true` → `{ done: true }` at hook entry so **no diffs and
  no child crawl** for that subtree (after-value still copied into merge).

**api-doc-viewer** — `CrawlRules<SchemaCrawlRule>`:

- `rules.kind` — tree node type to create.
- `rules.transformers` — array reduced over `value` before node creation.
- `rules.complex` — simple vs complex node split.
- Path keys often use **rule factories** returning nested rules, e.g.
  `'/properties': { '/*': () => jsonSchemaCrawlRules(kind.property) }`.

Path keys may be **functions** `(ctx: CrawlRulesContext) => CrawlRules<R>` for
context-sensitive rule trees (json-schema `/items` branching on numeric vs
non-numeric keys).

## `getNodeRules` outside a crawl

Call when you already know `(parentRules, key, path, value)` but are not
inside a json-crawl walk:

- **api-diff** `createChildContext` — resolve compare rules for the next merged
  key (path is often a sentinel like `ANY_COMBINER_PATH`, not the literal crawl
  path).
- **api-diff** `jsonSchema.resolver` — resolve combiner item rules before
  nested compare.
- **api-unifier** resolvers — pick merge/unify behaviour per property.

Signature: `getNodeRules(rules, key, path, value) → CrawlRules<R> | undefined`.
Matching order: exact `/key` → longest `/^` prefix → `/*` → `/**` (global
re-attaches `/**` on the result).

## Cycle handling — three established patterns

json-crawl does **not** detect cycles. APIHUB repos each implement guards:

1. **`syncClone` + `createCycledJsoHandlerHook`** (api-unifier) — Map from
   source object to copy state; on re-entry assign existing clone and
   `{ done: true }`; use `afterHooksHook` / `exitHook` to finalize partial
   copies. Required for `$ref` graphs and identity-preserving ddlapi clones.
   At root, remaps `key ?? JSON_ROOT_KEY` when writing into `state.node`.

2. **`syncCrawl` + `Set<unknown>`** (api-unifier hash) — add object to set on
   entry; on re-entry `{ done: true }` (optionally pass `undefined` value to
   skip nested hash materialization).

3. **`syncCrawl` + `Map<unknown, TreeNode>`** (api-doc-viewer) — cache built
   nodes by **source object identity**; on re-entry create a cycle clone node,
   attach to parent, `{ done: true }`. Copy the cache into new state when
   descending (`new Map(state.alreadyConvertedValuesCache)`).

Pick the pattern that matches whether you clone or walk in place and whether
downstream needs a back-reference node.

## Sparse arrays and `anyArrayKeys`

When aligning custom iteration with crawl order (compare array mapping, deep
equals, origin walks), use `anyArrayKeys` — not `Object.keys`, spread indices,
or `.map((_, i) => i)`. Crawl visits holes, negative indices, and symbol keys
via `Reflect.ownKeys` semantics.

## Rule trees — prefix and spread

OpenAPI/AsyncAPI extension rules repeat this shape in both unifier and diff:

```typescript
const extensionPrefixRules: CrawlPrefixRules<MyRulePayload> = {
  'x-': {
    /* payload for any x-* key */
    '/*': { /* every child under that extension */ },
    '/**': { /* all descendants */ },
  },
}
export const rules = { '/^': extensionPrefixRules }
```

Nest domain rule objects under path keys and spread shared fragments
(`...openApiSpecificationExtensionRules`) — `mergeRules` composes path keys but
**throws if two merged objects define the same non-path field** (e.g. two `$`
handlers).

## `syncCrawl` skip-root mode

Fourth argument `skipRootLevel: true` — when the root is an object, hooks never
run for the root visit; descent starts at top-level keys. Rare in the three
repos but available when the container should not be processed.

## Common pitfalls

- **`done: true` vs `terminate: true`** — `done` skips one subtree; `terminate`
  stops the whole walk. Compare uses `done` heavily; do not use `terminate` for
  subtree suppression.
- **`ignoreDifference` must short-circuit before diff creation** — returning
  `done` late still emits child diffs.
- **Clone hooks and sharing** — api-unifier comments warn: do not recreate nested
  objects that other references share; clone into `state.node` or mutate in place
  deliberately.
- **`mergeRules` duplicate custom keys** — split shared payloads across path
  keys, not duplicate top-level rule fields.
- **BFS (`breadthFirstTraverse`)** — does not propagate `params.state`; unused in
  APIHUB tree/diff/unifier pipelines. Prefer depth-first crawl.
