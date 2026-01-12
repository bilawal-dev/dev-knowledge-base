# JavaScript / TypeScript Class Export Patterns

A guide to singleton vs static class patterns and when to use each approach.

---

## 1. `export default new Class()` - Singleton Object

```typescript
class EmailService {
  private cache = new Map<string, any>();

  async send() {}
}

export default new EmailService();
```

### What this is

* A real object is created
* Created once when the file is loaded
* Same instance shared everywhere (Node module cache)

### What it supports

| Feature | Supported |
|---------|-----------|
| Instance state (`this.cache`, `this.db`, `this.client`) | Yes |
| Connections, caches, queues | Yes |
| Encapsulation of resources | Yes |

### Mental model

> "This represents a thing/service that lives in memory."

### Typical uses

* EmailService
* BookingService
* DB client
* Redis client
* Queue manager
* Payment service

### Pros / Cons

| Pros | Cons |
|------|------|
| Real singleton | Slightly more boilerplate |
| Clean ownership of state | You must manage lifecycle |
| Easier testing & mocking | |
| Future-proof (DI, multiple instances later) | |

---

## 2. `export default class { static ... }` - Static Class (No Objects)

```typescript
export default class PasswordUtil {
  static async hash(pwd: string) {}
  static async compare(pwd: string, hash: string) {}
}
```

Usage:

```typescript
await PasswordUtil.hash("123");
```

### What this is

* No object created
* Just a class as a namespace
* Methods live on the class itself

### About `this`

`this` refers to the class itself. Can only access static data & static methods:

```typescript
static cache = new Map();
static get() {
  return this.cache;
}
```

### Mental model

> "This represents logic, not a thing."

### Typical uses

* Utils
* Helpers
* Validators
* Formatters
* Mappers
* Pure business rules

### Pros / Cons

| Pros | Cons |
|------|------|
| Simple | No instance |
| No instantiation | Harder to mock |
| Good organization for stateless logic | Static state = global state |
| | Bad for services with resources |

---

## 3. Can Static Classes Have State?

Yes, technically:

```typescript
static cache = new Map();
```

But architecturally:

**Static state = global variable in disguise**

### Use this only for:

* Constants
* Pure caches
* Framework-level helpers

### Avoid for:

* DB clients
* External integrations
* Business services

---

## 4. Instance vs Static - The Real Rule

### Use `new Class()` when:

* The module owns state
* Manages connections/resources
* Represents a service

**Examples:** Email service, Booking service, File storage service, Payment service

### Use static when:

* No per-object data
* Pure functions
* Business logic helpers
* Structural organization

**Examples:** Validators, Crypto helpers, Date utils, Slot generators

---

## 5. Are Both "Singletons"?

| Pattern | Type |
|---------|------|
| `export default new Class()` | True singleton object |
| `export default class { static ... }` | Single shared class, but no instance exists |

Both are "one shared thing", but they are not the same pattern.

---

## 6. EmailService Example (Correct Design)

```typescript
class EmailService {
  private transporterCache = new Map<string, Transporter>();

  private getTransporter(key: string) {
    if (!this.transporterCache.has(key)) {
      this.transporterCache.set(key, createTransporter(key));
    }
    return this.transporterCache.get(key)!;
  }

  async send(dto: SendMailDto) {
    const t = this.getTransporter(dto.provider);
    await t.sendMail(dto.payload);
  }
}

export default new EmailService();
```

### Why instance is better:

* Owns resources
* Owns cache
* Testable
* Replaceable
* Scalable design

---

## 7. Final Mental Model

| Scenario | Pattern |
|----------|---------|
| Represents a thing | Make an object |
| Represents logic | Make it static |
| State + resources | Singleton instance |
| Stateless helpers | Static class |

---

This document serves as a reference for choosing between singleton instances and static classes in JavaScript/TypeScript projects.
