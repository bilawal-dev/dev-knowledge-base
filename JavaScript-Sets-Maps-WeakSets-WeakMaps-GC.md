# Comprehensive Guide: JavaScript Sets, Maps, WeakSets, WeakMaps & Garbage Collection

This document consolidates all my learnings about JavaScript Set, Map, WeakSet, WeakMap, garbage collection, object reachability, scope-based memory behavior, and real-world backend/server use cases.
The goal is to serve as a go-to reference to quickly recall what to use, when, and why.

## 1. Core Mental Models (Read This First)

- **Garbage Collection** is based on reachability, not variables.
- **Scope** decides lifecycle.
- **Map / Set** hold strong references (own objects).
- **WeakMap / WeakSet** hold weak references (borrow objects).
- If **you own the lifecycle** → Map / Set
- If **the object owns the lifecycle** → WeakMap / WeakSet

## 2. Garbage Collection (GC) – The Foundation

### What is Garbage Collection?

Automatic memory cleanup of objects that are no longer reachable.

### Core Rule

- If an object is **reachable** → it stays in memory
- If an object is **unreachable** → GC may free it

### How Objects Become Unreachable

An object is garbage-collected when:

- A function finishes and no closures reference it
- A request completes
- A job finishes
- A DOM node is removed
- No global or long-lived structure references it

⚠️ **Setting `obj = null` is NOT required**
It only removes one reference.

### Strong vs Weak References

| Reference Type | Prevents GC |
|---------------|-------------|
| Local variable | ✅ |
| Global variable | ✅ |
| Object property | ✅ |
| Array | ✅ |
| Map | ✅ |
| Set | ✅ |
| WeakMap | ❌ |
| WeakSet | ❌ |

## 3. Set – Unique Value Collection

### What Problem Set Solves

Store unique values and perform fast existence checks.

### Core Characteristics

- Stores values only
- No duplicates
- Maintains insertion order
- Any type allowed
- Faster `.has()` than array `.includes()`

### Common Set Methods

```javascript
const set = new Set();

set.add(value);
set.has(value);
set.delete(value);
set.clear();
set.size;
```

### Typical Use Cases

**Uniqueness checks**
```javascript
const ids = new Set(arr);
ids.size === arr.length;
```

**Tracking visited items**
- Graph traversal
- Deduplication
- Feature flags
- Request-scoped uniqueness

```javascript
function handler() {
  const seen = new Set();
}
```

✔ Automatically GC'd after function ends

### Important Note (Objects in Set)

Objects are compared by **reference**, not value.

```javascript
new Set([{ id: 1 }, { id: 1 }]).size; // 2
```

## 4. Map – Proper Key-Value Store

### Why Map Exists

Objects only allow string/symbol keys.
Maps allow **any type** as keys.

### Common Map Methods

```javascript
const map = new Map();

map.set(key, value);
map.get(key);
map.has(key);
map.delete(key);
map.clear();
map.size;
```

### Why Map Over Object {}

| Feature | Object | Map |
|---------|--------|-----|
| Key types | string only | any |
| Iteration | awkward | clean |
| Size | manual | O(1) |
| Prototype risk | yes | no |
| Runtime cache | meh | ideal |

### Typical Map Use Cases

- **Global in-memory cache**
  ```javascript
  const cache = new Map();
  ```
- Lookup tables
- LRU caches
- ID → data mapping
- Runtime metrics

## 5. Scope Decides Everything (Very Important)

### Function / Request Scope

```javascript
function handler() {
  const map = new Map();
}
```

- Map becomes unreachable after function returns
- GC cleans it
- No WeakMap needed

### Global Scope

```javascript
const map = new Map();
```

- Lives for entire process lifetime
- Must manually delete entries
- Risk of memory leaks

## 6. Why Map / Set Can Cause Memory Leaks

```javascript
const map = new Map();

let obj = {};
map.set(obj, "data");
obj = null;
```

❌ Object is still reachable via map
❌ GC cannot free it
❌ Memory accumulates silently

This is the exact problem WeakMap solves.

## 7. WeakMap – Object-Linked Metadata

### What WeakMap Is

- Keys **must be objects**
- Values can be anything
- Keys are held **weakly**
- Does not prevent GC

### WeakMap Limitations (By Design)

- ❌ No `.size`
- ❌ No iteration
- ❌ No primitive keys

These limitations exist to hide GC behavior.

### Core WeakMap Example

```javascript
const wm = new WeakMap();

let user = {};
wm.set(user, "meta");

user = null; // GC can clean everything
```

### When WeakMap Entry Disappears

When the key object becomes **unreachable**
Not when you delete it explicitly.

## 8. WeakSet – Object Existence Tracking

### WeakSet Characteristics

- Stores objects only
- No duplicates
- Weak references
- No iteration

### Typical WeakSet Use Case

```javascript
const processed = new WeakSet();

function process(obj) {
  if (processed.has(obj)) return;
  processed.add(obj);
}
```

Automatically cleans when objects die.

## 9. WeakMap vs Map Inside Scope (Critical Insight)

These two are **equivalent** in safety:

```javascript
function handler(req) {
  const map = new Map();
  map.set(req, data);
}

const wm = new WeakMap();
function handler(req) {
  wm.set(req, data);
}
```

### Why?

- **First:** Map dies with function
- **Second:** Key dying removes entry

✅ **WeakMap is only needed when the Map outlives the object.**

## 10. Real-World Backend Use Cases

### 1. Request Metadata (Express / Fastify)

```javascript
const requestMeta = new WeakMap();

app.use((req, res, next) => {
  requestMeta.set(req, { start: Date.now() });
  next();
});
```

Request finishes → req unreachable → metadata removed

### 2. WebSocket Connections

```javascript
const socketData = new WeakMap();

wss.on("connection", (ws) => {
  socketData.set(ws, { userId });
});
```

Socket closes → auto cleanup

### 3. Job / Queue Metadata

```javascript
const jobMeta = new WeakMap();
jobMeta.set(job, { startedAt });
```

Job finishes → GC cleans it

### 4. ORM Entity Caching

```javascript
const computed = new WeakMap();

function getComputed(entity) {
  if (!computed.has(entity)) {
    computed.set(entity, heavyCompute(entity));
  }
  return computed.get(entity);
}
```

Entity lifecycle controls cache lifecycle

## 11. When NOT to Use WeakMap / WeakSet

❌ Business data
❌ Anything needing iteration
❌ LRU caches
❌ TTL-based caches
❌ Debuggable state
❌ Cross-request persistence

**Use Map instead.**

## 12. Decision Table (Quick Recall)

| Scenario | Use |
|----------|-----|
| Global cache by ID | Map |
| Uniqueness check | Set |
| Request-scoped state | Map / Set |
| Global object-keyed metadata | WeakMap |
| Tracking object existence | WeakSet |
| Cache needing TTL | Map |
| Lifecycle not yours | WeakMap |

## 13. One-Line Recall Hooks

- **Set** → "Is it unique?"
- **Map** → "Give me value for this key"
- **WeakMap** → "Attach data without owning lifecycle"
- **WeakSet** → "Have I seen this object?"
- **GC** → "Unreachable = collectible"
- **Scope > structure**

## 14. Final Takeaways

- WeakMap / WeakSet are **memory safety tools**
- They are **rare but critical** in long-running servers
- **Scope often replaces** the need for WeakMap
- Global Maps **must be cleaned manually**
- Garbage collection follows **reachability, not intent**

## 15. Recommended Practice

1. **Default to Map / Set**
2. **Let scope handle cleanup first**
3. **Use WeakMap only when:**
   - Keys are objects
   - Lifecycle isn't yours
   - Data is global
4. **Never use WeakMap for business state**
5. **Think in terms of ownership**

---

This document consolidates all concepts, mental models, edge cases, and real-world backend scenarios related to JavaScript Sets, Maps, WeakSets, WeakMaps, and garbage collection.
It is designed to be a future recall document you can skim and immediately reconstruct the full picture.
