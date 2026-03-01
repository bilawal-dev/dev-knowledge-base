# Node.js TypeScript Workflow — ts-node, Nodemon, and Build Pipeline

A guide to understanding how Node.js, TypeScript, ts-node, and nodemon work together in development and production.

---

## 1. Node.js vs TypeScript vs ts-node

| Tool | Role |
|---|---|
| Node.js | Runs `.js` files natively — cannot run `.ts` directly |
| TypeScript | Superset of JavaScript — adds types, interfaces, enums. Must be compiled to JS |
| ts-node | TypeScript execution engine — transpiles + runs `.ts` in memory. Dev only |

```bash
# Node.js — runs compiled JS
node build/server.js

# ts-node — runs TypeScript directly in dev
ts-node src/server.ts
ts-node --transpile-only src/server.ts
```

> **Key point:** `ts-node` is for development only — not production.

---

## 2. Nodemon

A process watcher that watches files and restarts your process on change.

**Why use it:**

* Auto-restart when you save `.ts`, `.json`, `.env`, etc.
* Forwards signals (`Ctrl+C`, `SIGTERM`) correctly
* Works with any engine: `node`, `ts-node`, `tsx`, etc.

**Important flags:**

| Flag | Purpose |
|---|---|
| `--exec` | Command to run on restart (e.g., `ts-node`) |
| `--watch` | Directories/files to monitor |
| `--ext` | File extensions to watch (`ts,json,ejs`) |
| `--delay` | Delay before restarting (ms) |
| `--signal` | Signal to send to child process on restart |

**Example dev command:**

```bash
nodemon --exec ts-node --transpile-only src/server.ts
```

* `nodemon` → watches files
* `--exec ts-node` → runs TypeScript engine
* `--transpile-only` → skips type checking for speed
* `src/server.ts` → entry point

---

## 3. ts-node --watch vs nodemon

| Feature | `ts-node --watch` | `nodemon --exec ts-node` |
|---|---|---|
| Watches TS files | Yes | Yes |
| Watches other files (`.json`, `.env`) | No (manual) | Yes (configurable) |
| Auto-restart on save | Yes | Yes |
| Configurable signals / delay | No | Yes |
| Team/Docker safe | Less predictable | Industry standard |

**Rule of thumb:** For teams and production-grade dev use nodemon + ts-node. For solo, experimental dev `ts-node --watch` is enough.

---

## 4. Professional TypeScript Workflow

**Step 1 — Development:**

```bash
"dev": "nodemon --exec ts-node --transpile-only src/server.ts"
```

* Auto-reload on changes
* Fast startup, skips type-checking
* Optional: separate `typecheck` script for type validation

**Step 2 — Build / Compile:**

```bash
"build": "tsc"
```

* Runs TypeScript compiler
* Checks all types, fails on errors
* Produces JavaScript in `build/` or `dist/`

**Step 3 — Production:**

```bash
"start": "node build/server.js"
```

* Pure Node execution — no `ts-node`, no `nodemon`
* Fastest runtime
* Guarantees TS compiled successfully

**Step 4 — Type Check (Optional):**

```bash
"typecheck": "tsc --noEmit"
```

* Validates types without emitting JS
* Run in CI or pre-commit hooks

---

## 5. Recommended package.json Scripts

```json
"scripts": {
  "dev": "nodemon --exec ts-node --transpile-only src/server.ts",
  "typecheck": "tsc --noEmit",
  "build": "tsc",
  "start": "node build/server.js",
  "db:migrate": "npx prisma migrate dev",
  "db:seed": "npx ts-node prisma/seed.ts",
  "db:reset": "npx prisma migrate reset && npm run db:seed"
}
```

---

## 6. Docker / CI Workflow

**Dockerfile:**

```dockerfile
FROM node:20
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build
CMD ["node", "build/server.js"]
```

* Only compiled JS runs in the container
* No `ts-node` dependency in production

**CI/CD pipeline:**

```bash
npm ci
npm run typecheck
npm run build
docker build ...
```

---

## 7. Key Takeaways

| Concept | Meaning |
|---|---|
| `nodemon` | Watches and restarts the process |
| `ts-node` | Runs TypeScript in dev |
| `--transpile-only` | Speed over type safety in dev |
| `tsc` | Compiler — authority — produces JS |
| `node build/server.js` | Production run |
| `typecheck` | Optional safety net |

**Never run TypeScript directly in production. Always compile first.**

---

This document serves as a reference for the professional TypeScript + Node.js development and deployment workflow.
