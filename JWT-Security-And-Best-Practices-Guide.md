# JWT Security & Best Practices

A practical guide to understanding JWTs — what they protect, what they don't, and how to use them correctly.

---

## 1. The Core Mental Model

JWTs are **readable but tamper-proof**.

Think of a JWT like a **sealed envelope with a wax seal**:
- Anyone can read the envelope (payload)
- But if they change anything, the seal (signature) breaks

> **JWT protects authenticity and integrity, not secrecy.**

---

## 2. JWT Structure

| Part | Purpose | Readable by client? |
|------|---------|---------------------|
| Header | Algorithm info (`HS256`, `RS256`) | ✅ Yes |
| Payload | Claims (`userId`, `role`, `exp`) | ✅ Yes |
| Signature | Proves authenticity | ❌ Needs secret |

The payload is just base64url-encoded — anyone can decode it:

```javascript
const payload = JSON.parse(atob(jwt.split('.')[1]));
```

---

## 3. What `jwt.verify` Actually Does

```javascript
const payload = jwt.verify(token, process.env.JWT_SECRET);
```

This:
1. Recalculates the signature using the secret + header + payload
2. Compares it to the signature in the token
3. Returns success **only if they match** and token is not expired

If someone changes `role: "ADMIN"` in the payload → signature breaks → `jwt.verify` fails → backend rejects it.

> **The secret is only needed to sign and verify, not to read.**

---

## 4. The Golden Rule

> **Never store sensitive data in a JWT.**

- ❌ Passwords, API keys, secrets
- ✅ `userId`, `email`, `role`, `exp` — non-confidential claims

If you need confidentiality, use **JWE** (encrypted JWT) or store data server-side.

---

## 5. Why JWT Is Still Useful

Even though the payload is readable, a client **cannot forge it** without the secret.

**Use cases:**
- Stateless auth (no server-side session store)
- Cross-service verification (microservices)
- Mobile apps and cross-domain APIs
- Serverless / edge environments

---

## 6. Common Security Mistakes

| Mistake | Why it's bad |
|---------|--------------|
| Storing JWT in `localStorage` | XSS can steal it |
| Weak secret (`"123456"`) | Token can be brute-forced |
| No `exp` claim | Token lives forever |
| No refresh token strategy | Stolen token = full access |
| Putting secrets in payload | Payload is readable by anyone |
| Not verifying `aud`, `iss` claims | Tokens can be reused across services |
| Accepting `alg: none` | Signature check is bypassed |

---

## 7. Secure JWT Setup

```
✅ httpOnly cookie (not localStorage)
✅ Secure + SameSite=Strict flags
✅ Strong 256-bit signing secret (or RS256 key pair)
✅ Short-lived access token (5–15 min)
✅ Rotating refresh tokens (stored in DB/Redis)
✅ Backend validates every protected request
```

---

## 8. httpOnly Cookies & JWT

- `httpOnly` cookies are **inaccessible to JavaScript** — only set via server response headers
- A malicious script **cannot steal** what it cannot access
- Frontend checks (does cookie exist?) are just **UX helpers**
- **All security must be enforced on the backend**

> Rule of thumb: always assume the client is malicious.

---

## 9. JWT vs Sessions

| Feature | JWT | Server Sessions |
|---------|-----|-----------------|
| Stateless | ✅ | ❌ |
| Horizontal scaling | ✅ Easy | Needs Redis |
| Immediate revocation | ❌ Needs extra logic | ✅ |
| Payload readable | ✅ | ❌ |

JWT is not "more secure" than sessions — it's a **different architecture**.

For monolith + web app → traditional sessions with Redis are often simpler.

---

## 10. One-Line Summary

JWTs are readable tokens whose purpose is **verification via signature** — never store sensitive data in them, always verify on the backend, and store them in httpOnly cookies.
