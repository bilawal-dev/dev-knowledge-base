# Service Pattern, Repository Pattern & Dependency Injection (Node.js / Express)

A comprehensive guide to designing clean, scalable, testable backend architectures in Node.js / Express applications.

---

## 1. Core Idea (Big Picture)

Modern backend architecture separates concerns into clear layers:

```
Request → Controller → Service → Repository → Database
```

Each layer answers a different question:

| Layer | Question it answers |
|-------|---------------------|
| Controller | How do we handle HTTP? |
| Service | What should the system do? (business logic) |
| Repository | How do we store/fetch data? |
| Database | Where is data physically stored? |

---

## 2. Repository Pattern — Data Access Layer

### Purpose

The Repository pattern abstracts all database logic into dedicated classes or modules.

It answers:
> "How do I talk to the database?"

### Responsibilities

- Writing DB queries (Prisma, Mongoose, SQL, etc.)
- Fetching data
- Storing data
- Updating/deleting data
- Handling transactions (sometimes)

### Example

```typescript
// user.repository.ts
export class UserRepository {
  async findById(id: string) {
    return prisma.user.findUnique({ where: { id } });
  }

  async findByEmail(email: string) {
    return prisma.user.findUnique({ where: { email } });
  }

  async create(data: CreateUserDto) {
    return prisma.user.create({ data });
  }
}
```

### Repository should NOT

- Contain business rules
- Validate workflows
- Know about HTTP (req/res)
- Hash passwords
- Decide permissions

---

## 3. Service Pattern — Business Logic Layer

### Purpose

The Service layer contains business rules and use-cases.

It answers:
> "What should happen in the system?"

### Responsibilities

- Validation rules
- Workflows (register user, place order, book meeting)
- Combining multiple repositories
- Calling external services (email, payments, queues)
- Enforcing domain logic

### Example

```typescript
// user.service.ts
export class UserService {
  constructor(private userRepo: UserRepository) {}

  async register(data: RegisterUserDto) {
    const existing = await this.userRepo.findByEmail(data.email);
    if (existing) throw new Error("Email already in use");

    const hashed = await hashPassword(data.password);

    return this.userRepo.create({
      ...data,
      password: hashed,
    });
  }
}
```

### Service should NOT

- Contain raw Prisma/SQL queries
- Know about Express request/response
- Care how data is physically stored

---

## 4. Controller — HTTP Layer (For Context)

Controllers are thin by design.

They only:
- Read input (req.body, req.params)
- Call services
- Return responses

```typescript
// user.controller.ts
const userService = new UserService(new UserRepository());

export const register = async (req, res) => {
  const user = await userService.register(req.body);
  res.status(201).json(user);
};
```

> Controllers should not contain business logic.

---

## 5. Dependency Injection (DI) — The Glue

### The famous line

```typescript
constructor(private userRepo: UserRepository) {}
```

This is TypeScript parameter property + dependency injection.

It is equivalent to:

```typescript
class UserService {
  private userRepo: UserRepository;

  constructor(userRepo: UserRepository) {
    this.userRepo = userRepo;
  }
}
```

### What actually happens

```typescript
const repo = new UserRepository();
const service = new UserService(repo);
```

At runtime:

```typescript
this.userRepo = repo
```

Now the service can use:

```typescript
this.userRepo.findById(...)
```

---

## 6. Why Dependency Injection Matters

Instead of:

```typescript
class UserService {
  private userRepo = new UserRepository(); // ❌ tightly coupled
}
```

We do:

```typescript
class UserService {
  constructor(private userRepo: UserRepository) {}
}
```

This gives:

- ✅ Loose coupling
- ✅ Easy testing
- ✅ Replaceable implementations
- ✅ Clear dependencies
- ✅ Better architecture

---

## 7. Testing Power (Real Reason DI Exists)

Because of DI, you can inject a fake repository:

```typescript
const fakeRepo = {
  findByEmail: async () => null,
  create: async (data) => ({ id: "1", ...data })
};

const service = new UserService(fakeRepo as any);
```

Now:
- No real DB
- No Prisma
- Only business logic tested

> This is impossible if the service creates its own repository.

---

## 8. Mental Model

Think of it like this:

| Component | Analogy |
|-----------|---------|
| Repository | "database adapter" |
| Service | "brain" |
| Controller | "HTTP remote control" |
| Dependency Injection | "plugging parts together" |

```
🧩 Controller plugs Service
🧩 Service plugs Repository
🧩 Repository plugs Database
```

---

## 9. Common Confusions

### ❌ "Service vs Repository — which one should I use?"

Wrong framing.

You use:
- **Repository** → for data
- **Service** → for logic

They solve different problems.

### ❌ "Service can directly use Prisma, so repo is useless"

Okay for very small apps.

But repositories become critical when:
- App grows
- Logic becomes complex
- You want real unit tests
- You may change DB or split microservices

---

## 10. Real-World Folder Structure

**Module-based:**

```
src/
  modules/
    user/
      user.controller.ts
      user.service.ts
      user.repository.ts
      user.routes.ts
      user.schema.ts      // Zod schemas (replaces dto)
```

**Or globally:**

```
src/
  controllers/
  services/
  repositories/
  routes/
  schemas/
  lib/
```

---

## 11. DTO vs Zod — Validation & Types

### What a DTO really is

**DTO = Data Transfer Object**

Traditionally, a DTO is just a TypeScript type/class that describes the shape of data moving between layers.

```typescript
export interface RegisterUserDto {
  email: string;
  password: string;
  name: string;
}
```

DTOs by themselves:
- ✅ Give type safety
- ❌ Do NOT validate at runtime
- ❌ Can accept invalid input from the real world

> **Important:** TypeScript types disappear at runtime. So `req.body` can still be garbage.

### What Zod changes

Zod gives you **runtime validation + static types** from the same source.

```typescript
import { z } from "zod";

export const RegisterUserSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
  name: z.string().min(1),
});

export type RegisterUserDto = z.infer<typeof RegisterUserSchema>;
```

Now you get:
- ✅ Runtime validation
- ✅ Type inference
- ✅ Auto-synced types
- ✅ Safer service layer

> Zod schema becomes the real DTO.

### Where Zod fits in the architecture

Best-practice placement:

```
Request → Controller → (Zod validation) → Service → Repository → DB
```

**Controller validates, Service trusts.**

```typescript
// user.controller.ts
export const register = async (req, res) => {
  const data = RegisterUserSchema.parse(req.body); // runtime validation
  const user = await userService.register(data);
  res.status(201).json(user);
};

// user.service.ts
async register(data: RegisterUserDto) {
  // here data is guaranteed valid
}
```

> Service never touches raw `req.body`.

### Zod vs classic DTOs

| Feature | DTO only | Zod |
|---------|----------|-----|
| Type safety | ✅ | ✅ |
| Runtime validation | ❌ | ✅ |
| Single source of truth | ❌ | ✅ |
| Request validation | ❌ | ✅ |
| Works with forms / APIs | 😐 | 🔥 |
| Prevents invalid service calls | ❌ | ✅ |

> **Conclusion:** Zod schemas are "living DTOs."

### Can Zod fully replace DTOs?

In most Express apps: **Yes.**

**Pattern:**
- Zod schema = validation + DTO definition
- `z.infer<>` = DTO type

You no longer need separate DTO files unless:
- You want multiple shapes (CreateUser, UpdateUser, PublicUser)
- You separate API DTOs from internal domain models

Even then, Zod can still power all of them.

### Advanced real-world pattern

```typescript
// user.schema.ts
export const CreateUserSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});

export const UpdateUserSchema = CreateUserSchema.partial();

export type CreateUserDto = z.infer<typeof CreateUserSchema>;
export type UpdateUserDto = z.infer<typeof UpdateUserSchema>;
```

### Zod does NOT replace services or repositories

Zod handles:
- Shape
- Validation
- Parsing
- Transformation

It does **NOT** handle:
- Business rules
- Permissions
- Database logic
- Architecture

So the layering still stays:

```
Controller (Zod) → Service → Repository
```

### Common mistakes with validation

- ❌ Validating inside repositories
- ❌ Passing raw `req.body` into services
- ❌ Using TS interfaces only for external input
- ❌ Putting business rules inside Zod refinements
- ❌ Letting controllers talk to Prisma directly

### When classic DTOs still make sense

- NestJS class-validator ecosystem
- Internal-only domain models
- Output shaping (PublicUser, SafeUser)
- Cross-service contracts (OpenAPI-first)

Even then, many teams still use Zod.

---

## 12. Relationship to Clean Architecture / DDD

This pattern is a simplified form of:
- Clean Architecture
- Hexagonal Architecture
- Domain-Driven Design

**Mapping:**

| Clean Architecture | Express app |
|--------------------|-------------|
| Controllers | Controllers |
| Use cases | Services |
| Gateways | Repositories |
| Entities | Domain models |
| DB | Prisma / Mongo / SQL |

---

## 13. Quick Decision Guide

| Concern | Goes where? |
|---------|-------------|
| HTTP, status codes | Controller |
| Validation (Zod) | Controller |
| Workflows, business rules | Service |
| Hashing, permissions | Service |
| Prisma/Mongoose/SQL | Repository |
| Joins, DB queries | Repository |
| Calling Stripe/Email | Service |
| Combining multiple repos | Service |

---

## 14. Key Mental Rules

1. Controllers are thin.
2. Services own business logic.
3. Repositories own database access.
4. Services depend on repositories.
5. Controllers depend on services.
6. Dependencies are injected, not created.
7. Business logic must be testable without a database.
8. Validate at controller boundary with Zod.
9. Services trust their input is already validated.

---

## 15. Quiz Concepts (Self-check)

- [ ] Repository contains only DB logic.
- [ ] Service contains business rules.
- [ ] Controller should not call Prisma directly.
- [ ] `constructor(private repo: Repo)` is dependency injection.
- [ ] DI allows mocking repositories.
- [ ] Service must not depend on Express.
- [ ] Repository must not depend on business rules.
- [ ] Thin controllers, fat services.
- [ ] You can swap Mongo → Postgres by replacing repositories.
- [ ] Architecture is about boundaries, not folders.
- [ ] Zod schemas can replace traditional DTOs.
- [ ] Validation happens at controller, not service.

---

## 16. Takeaways

| Concept | Purpose |
|---------|---------|
| Repository Pattern | Data abstraction |
| Service Pattern | Business abstraction |
| Dependency Injection | Loose coupling |
| Zod | Runtime validation + type inference |
| DTO | Shape definition (Zod replaces this) |

This structure scales from monolith → SaaS → microservices.

This is the backbone of NestJS, Clean Architecture, and serious backend systems.

---

## 17. Recommended Practice

1. Start small, but design boundaries early.
2. Always keep Prisma/Mongoose out of controllers.
3. Let services express business language.
4. Inject dependencies manually in Express.
5. Consider DI containers (NestJS, tsyringe, Inversify) for large apps.
6. Write unit tests at the service level.
7. Use Zod schemas as DTOs in modern Express apps.
8. Validate at controller boundary, trust in services.

---

## 18. One-Line Summary

Controllers handle HTTP and validation, services handle thinking, repositories handle data, and dependency injection connects everything.