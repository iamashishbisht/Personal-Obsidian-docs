Rate limiting helps protects services from being overwhelmed by too many requests from a single user or client.

> [!Question] 1. Explain token bucket and leaky bucket in detail

# 1. First: Why do we need Token Bucket / Leaky Bucket?

Suppose your API can safely handle **10 requests/second**.

A client suddenly sends:

```
100 requests
   ↓
Your API
```

If you allow all 100 immediately, your service may get overloaded.

So we need **rate limiting / traffic shaping**.

Two classic algorithms are:

- **Token Bucket** → controls how many requests can pass, while allowing bursts.
- **Leaky Bucket** → controls the rate at which requests leave, producing a smoother flow.

The easiest mental model:

> 🪣 **Token Bucket:** "Do I have a token to spend?"
> 
> 🚰 **Leaky Bucket:** "The pipe can only let requests out at this rate."

---

# 2. Token Bucket

Imagine a bucket containing tokens.

```
             Tokens added
                  ↓
            +-----------+
            | 🟡 🟡 🟡  |
            | 🟡 🟡     |   ← Token Bucket
            +-----------+
                  |
                  | Request needs 1 token
                  ↓
               API Server
```

There are usually two important parameters:

```
Bucket capacity = 10 tokens
Refill rate     = 2 tokens/second
```

Meaning:

- Bucket can hold at most **10 tokens**
- Every second, **2 new tokens** are generated
- Each request consumes **1 token**

---

# 3. Step-by-step example

Suppose initially:

```
Bucket = 10 tokens
```

Client sends request #1.

```
Request
   ↓
Take 1 token
   ↓
Bucket = 9
   ↓
Request allowed
```

Request #2:

```
Bucket = 8
Request allowed
```

Eventually:

```
Bucket = 0
```

Now another request arrives.

```
Request
   ↓
Any token?
   ↓
NO
```

What happens depends on the implementation:

```
Reject immediately
        OR
Wait until a token becomes available
```

For a typical API rate limiter, it is often rejected with:

```
HTTP 429 Too Many Requests
```

---

# 4. Why is it called "Token Bucket"?

Because requests don't enter the bucket.

**Tokens are in the bucket.**

That's a very important distinction.

Wrong mental model:

```
Request → bucket
```

Correct:

```
             tokens
               ↓
        +--------------+
        | 🟡 🟡 🟡 🟡  |
        | 🟡 🟡        |
        +--------------+
               ↑
           request
               |
        "Can I take a token?"
```

A request is basically asking:

> "Can I spend one token?"

If yes → request passes.

---

# 5. The really important part: Burst traffic

This is where Token Bucket becomes interesting.

Suppose:

```
Capacity = 10
Refill = 2 tokens/sec
```

Imagine nobody has made requests for 5 seconds.

Tokens refill until:

```
10 tokens
```

Now suddenly:

```
Client
 ↓
10 requests instantly
```

All 10 can pass!

Why?

Because we accumulated 10 tokens.

```
Before:

🟡 🟡 🟡 🟡 🟡 🟡 🟡 🟡 🟡 🟡
10 tokens

10 requests arrive:

R1 → 🟡
R2 → 🟡
R3 → 🟡
...
R10 → 🟡

Bucket = 0
```

So Token Bucket allows a **burst**.

This is one of its biggest characteristics.

---

# 6. But doesn't that violate "2 requests/second"?

No.

This is a subtle but important point.

If you configure:

```
Capacity = 10
Refill = 2/sec
```

you're not saying:

> "Exactly 2 requests can happen every second."

You're saying roughly:

> "Requests consume tokens at whatever speed they arrive, but tokens are replenished at 2/sec and the bucket can accumulate up to 10."

Therefore:

```
Idle for a while
     ↓
Bucket fills
     ↓
Burst of requests allowed
     ↓
Bucket becomes empty
     ↓
Only ~2 tokens/sec become available
```

---

# 7. Token Bucket timeline

Let's make this concrete.

Configuration:

```
Capacity = 5
Refill = 1 token/sec
```

Initially:

```
Time 0

🟡 🟡 🟡 🟡 🟡
5 tokens
```

Client sends 3 requests immediately:

```
R1 → token
R2 → token
R3 → token

Remaining:

🟡 🟡
2 tokens
```

At 1 second:

```
+1 token

🟡 🟡 🟡
3 tokens
```

At 2 seconds:

```
🟡 🟡 🟡 🟡
4 tokens
```

At 3 seconds:

```
🟡 🟡 🟡 🟡 🟡
5 tokens
```

But bucket cannot become:

```
🟡 🟡 🟡 🟡 🟡 🟡
```

because capacity is 5.

Extra tokens are simply not accumulated.

---

# 8. Token Bucket formula

You can think of it as:

```
new_tokens =
    old_tokens
    + elapsed_time × refill_rate
```

but capped at:

```
bucket_capacity
```

For example:

```
old tokens = 3
elapsed = 2 seconds
rate = 2 tokens/sec
```

Then:

```
3 + (2 × 2)
= 7
```

If capacity is 5:

```
min(7, 5)
= 5
```

So:

```
Bucket = 5
```

---

# 9. Now Leaky Bucket

Now forget tokens.

Imagine a bucket with a hole at the bottom.

```
          Requests
             ↓
        R R R R R
       +---------+
       | R R R R |
       | R R R   |
       |         |
       +----↓----+
            |
            |
            ↓
         API Server
```

The bucket "leaks" at a fixed rate.

For example:

```
Leak rate = 2 requests/sec
```

So regardless of how quickly requests arrive:

```
R R R R R R R R R
        ↓
     bucket
        ↓
     R
        ↓
     R
        ↓
     R
```

The output is approximately:

```
R → 0.5 sec → R → 0.5 sec → R → ...
```

The important idea:

> **Leaky Bucket smooths the outgoing traffic.**

---

# 10. Example: 10 requests arrive instantly

Suppose:

```
Leaky Bucket
Rate = 2 requests/sec
```

At time 0:

```
R1 R2 R3 R4 R5 R6 R7 R8 R9 R10
                ↓
             bucket
```

They don't all hit your API simultaneously.

Instead:

```
Time       Request leaving

0 sec      R1
0.5 sec    R2
1 sec      R3
1.5 sec    R4
2 sec      R5
2.5 sec    R6
...
```

So the API receives:

```
R → R → R → R → R
```

rather than:

```
RRRRRRRRRR
```

This is **traffic smoothing**.

---

# 11. The bucket can fill up

Suppose the bucket capacity is:

```
5 requests
```

and the server is leaking at:

```
1 request/sec
```

Suddenly 10 requests arrive:

```
R1 R2 R3 R4 R5 R6 R7 R8 R9 R10
```

Bucket can only hold:

```
R1
R2
R3
R4
R5
```

The remaining requests:

```
R6 R7 R8 R9 R10
```

must either be:

```
rejected
```

or handled according to the implementation.

So Leaky Bucket can also involve a queue.

---

> [!Question] ok so token bucket or leaky bucket has own bucket per user or globally for all users a single bucket? if its global chances are that some user will not get chance to hit the server and get response?

Yes — **it can be either**, and this is an important design decision.

Yes — **it can be either**, and this is an important design decision.

### 1. Usually, for per-user/API-client limiting → one bucket per user

Suppose you have:

```
Rate limit = 10 requests/sec per user
```

Then you conceptually have:

```
User A → 🪣 Bucket A → API
User B → 🪣 Bucket B → API
User C → 🪣 Bucket C → API
```

Each bucket is independent.

So if User A sends 100 requests:

```
User A → 🪣 EMPTY → requests rejected
```

User B can still send requests normally:

```
User B → 🪣 tokens available → API → response
```

This prevents one user from consuming everyone else's quota.

---

### 2. But you can also have a global bucket

Suppose you say:

```
Entire API
↓
1000 requests/sec maximum
```

Then you could have:

```
All users
    ↓
+----------------+
| Global Bucket  |
| 🟡🟡🟡🟡🟡     |
+----------------+
        ↓
      API
```

Now **yes, your concern is correct**.

Imagine:

```
Global limit = 10 requests/sec

User A → 8 requests
User B → 2 requests
User C → 1 request
```

The first 10 consume the available capacity.

User C's request might be rejected/throttled even though **User C itself wasn't abusing anything**.

So a pure global bucket can create **fairness problems**.

---

# 3. Real systems often use BOTH

This is the important part.

You don't necessarily choose:

> "Either per-user OR global."

You can have multiple layers.

For example:

```
                    API Gateway

                 ┌───────────────┐
                 │ Global Bucket │
                 │ 10,000 req/s  │
                 └───────┬───────┘
                         ↓
              ┌────────────────────┐
              │ Per-user buckets   │
              │                    │
              │ User A → 🪣 10/s   │
              │ User B → 🪣 10/s   │
              │ User C → 🪣 10/s   │
              └─────────┬──────────┘
                        ↓
                     Backend
```

So:

**Global bucket** protects your entire infrastructure.

**Per-user bucket** prevents one user from monopolizing it.

---

# 4. Example

Suppose your backend can safely handle:

```
10,000 requests/sec
```

You configure:

```
Global:
10,000 req/sec

Per user:
100 req/sec
```

Now:

### User A sends 5,000 req/sec

Per-user bucket says:

```
❌ User A exceeded 100/sec
```

So most of A's requests are rejected.

But:

```
User B → 50/sec → allowed
User C → 80/sec → allowed
User D → 20/sec → allowed
```

They aren't affected by A's abuse.

# 5. But there's another problem

Suppose:

```
Global = 100 req/sec
Per-user = 100 req/sec
```

And you have:

```
User A → 100 req/sec
User B → 100 req/sec
User C → 100 req/sec
```

All three individually satisfy their limit.

But together:

```
100 + 100 + 100
       =
300 req/sec
```

Your server only handles:

```
100 req/sec
```

So the **global limit is still necessary**.

That's why layered rate limiting is common.

---

# 6. One more subtle point: "per user" isn't necessarily literally a logged-in user

The bucket key could be based on:

```
userId
```

or:

```
API key
```

or:

```
client/application ID
```

or sometimes:

```
IP address
```

For example:

```
bucket:user:12345
bucket:user:67890
```

or:

```
bucket:api-key:abc123
bucket:api-key:xyz789
```

The key determines **who gets their own bucket**.

# 7. And in a distributed system...

This becomes even more interesting.

Suppose you have:

```
             Load Balancer
             /     |     \
            ↓      ↓      ↓
         Gateway1 Gateway2 Gateway3
```

If you store the bucket only in each server's memory:

```
Gateway1 → User A bucket
Gateway2 → User A bucket
Gateway3 → User A bucket
```

you don't really have **one bucket per user** anymore.

User A could potentially get:

```
10 requests → Gateway1
10 requests → Gateway2
10 requests → Gateway3
```

even though you intended:

```
10 requests total
```

So distributed rate limiting often needs a **shared state store** such as Redis, or another coordinated mechanism.

That's where the topic becomes much more interesting.

---

### The mental model I'd recommend

Think of rate limiting as having **two independent questions**:

```
1. WHO are we limiting?
        ↓
   user / API key / IP / global

2. WHAT are we protecting?
        ↓
   individual user / entire backend
```

And you can have:

```
Per-user bucket
       +
Global bucket
       ↓
Strong protection + reasonable fairness
```

So yes, **your concern about a single global bucket is valid**. A global-only bucket can cause one user's traffic to consume capacity that another user could otherwise have used.

> [!example] Code example : Token Bucket

```
package implementations.java.rate_limiting;

import java.time.Instant;

public class TokenBucket {
    private final long capacity;        // Maximum number of tokens the bucket can hold
    private final double fillRate;      // Rate at which tokens are added to the bucket (tokens per second)
    private double tokens;              // Current number of tokens in the bucket
    private Instant lastRefillTimestamp; // Last time we refilled the bucket

    public TokenBucket(long capacity, double fillRate) {
        this.capacity = capacity;
        this.fillRate = fillRate;
        this.tokens = capacity;  // Start with a full bucket
        this.lastRefillTimestamp = Instant.now();
    }

    public synchronized boolean allowRequest(int tokens) {
        refill();  // First, add any new tokens based on elapsed time

        if (this.tokens < tokens) {
            return false;  // Not enough tokens, deny the request
        }

        this.tokens -= tokens;  // Consume the tokens
        return true;  // Allow the request
    }

    private void refill() {
        Instant now = Instant.now();
        // Calculate how many tokens to add based on the time elapsed
        double tokensToAdd = (now.toEpochMilli() - lastRefillTimestamp.toEpochMilli()) * fillRate / 1000.0;
        this.tokens = Math.min(capacity, this.tokens + tokensToAdd);  // Add tokens, but don't exceed capacity
        this.lastRefillTimestamp = now;
    }
}
```

> [!example] Code example : Leaky Bucket

https://github.com/ashishps1/awesome-system-design-resources/blob/main/implementations/java/rate_limiting/LeakyBucket.java

