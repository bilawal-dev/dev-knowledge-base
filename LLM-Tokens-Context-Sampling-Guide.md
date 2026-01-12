# Complete Guide: LLM Tokens, Context Windows, and Sampling Parameters

This document consolidates learnings about how LLMs process text (tokens), manage memory (context windows), control output randomness (temperature, top-p), and practical strategies for cost optimization.

---

## 1. What is a Token?

A token is **not** a word. It's a chunk of text that the model actually reads and processes.

### Rough Intuition

* 1 English word ≈ 0.75 tokens
* 1 character ≈ 0.25 tokens
* Code is more token-dense than English (symbols, casing, punctuation add up)

### Examples

| Text | Approximate Tokens |
|------|-------------------|
| `"Hello world"` | ~2 tokens |
| `const userId = req.params.userId;` | ~9–12 tokens |

### Why Tokens Matter

LLMs:
* **Read** tokens
* **Think** in tokens
* **Charge** you per token
* **Forget** things beyond token limits

---

## 2. Input Tokens vs Output Tokens

Every API request has two sides:

### Input Tokens

* System prompt
* Developer instructions
* User message
* Retrieved docs (RAG)
* Chat history

### Output Tokens

* The model's response

**Important:** You pay for both.

---

## 3. Cost Model (How Billing Works)

```
Cost = (input tokens × input price) + (output tokens × output price)
```

Even if the response is tiny, long prompts are expensive.

### Real-Life Example

Assume:
* Input: 6,000 tokens
* Output: 300 tokens

If input pricing is high (which it usually is):
* **Your prompt costs more than the answer**

---

## 4. Context Window (The Model's Memory)

### Definition

The context window is the maximum number of tokens the model can "see" at once.

**Think of it like RAM, not disk.**

If you exceed it:
* Old tokens are dropped
* Or the request fails
* Or reasoning quality collapses

### Context Window Example

Model context window = 128k tokens

Your prompt breakdown:

| Component | Tokens |
|-----------|--------|
| System prompt | 2k |
| Chat history | 50k |
| RAG docs | 60k |
| User message | 1k |
| **Total** | **113k** |

You only have **15k left for output**.

If the model needs more → truncated response.

---

## 5. Context Window vs Memory vs State

| Concept | What it is | Persists? |
|---------|-----------|-----------|
| Context window | Tokens model sees | No |
| Chat history | You resend manually | No |
| Vector DB (RAG) | External memory | Yes |
| DB / Redis | Real state | Yes |

**LLMs do not remember unless you resend or retrieve.**

---

## 6. Temperature (The "Creativity Dial")

### Definition

Controls randomness in the model's output.

* **Lower** → deterministic, repetitive, predictable
* **Higher** → creative, varied, sometimes "weird"

### Range: 0 → 2 (practically 0–1.5)

| Temp | Behavior | Example (writing JS function) |
|------|----------|------------------------------|
| 0.0 | Always picks most likely token | `function sum(a,b){return a+b}` |
| 0.5 | Slight creativity, safe | `function sum(a, b){return a + b;}` |
| 1.0 | Creative, may vary | `const sum = (a, b) => a + b; // easy addition` |
| 1.5 | Very random, might hallucinate | `const addNumbers = (x, y) => x + y; // watch out` |

### Analogy: The Chef

Temperature = how wild your chef is willing to experiment.

* **0** → follows recipe exactly
* **1** → adds own flair
* **2** → may throw pineapple on steak

---

## 7. Top-p (Nucleus Sampling)

### Definition

Controls the cumulative probability of tokens considered at each step.

Instead of picking any token, only pick from top tokens that make up p% of probability mass.

### Range: 0 → 1

| top_p | Behavior | Notes |
|-------|----------|-------|
| 1.0 | No restriction | Same as not using it |
| 0.9 | Pick tokens summing ≤ 90% probability | Cuts low-probability "weird" tokens |
| 0.5 | Very strict | Very safe, repetitive |

### Analogy: The Ingredient List

Chef has 100 ingredients:

* **Top-p = 0.9** → only use ingredients covering 90% "popularity"
* **Top-p = 0.5** → only safe, common ingredients

---

## 8. Temperature & Top-p Combined Analogy

Think of temp & top-p as how much freedom you give a junior dev:

| Setting | Meaning |
|---------|---------|
| Temp 0 | Strictly follow style guide, no deviation |
| Temp 1 | Can improvise naming, comments |
| Top-p 0.5 | Only picks from approved libraries & patterns |
| Top-p 1 | Can grab anything from npm |

---

## 9. Why Devs Accidentally Burn Money

### Common Mistake #1: Sending Entire Chat History

```javascript
// Bad
messages: fullConversationSinceDayOne
```

**Instead:**
* Summarize
* Truncate
* Keep only what matters

### Common Mistake #2: Stuffing Huge RAG Chunks

```
// Bad
"Here is the entire database dump, answer my question"
```

**Instead:**
* Retrieve only relevant chunks
* 1–3k tokens is usually enough

### Common Mistake #3: Repeating Instructions Every Request

```
// Bad - repeated 500 times/day = real money
System prompt: "You are an expert senior software engineer..."
```

---

## 10. max_output_tokens (Hard Safety Lever)

```javascript
max_output_tokens: 400
```

Prevents:
* Runaway costs
* Infinite rambles
* Accidental essays

**Warning:** Without this, models can output thousands of tokens.

---

## 11. Code Examples

### Naive Implementation (Expensive)

```javascript
const response = await client.responses.create({
  model: "gpt-4.1",
  input: [
    {
      role: "system",
      content: systemPrompt
    },
    ...fullChatHistory,   // 🔥 huge
    {
      role: "user",
      content: userMessage
    }
  ]
});
```

### Better Implementation (Production-Grade)

```javascript
const response = await client.responses.create({
  model: "gpt-4.1",
  input: [
    {
      role: "system",
      content: systemPrompt
    },
    {
      role: "user",
      content: summarizedContext // 500 tokens max
    },
    {
      role: "user",
      content: userMessage
    }
  ],
  max_output_tokens: 400
});
```

---

## 12. Context Compression Techniques

### 1. Summarization

Store a summary instead of full history:

```
User is building a SaaS booking system using Prisma + Postgres.
Current issue: overlapping booking validation.
```

Instead of 30 messages.

### 2. Sliding Window

* Keep last 5 messages
* Drop the rest

### 3. RAG with Relevance

* Retrieve top 3 chunks
* Not entire docs

---

## 13. Mental Model

```
Tokens are money
Context is RAM
LLMs are stateless CPUs
```

If you design like a backend engineer:
* You'll save cost
* You'll improve accuracy
* You'll scale safely

---

## 14. Practical Rules of Thumb

1. Keep prompts < 2k tokens when possible
2. Never send raw DB dumps
3. Always cap output tokens
4. Prefer summaries over history
5. Measure tokens in dev (log them)

---

## 15. Quick Reference Table

| Concept | What It Controls | Default Behavior |
|---------|-----------------|------------------|
| Token | Unit of text processing | ~0.75 per English word |
| Context Window | Max tokens model can see | Model-dependent (8k–128k+) |
| Temperature | Output randomness | Higher = more creative |
| Top-p | Token probability cutoff | 1.0 = no restriction |
| max_output_tokens | Response length limit | Prevents runaway output |

---

This document serves as a reference for understanding LLM internals from a developer's perspective—focusing on practical cost optimization, context management, and output control.
