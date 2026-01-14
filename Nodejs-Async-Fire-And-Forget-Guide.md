# Node.js: Await vs Fire-and-Forget Patterns

A guide to understanding when to await async operations and when to let them run in the background safely.

---

## 1. Using `await` (Strict / Slower API)

```javascript
app.post("/reset-password", async (req, res) => {
  await sendEmail(req.body.email);
  res.json({ success: true });
});
```

### What happens

* Request waits for email to complete
* Response sent after completion

### Pros / Cons

| Pros | Cons |
|------|------|
| Strong guarantee | Slower API |
| Simple error handling | Poor UX for user-facing endpoints |

### Use when

* Payments
* DB transactions
* Critical business logic

---

## 2. Fire-and-Forget (Unsafe)

```javascript
app.post("/reset-password", async (req, res) => {
  sendEmail(req.body.email);
  res.json({ success: true });
});
```

### What happens

* API responds immediately
* Email runs in background

### Problem

* If promise rejects -> unhandled rejection
* Can crash Node.js
* No error visibility

**Not safe for production**

---

## 3. Fire-and-Forget (Correct Pattern)

```javascript
app.post("/reset-password", async (req, res) => {
  sendEmail(req.body.email).catch(err => {
    console.error("Email failed:", err);
  });

  res.json({ success: true });
});
```

### Why this works

* API returns immediately
* Async work continues
* Errors are handled safely

**Key insight:** `.catch()` registers an error handler - it does NOT wait.

---

## 4. Timeline Comparison

### With await

```
request → wait → response
```

### Fire-and-forget

```
request → response
             └─ async work continues
```

---

## 5. Important Node.js Behavior

* Async work is not tied to HTTP requests
* Sending a response does not stop async execution
* Tasks run as long as the Node process is alive

### Work can be interrupted by:

* Process crash
* Restart
* Serverless shutdown

---

## 6. Best-Practice Helper

```typescript
function fireAndForget(promise: Promise<any>) {
  promise.catch(console.error);
}
```

Usage:

```typescript
fireAndForget(sendEmail(email));
```

---

## 7. When to Use What

| Use `await` | Use fire-and-forget |
|-------------|---------------------|
| When correctness depends on the result | Emails |
| When failure must stop the request | Logs |
| | Analytics |
| | Notifications |
| | Side effects |

---

## 8. Mental Model

| Concept | Meaning |
|---------|---------|
| `await` | Pause this function |
| No `await` | Run async in background |
| `.catch()` | Safety, not waiting |

**Never fire-and-forget without handling errors.**

---

This document serves as a reference for handling async operations in Node.js APIs, balancing responsiveness with reliability.
