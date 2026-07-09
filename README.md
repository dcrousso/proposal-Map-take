# `Map.prototype.take` & `WeakMap.prototype.take`

A proposal to add a `take(key)` method to `Map` and `WeakMap` that **removes** the entry for `key` and returns its **value** (or `undefined` if the key was absent) in a single operation.

- **Stage:** 0 (sketch / pre-proposal)
- **Champion:** Devin Rousso
- **Authors:** Devin Rousso

## Motivation

"Read a value and remove it in the same step" is one of the most common things people do with a map (e.g. consume a pending callback keyed by request id, pop a job off a work queue, evict and inspect a cache entry, hand off ownership of a buffered chunk, drain a "waiting for X" table, etc.).

JavaScript can express every other member of the CRUD quartet as a single call that returns something useful (e.g. `map.get(k)`, `map.set(k, v)` (returns the map), and `map.has(k)`) **except** the "remove and read" combination.

[`Map.prototype.delete`](https://tc39.es/ecma262/#sec-map.prototype.delete) returns a **boolean** telling you whether anything was removed (i.e. it throws the value away).

So today you must write the value-preserving version by hand, which requires **two lookups** of the same key:

```js
function take(map, key) {
  const value = map.get(key); // lookup #1
  map.delete(key);            // lookup #2
  return value;
}
```

This has real downsides:

1. **Two lookups instead of one.**  For hot paths (dispatch tables, per-frame caches, etc.) the extra lookup is pure overhead an engine could avoid if the operation were a single method.
2. **It's a footgun to inline.**  `const v = map.get(k); map.delete(k); return v;` is easy to reorder incorrectly (i.e. `delete` before capturing the value) and the `delete` boolean result is silently discarded so linters can't help.
3. **Everyone re-implements it.**  The helper is reinvented under a dozen names (e,g, `getAndDelete`, `getAndRemove`, `pop`, `pull`, `popEntry`, `take`, `fetch`, etc.) in codebase after codebase.

## Proposal

```js
Map.prototype.take(key)     // the removed value, or undefined if key was absent
WeakMap.prototype.take(key) // the removed value, or undefined if key was absent
```

`take(key)` is defined to be observably equivalent to reading `get(key)` and then performing `delete(key)`, returning the value that `get` would have returned.

It mutates the map in place and returns the value directly (i.e. it does **not** return the map) because the whole point is to hand the value back to the caller.

### Examples

```js
// Consume request.
const resolveForRequestId = new Map();
function onResponse(id, payload) {
  const resolve = resolveForRequestId.take(id); // grab it and clear it in one step
  resolve?.(payload);
}

// Move ownership.
const buffers = new Map();
// ...
const chunk = buffers.take(streamId); // caller now owns `chunk`

// One-off logic.
const stateForObject = new WeakMap();
function consume(obj) {
  const state = stateForObject.take(obj);
  return state;
}
```

### Semantics

| Situation | `take(key)` returns | Side effect |
| --- | --- | --- |
| `key` present with value `v` | `v` | entry removed |
| `key` present with value `undefined` | `undefined` | entry removed |
| `key` absent | `undefined` | none |
| `WeakMap` and `key` cannot be held weakly | `undefined` | none |
| `this` is not a `Map`/`WeakMap` | — | throws `TypeError` |

Like `get`, a return of `undefined` is ambiguous between "absent" and "present with value `undefined`".

If you need to tell them apart then you can always call `has(key)` first.

This mirrors `get` exactly, so it adds no new ambiguity to the language.

## Prior Art

"Remove and return the value" is the **default** shape of this operation in most languages' standard libraries.

JavaScript is the outlier in returning only a boolean from `delete`.

| Language | API | Removes & returns value? | Notes |
| --- | --- | --- | --- |
| **Rust** | [`HashSet::take(&value) -> Option<T>`](https://doc.rust-lang.org/std/collections/struct.HashSet.html#method.take) | ✅ | Direct precedent for the **name** `take`: "removes and returns the value in the set, if any". |
| **Rust** | [`HashMap::remove(&k) -> Option<V>`](https://doc.rust-lang.org/std/collections/struct.HashMap.html#method.remove) / [`remove_entry`](https://doc.rust-lang.org/std/collections/struct.HashMap.html#method.remove_entry) | ✅ | Returns the value (or the `(k, v)` pair). |
| **Python** | [`dict.pop(key[, default])`](https://docs.python.org/3/library/stdtypes.html#dict.pop) | ✅ | Returns value; raises `KeyError` if absent and no default given. |
| **Ruby** | [`Hash#delete(key)`](https://ruby-doc.org/core/Hash.html#method-i-delete) | ✅ | Returns the value (or `nil`); optional block for the missing case. |
| **Java** | [`Map.remove(Object key) -> V`](https://docs.oracle.com/javase/8/docs/api/java/util/Map.html#remove-java.lang.Object-) | ✅ | Returns the previous value, or `null`. |
| **C#** | [`Dictionary.Remove(key, out value) -> bool`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.dictionary-2.remove#system-collections-generic-dictionary-2-remove(-0-1@)) | ✅ | The `out` overload (added in .NET Core 3.0) hands back the removed value. |
| **Swift** | [`Dictionary.removeValue(forKey:) -> Value?`](https://developer.apple.com/documentation/swift/dictionary/removevalue(forkey:)) | ✅ | Returns the removed value, or `nil`. |
| **Kotlin** | [`MutableMap.remove(key): V?`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/-mutable-map/remove.html) | ✅ | Returns the previous value, or `null`. |
| **C++** | [`std::map::extract` / `unordered_map::extract`](https://en.cppreference.com/w/cpp/container/unordered_map/extract) (C++17) | ✅ | Removes and hands back a node handle owning the key/value. |
| **Perl** | [`delete $hash{$key}`](https://perldoc.perl.org/functions/delete) | ✅ | The `delete` operator returns the deleted value. |
| **Go** | [`delete(m, k)`](https://go.dev/ref/spec#Deletion_of_map_elements) | ❌ | No return; you must write `v, ok := m[k]; delete(m, k)` — the same two-step JS is stuck with. |
| **JavaScript** | [`Array.prototype.pop() -> element`](https://tc39.es/ecma262/#sec-array.prototype.pop) | ✅ | JS's existing remove-and-return: pops the **last** element and returns it (or `undefined`). `take` is the by-**key** analogue for maps. |
| **JavaScript (today)** | [`Map.prototype.delete -> boolean`](https://tc39.es/ecma262/#sec-map.prototype.delete) | ❌ | Returns only whether something was removed; the value is discarded. |

### Precedents

`take` is not a foreign shape for JavaScript as the language already has a "remove an element and return its value" method in `Array.prototype.pop`.

It mutates the collection in place, returns the removed element, and yields `undefined` when there is nothing to remove.

`take` applies that exact contract to keyed collections (i.e. addressed by key instead of by position):

| | `Array.prototype.pop()` | `Map.prototype.take(key)` |
| --- | --- | --- |
| Selects what to remove by | position (last index, LIFO) | key |
| Argument | none | the key |
| Returns | the removed element | the removed value |
| When nothing matches | `undefined` (empty array) | `undefined` (absent key) |
| Mutates in place | ✅ | ✅ |
| Returns the container for chaining | ❌ (returns the element) | ❌ (returns the value) |

Because `pop` already establishes "mutate and hand back the value" as an idiomatic JavaScript pattern, `take` is a small, consistent extension of it rather than a new concept (i.e. it just removes by key).

### Naming

- **`take`** matches Rust's [`HashSet::take`](https://doc.rust-lang.org/std/collections/struct.HashSet.html#method.take) ("remove and return") and reads naturally "take the value out of the map".
- **`pop`** is misleading as `Array.prototype.pop takes no argument and removes from the end (i.e. LIFO) instead of by index.
- **`remove`** would sit confusingly beside the existing `delete` (i.e. two spellings with different return types) and collides with the many userland `remove` methods.

#### Alternatives

`getAndDelete` is an equally acceptable name.

Its advantage is that it extends the existing "family" of `get`, `getOrInsert`, `getOrInsertComputed`, etc. so the read-then-mutate relationship is explicit and greppable (i.e. `getAndDelete` reads literally as "`get` and then `delete`").

## Usage

> Counts below come from GitHub `GET /search/code` which is a token-based index, covers only the default branch of public repos, and does **not** verify that the matched tokens act on the same key/variable.

| Query | Files |
| --- | ---: |
| `"getAndDelete(" language:TypeScript` | ~260 |
| `"getAndDelete(" language:JavaScript` | ~281 |
| `"popEntry(" in:file` (JS/TS) | ~3,270 |
| `getAndRemove in:file` (all langs, token match) | ~48,000 |

| Query | Files |
| --- | ---: |
| `cache.get cache.delete language:TypeScript` | ~143,000 |
| `map.get map.delete language:JavaScript` | ~99,800 |
| `"get(key)" "delete(key)" language:TypeScript` | ~44,600 |

## Examples

```js
// pull out the pending request/callback when its response arrives
let responseData = this._pendingResponses.take(sequenceId) || {};

// remove and read a worker as it goes away
let worker = this._workerForId.take(workerId);

// adopt resources that were buffered for a target
let resources = this._orphanedResources.take(target.identifier);
```

## Questions

- `take` vs `getOrDelete` vs something else?
- This sketch returns `undefined`, mirroring `get`, so `take` is a drop-in replacement for the `get` and `delete` pair, but an alternative could be a Python-style optional default `take(key, defaultValue)`.
- Is it worth adding `Set.prototype.take` and/or `WeakSet.prototype.take` for parity like how `Set.prototype.entries` returns a two-value array to match `Map.prototype.entries`?

## Polyfill

```js
'use strict';

(function installMapTakePolyfill() {
  function defineTake(Ctor) {
    if (typeof Ctor !== 'function' || typeof Ctor.prototype.take === 'function')
        return;

    const get = Ctor.prototype.get;
    const del = Ctor.prototype.delete;

    function take(key) {
      const value = get.call(this, key);
      del.call(this, key);
      return value;
    }

    Object.defineProperty(Ctor.prototype, 'take', {
      value: take,
      writable: true,
      enumerable: false,
      configurable: true,
    });
  }

  defineTake(typeof Map === 'function' ? Map : undefined);
  defineTake(typeof WeakMap === 'function' ? WeakMap : undefined);
})();
```
