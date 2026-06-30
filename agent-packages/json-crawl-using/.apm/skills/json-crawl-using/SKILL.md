---
name: json-crawl-using
description: Use when consuming json-crawl from another TypeScript project — crawling or cloning JSON trees, wiring path rules, or reading hook lifecycle semantics.
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

Typical consumer usage falls into four roles: **type-only** (`JsonPath`),
**guards** (`isObject`/`isArray`/`anyArrayKeys`), **depth-first walks**
(`syncCrawl` + hooks + rules), and **deep copies** (`syncClone` + hooks +
rules). Async `crawl`/`clone`, library `transform`, and `breadthFirstTraverse`
are available when hooks are genuinely async, you need in-place mutation, or
level-order traversal is required; most pipelines use the sync depth-first APIs.

## `JsonPath` — shared address type

`JsonPath` is `PropertyKey[]`. Store document locations as json-crawl paths:
diff entries, origin chains, tree node ids (often `'#' + buildPointer(path)`),
validation errors, and UI change highlights. Keys in a path are whatever the
crawler visited — strings, numbers, and symbols (though most hooks skip symbol
keys; see below).

## Pick sync crawl vs sync clone

| API | When to use it |
|-----|----------------|
| `syncCrawl` | Read-only or in-place tree walks — compare/merge, tree builders, hash scans, validation passes. Mutate **source** objects only when the hook intentionally writes into `value` or the parent container. |
| `syncClone` | Produce a new object graph — normalize/validate/unify/merge pipelines, identity-preserving clones with cycle repair, rule-driven hash views. User hooks run **before** the built-in clone hook. Result is `root[JSON_ROOT_KEY]` (`'#'`). |

Both accept `{ state, rules }` in `params`. Pass `rules` as a single object or an
array (merged via `mergeRules`).

## Hook context and control flags

Each hook receives `{ value, path, key, state, rules? }`. At the **root**,
`key === undefined` and `path === []`.

Return `void` or a partial response. Fields that matter in consumer code:

| Field | Typical use |
|-------|-------------|
| `value` | Replace node value for later hooks / descent (transformers, adapters). |
| `state` | Carry parent context, merge caches, depth counters, mapping stacks — **must** be returned when you replace it (`{ ...state, parent: newNode }`). |
| `rules` | Override rules for descendants (rare; usually rules come from the static rule tree). |
| `done: true` | **Prune subtree** — cycle guard hit, symbol key, ignored branch, hash skip, or "already built this node". |
| `terminate: true` | Abort entire walk (equality helper, early exit). |
| `exitHook` | Deferred work **after children** — merge-cache cleanup, cycle-clone completion. |
| `afterHooksHook` | Runs after all hooks on the node, before children — marks partial completion so re-entrant reference graphs get a stable back-reference. |

Run **cycle guards first**, then value transformers, then node-creation hooks.
Multiple hooks merge responses in array order.

### Symbol keys

Metadata attached via `symbol` properties is usually not part of the public
document surface. Hooks in compare, tree builders, and hash pipelines
consistently bail out:

```typescript
if (typeof key === 'symbol') {
  return { done: true }
}
```

Some cycle guards may still pass `{ value }` for excluded components — follow
the local hook contract, but default to skipping symbols.

## Extending `CrawlRules<R>` — the consumer pattern

Reserved path keys (`/**`, `/*`, `/^`, `/keyName`) select **where** a rule
applies. Custom fields on the generic `R` select **what happens** there. Read
them from `ctx.rules` inside hooks; do not re-parse paths by hand when rules
already encode the mapping.

Define a payload type `R` and attach it under path keys:

```typescript
type MyRule = {
  handler?: (value: unknown) => unknown
  skipSubtree?: boolean
  nodeKind?: string
}

const rules: CrawlRules<MyRule> = {
  '/properties': {
    '/*': { nodeKind: 'property' },
  },
  '/^': {
    'x-': { skipSubtree: false, '/*': {}, '/**': {} },
  },
  $: { handler: rootHandler },
}
```

Common payload shapes across consumers:

- **Normalization / merge** — flags such as `merge`, `validate`, `unify`,
  `hashOwner`, `referenceHandler`. OpenAPI `x-*` extensions often use
  `CrawlPrefixRules<R>` under `'/^'` with an `'x-'` prefix entry.
- **Compare / diff** — a classifier at **`rules.$`**, plus fields like
  `compare`, `mapping`, `adapter`, `ignoreDifference`, `ignoreKeyDifference`,
  `description`, `newCompareScope`, `syntheticDiffs`. Set
  `ignoreDifference: true` → `{ done: true }` at hook entry so **no diffs and
  no child crawl** for that subtree.
- **Tree building** — `kind` (node type to create), `transformers` (array
  reduced over `value` before node creation), `complex` (simple vs complex node
  split). Path keys often use **rule factories** returning nested rules, e.g.
  `'/properties': { '/*': () => schemaCrawlRules(kind.property) }`.

Path keys may be **functions** `(ctx: CrawlRulesContext) => CrawlRules<R>` for
context-sensitive rule trees (e.g. json-schema `/items` branching on numeric vs
non-numeric keys).

## `getNodeRules` outside a crawl

Call when you already know `(parentRules, key, path, value)` but are not
inside a json-crawl walk — e.g. resolving compare rules for the next merged
key, picking combiner item rules before nested compare, or selecting
merge/unify behaviour per property. The `path` argument may be a sentinel, not
the literal crawl path.

Signature: `getNodeRules(rules, key, path, value) → CrawlRules<R> | undefined`.
Matching order: exact `/key` → longest `/^` prefix → `/*` → `/**` (global
re-attaches `/**` on the result).

## Cycle handling — three established patterns

json-crawl does **not** detect cycles. Consumers must track visited references:

1. **`syncClone` + source→copy Map** — Map from source object to clone state; on
   re-entry assign existing clone and `{ done: true }`; use `afterHooksHook` /
   `exitHook` to finalize partial copies. Required for `$ref` graphs and
   identity-preserving clones. At root, remaps `key ?? JSON_ROOT_KEY` when
   writing into `state.node`.

2. **`syncCrawl` + `Set<unknown>`** — add object to set on entry; on re-entry
   `{ done: true }` (optionally pass `undefined` value to skip nested
   materialization).

3. **`syncCrawl` + `Map<unknown, BuiltNode>`** — cache built nodes by **source
   object identity**; on re-entry create a cycle back-reference node, attach to
   parent, `{ done: true }`. Copy the cache into new state when descending
   (`new Map(state.alreadyConvertedValuesCache)`).

Pick the pattern that matches whether you clone or walk in place and whether
downstream needs a back-reference node.

## Sparse arrays and `anyArrayKeys`

When aligning custom iteration with crawl order (array mapping, deep equals,
origin walks), use `anyArrayKeys` — not `Object.keys`, spread indices, or
`.map((_, i) => i)`. Crawl visits holes, negative indices, and symbol keys via
`Reflect.ownKeys` semantics.

## Rule trees — prefix and spread

Extension-key rules often repeat this shape:

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

Nest domain rule objects under path keys and spread shared fragments —
`mergeRules` composes path keys but **throws if two merged objects define the
same non-path field** (e.g. two `$` handlers).

## `syncCrawl` skip-root mode

Fourth argument `skipRootLevel: true` — when the root is an object, hooks never
run for the root visit; descent starts at top-level keys. Use when the
container object itself should not be processed.

## Common pitfalls

- **`done: true` vs `terminate: true`** — `done` skips one subtree; `terminate`
  stops the whole walk. Do not use `terminate` for subtree suppression.
- **`ignoreDifference` must short-circuit before diff creation** — returning
  `done` late still emits child diffs.
- **Clone hooks and sharing** — do not recreate nested objects that other
  references share; clone into `state.node` or mutate in place deliberately.
- **`mergeRules` duplicate custom keys** — split shared payloads across path
  keys, not duplicate top-level rule fields.
- **BFS (`breadthFirstTraverse`)** — does not propagate `params.state`. Prefer
  depth-first crawl when state must flow through the walk.
