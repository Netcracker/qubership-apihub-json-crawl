---
name: json-crawl-using
description: Use when consuming json-crawl from another TypeScript project — crawling or cloning JSON trees, wiring path rules, or reading hook lifecycle semantics.
---

# Using json-crawl

Import from the package root only:

```typescript
import {
  crawl, syncCrawl, clone, syncClone, transform,
  breadthFirstTraverse, getNodeRules, mergeRules,
  isObject, isArray, anyArrayKeys,
  JSON_ROOT_KEY, JsonPath, CrawlRules, CrawlHook, SyncCrawlHook,
} from '@netcracker/qubership-apihub-json-crawl'
```

Internal paths under the package are unstable. The library ships dual ESM/CJS
builds with type declarations.

## Pick the right entry point

| API | Order | Async | Typical use |
|-----|-------|-------|-------------|
| `syncCrawl` | Depth-first | No | Tree walks, rule-driven transforms (most APIHUB code). |
| `crawl` | Depth-first | Yes (hooks may be async) | Same as above when hooks await I/O. |
| `syncClone` / `clone` | Depth-first | clone: optional async hooks | Deep copy; append custom hooks before the built-in clone hook runs. |
| `transform` | Depth-first | Yes | Mutable deep copy; `undefined` from a hook **deletes** the key/index. |
| `breadthFirstTraverse` | Breadth-first | No | Level-order visits only — see state caveat below. |

Prefer `syncCrawl`/`syncClone` in hot paths; reach for `crawl`/`clone` only when
hooks are genuinely async.

## Hook context and responses

Each hook receives `CrawlContext`: `{ value, path, key, state, rules? }`.

- At the **root**, `key` is `undefined` and `path` is `[]`.
- Return `void` or a partial `CrawlHookResponse`:

```typescript
syncCrawl(source, ({ value, path, key, state, rules }) => {
  // mutate state, optionally replace value/rules for descendants
  return { value, state, done: false }
}, { state: { myCounter: 0 }, rules: myRules })
```

| Field | Effect |
|-------|--------|
| `value` | Replaces the value seen by later hooks and descent logic. |
| `state` | Carried to child nodes (depth-first walkers only). |
| `rules` | Rules for child nodes. |
| `done: true` | Skip descending into this node's children. |
| `terminate: true` | Stop the entire walk. |
| `afterHooksHook` | Runnable after **all** hooks on this node, before children. |
| `exitHook` | Runnable when leaving this node (after children). |

Multiple hooks run in array order; each may patch `value`/`state`/`rules`.

## Path rules

Attach rules via `params.rules` (single object or array — arrays are merged
with `mergeRules`).

```typescript
type MyRules = CrawlRules<{ onLeaf?: (v: unknown) => void }>

const rules: MyRules = {
  '/**': { onLeaf: (v) => { /* every node */ } },
  '/*': { /* fallback for keys without a specific rule */ },
  '/^': {
    'x-': { /* longest prefix match on stringified key */ },
  },
  '/components': { /* exact key `components` */ },
}
```

`getNodeRules(parentRules, key, path, value)` is exported if you need the same
matching outside a crawl (e.g. diff engines comparing two trees).

Custom fields on your `CrawlRules<R>` generic (like `$` handlers in api-unifier)
sit alongside the reserved `/…` keys — read them from `ctx.rules` inside hooks.

## Clone state pattern

`syncClone`/`clone` use `CloneState`: `{ root, node, …yourFields }`. User hooks
run **before** the built-in clone hook. The clone hook writes into `state.node`
and descends with `state.node = state.node[key]`. The returned value is
`root[JSON_ROOT_KEY]` where `JSON_ROOT_KEY` is `'#'`.

## Sparse arrays and symbols

When mirroring crawl key order in your own logic, use `anyArrayKeys` — not
`Object.keys`, `[...arr.keys()]`, or index maps. Crawl visits negative indices,
holes, and symbol properties on arrays the way `Reflect.ownKeys` exposes them.

## `syncCrawl` skip-root mode

Pass `skipRootLevel: true` as the fourth argument to omit the root hook call
when the root is an object — useful when rules/hooks should only touch
properties, not the container.

## Cycles and `breadthFirstTraverse` state

- **No cycle detection** — track visited objects yourself or risk infinite loops.
- **`breadthFirstTraverse` resets `state` to `{}` on every node** — it does not
  propagate `params.state` between queue items. Use depth-first crawl if shared
  state matters.

## Common pitfalls

- **`mergeRules` throws on duplicate custom keys** — only path keys (`/` prefix)
  compose; two objects both defining `$` (or any non-path field) is an error.
- **`transform` deletes on `undefined`** — return the existing value explicitly
  if you mean to keep a property.
- **Mixing BFS and DFS semantics** — rule propagation matches, but only
  depth-first walkers carry `state` and honour `exitHook` timing the same way.
