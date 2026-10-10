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


> [!Question] Explain in detail on fixed window, sliding window log and sliding window counter rate limiting algorithm.

Assume:

> **Rate limit = 5 requests per 10 seconds per user**

So for user `Ashish`, the server should allow at most roughly 5 requests in any 10-second period.

The three algorithms differ mainly in **how they define and calculate that 10-second window**:

1. **Fixed Window** → divide time into fixed boxes.
2. **Sliding Window Log** → remember the exact timestamp of every request.
3. **Sliding Window Counter** → remember counts of recent fixed windows and mathematically estimate the current sliding window.

---

# 1. Fixed Window Counter

This is the simplest algorithm.

Imagine time is divided into boxes:

```
10:00:00 ───────── 10:00:10 ───────── 10:00:20
       Window 1             Window 2
       max = 5              max = 5
```

For every user, maintain:

```
user = Ashish

current window = 10:00:00 - 10:00:10
request count = 0
limit = 5
```

Every request increments the counter.

### Example

```
10:00:01  request → count = 1 ✅
10:00:02  request → count = 2 ✅
10:00:03  request → count = 3 ✅
10:00:04  request → count = 4 ✅
10:00:05  request → count = 5 ✅
10:00:06  request → count = 6 ❌
```

At `10:00:10`, the window expires.

Then:

```
10:00:10 - 10:00:20

count = 0
```

And the user gets another 5 requests.

---

## The big problem with Fixed Window

There is a **boundary problem**.

Suppose:

```
Window 1
09:59:50 ───────── 10:00:00

Window 2
10:00:00 ───────── 10:00:10
```

User sends:

```
09:59:59 → 5 requests
```

All 5 are allowed.

Then:

```
10:00:01 → 5 requests
```

Those 5 are also allowed.

So:

```
09:59:59
09:59:59
09:59:59
09:59:59
09:59:59
       ↓
10:00:01
10:00:01
10:00:01
10:00:01
10:00:01
```

That's **10 requests within roughly 2 seconds**, even though the configured limit is:

```
5 requests / 10 seconds
```

This is called the **boundary/burst problem**.

---

# 2. Sliding Window Log

This tries to solve the boundary problem.

Instead of saying:

> "Which fixed window are we in?"

we say:

> **"Look at the last 10 seconds from this exact moment."**

For every user, we store the timestamp of each request.

Example:

```
Ashish:

10:00:01
10:00:03
10:00:04
10:00:07
10:00:09
```

Now another request arrives at:

```
10:00:09
```

We look back 10 seconds:

```
10:00:09 - 10 seconds
       ↓
09:59:59
```

All timestamps between:

```
09:59:59 → 10:00:09
```

are counted.

There are already 5.

Therefore:

```
new request ❌
```

---

## Now look at what happens at 10:00:12

We have:

```
10:00:01
10:00:03
10:00:04
10:00:07
10:00:09
```

Current time:

```
10:00:12
```

Sliding window:

```
10:00:02 ───────── 10:00:12
```

The request at:

```
10:00:01
```

is now outside the window.

So:

```
10:00:03
10:00:04
10:00:07
10:00:09
```

= 4 requests.

Therefore:

```
new request → count becomes 5 ✅
```

---

# Why is it called "Sliding Window"?

Because the window moves continuously with time.

At:

```
10:00:10

window:
10:00:00 ───────── 10:00:10
```

At:

```
10:00:11

window:
10:00:01 ───────── 10:00:11
```

At:

```
10:00:12

window:
10:00:02 ───────── 10:00:12
```

At:

```
10:00:13

window:
10:00:03 ───────── 10:00:13
```

The window isn't tied to `10:00:00`, `10:00:10`, etc.

It moves with every request/current time.

---

# The problem with Sliding Window Log

It is accurate, but it can consume a lot of memory.

Suppose:

```
1 million users
```

and each user makes:

```
100 requests within the window
```

You potentially need to store:

```
1 million × 100
= 100 million timestamps
```

That's a lot of data.

And every request may involve:

1. Adding a timestamp
2. Removing expired timestamps
3. Counting timestamps

So this is accurate but potentially expensive.

---

# 3. Sliding Window Counter

This tries to get the **benefit of sliding window** without storing every individual request timestamp.

Instead of storing:

```
request 1 → 10:00:01
request 2 → 10:00:03
request 3 → 10:00:04
request 4 → 10:00:07
request 5 → 10:00:09
```

we store **counts of fixed windows**.

For example:

```
10-second rate limit
```

We might divide it into smaller fixed windows.

Or, conceptually, maintain:

```
Previous window count
Current window count
```

Suppose:

```
Previous window:
10:00:00 - 10:00:10
requests = 4

Current window:
10:00:10 - 10:00:20
requests = 3
```

Now suppose current time is:

```
10:00:13
```

The current 10-second sliding window is:

```
10:00:03 ───────── 10:00:13
```

Notice something important:

The current sliding window contains:

```
part of previous window
+
part of current window
```

We therefore estimate how much of the previous window should count.

---

# The Sliding Window Counter formula

A commonly used approximation is:

```
estimated count
=
previous window count × previous-window overlap
+
current window count
```

More precisely:

```
estimated count =
previous_count × (1 - elapsed_fraction)
+ current_count
```

Where:

```
elapsed_fraction =
elapsed time in current fixed window
------------------------------------
length of fixed window
```

---

# Let's calculate it

Suppose:

```
Limit = 5 requests / 10 seconds
```

And:

```
Previous window:
10:00:00 - 10:00:10
count = 4

Current window:
10:00:10 - 10:00:20
count = 3
```

Current time:

```
10:00:13
```

So 3 seconds have elapsed in the current window.

Therefore:

```
elapsed fraction = 3 / 10
                 = 0.3
```

The previous window contributes approximately:

```
4 × (1 - 0.3)
= 4 × 0.7
= 2.8
```

Current count:

```
3
```

Therefore:

```
estimated count = 2.8 + 3
                = 5.8
```

Since:

```
5.8 > 5
```

we reject the request.

---

# Why are we multiplying the previous count?

This is the key idea.

Imagine:

```
Previous window
10:00:00                    10:00:10
     |--------------------------|
              4 requests
```

Current time:

```
10:00:13
```

The sliding window is:

```
10:00:03                    10:00:13
     |--------------------------|
```

Only this portion of the previous window matters:

```
10:00:03 ─────── 10:00:10
```

That's roughly:

```
70%
```

of the previous window.

So we estimate:

```
70% of previous requests
```

instead of storing exactly which timestamps those requests had.

---

# Very important: Sliding Window Counter is an approximation

This is something people often miss.

Sliding Window Log:

```
exact
```

because it knows:

```
10:00:01
10:00:03
10:00:04
...
```

Sliding Window Counter:

```
approximation
```

because it only knows:

```
previous window = 4
current window = 3
```

It doesn't know exactly when those 4 previous requests occurred.

Therefore it estimates.

---

# Compare all three

Suppose:

```
Limit = 5 requests / 10 seconds
```

|Algorithm|Stores|Accuracy|Memory|Boundary problem|
|---|---|---|---|---|
|Fixed Window|Counter|Low/Moderate|Very low|Yes|
|Sliding Window Log|Every timestamp|Very high/exact|High|No|
|Sliding Window Counter|Window counts|Approximate|Low|Much better|

---

# Visual mental model

Remember them like this.

### Fixed Window

Think of **calendar boxes**:

```
|---------|---------|---------|
 Window 1  Window 2  Window 3
    5          5          5
```

The box doesn't move.

---

### Sliding Window Log

Think of a **moving ruler** over individual requests:

```
requests:

  •    • •       •     •
  |--------------------|
          last 10 sec
```

Move the ruler:

```
    |--------------------|
```

and count the dots inside it.

Very accurate, but expensive.

---

### Sliding Window Counter

Think of **two buckets of numbers**:

```
Previous window       Current window
       4                    3
       ↓                    ↓
       └────── estimate ────┘
```

Instead of remembering every dot, remember:

```
Previous = 4
Current  = 3
```

Then estimate how much of the previous bucket belongs to the moving window.

---

# Now let's connect this to your previous Token Bucket question

These algorithms are **different concepts** from Token Bucket and Leaky Bucket.

You can think of rate limiting algorithms in two broad styles:

### Counter/window-based

```
Fixed Window
Sliding Window Log
Sliding Window Counter
```

They ask:

> **"How many requests have happened recently?"**

---

### Traffic-shaping/token-based

```
Token Bucket
Leaky Bucket
```

They ask more about:

> **"Can this request pass right now based on available capacity/tokens?"**

Token Bucket, in particular, allows controlled bursts because tokens can accumulate.

---

# One important correction to a common misunderstanding

The algorithm itself doesn't inherently mean:

```
ONE bucket for the entire system
```

or:

```
ONE bucket per user
```

That's a **policy/design decision**.

For example, you could have:

```
User A → 5 req / 10 sec
User B → 5 req / 10 sec
User C → 5 req / 10 sec
```

with separate counters/windows.

You could also have:

```
ALL USERS → 10,000 req / sec
```

with one global limiter.

Or both:

```
Global:
10,000 req/sec

AND

Per user:
5 req/sec
```

So in a real API gateway you might have **multiple rate limits simultaneously**.

For example:

```
                   API Gateway
                       │
          ┌────────────┴────────────┐
          ↓                         ↓
   Global limiter              User limiter
   10,000 req/sec               5 req/sec
          │                         │
          └────────────┬────────────┘
                       ↓
                    Backend
```

That distinction is important when you move from learning the algorithms to designing a production rate limiter.



> [!Question] I have leaky bucket lets say allowing 5 request per 10 sec and similarly we have fixed window size where a window is of 10 sec and allows 5 request max in that window, so isn't both same then?

Yes — **if you describe Leaky Bucket only as “5 requests every 10 seconds,” it sounds identical to Fixed Window. But they are fundamentally different.**

The key is **what happens to requests after the limit is reached**.

### Fixed Window

Configuration:

```
10-second window
Maximum = 5 requests
```

Think:

```
10:00:00 ─────────────── 10:00:10
       max 5 requests
```

Requests:

```
10:00:01  ✅
10:00:02  ✅
10:00:03  ✅
10:00:04  ✅
10:00:05  ✅
10:00:06  ❌
```

The 6th request is simply **rejected**.

Then at `10:00:10`, a **new window starts**:

```
10:00:10 ─────────────── 10:00:20
       counter = 0
```

So you can get the boundary burst:

```
09:59:59 → 5 requests ✅
10:00:01 → 5 requests ✅
```

That's potentially **10 requests in ~2 seconds**.

---

# Leaky Bucket is different

Here's the important mental model:

**Leaky Bucket doesn't normally mean "5 requests are allowed in every 10-second box."**

Instead, imagine a bucket/queue:

```
             Requests
          ↓   ↓   ↓   ↓
        ┌─────────────┐
        │             │
        │   BUCKET    │
        │             │
        └──────┬──────┘
               ↓
          fixed rate
               ↓
          Server
```

Suppose the bucket drains at:

```
5 requests / 10 sec
```

That's:

```
1 request every 2 seconds
```

If 5 requests arrive quickly:

```
t=0       Request A ──┐
t=0       Request B ──┤
t=0       Request C ──┤ → bucket
t=0       Request D ──┤
t=0       Request E ──┘

             ↓
          queue them

             ↓
t=0      A → server
t=2      B → server
t=4      C → server
t=6      D → server
t=8      E → server
```

So **the requests aren't necessarily rejected just because 5 have arrived**.

They can be **queued and released at a controlled rate**.

---

# This is the fundamental difference

### Fixed Window

> "Have I already received 5 requests during this 10-second window?"

```
5 requests → allow
6th request → reject
```

### Leaky Bucket

> "Can I put this request into my bucket/queue, and then I'll process requests at my fixed drain rate?"

```
requests arrive quickly
        ↓
     queue
        ↓
1 request every 2 sec
        ↓
     server
```

---

## But there's an important terminology issue

You will sometimes see tutorials describing **Leaky Bucket as a rate limiter that simply rejects requests when the bucket is full**, rather than queueing them.

For example:

```
Bucket capacity = 5
Drain rate = 1 request / 2 sec
```

If 5 requests are already waiting:

```
6th request → bucket full → reject
```

Even then, it is **still not the same as Fixed Window**, because the bucket's capacity and **continuous drain rate** determine admission.

There are no hard 10-second reset boundaries.

For example:

```
Fixed Window:

10:00:00 ───────── 10:00:10
     5 allowed

10:00:10
     RESET!
     ↓
     5 more allowed
```

Whereas Leaky Bucket is continuously draining:

```
10:00:00 → drain
10:00:02 → drain
10:00:04 → drain
10:00:06 → drain
10:00:08 → drain
10:00:10 → drain
...
```

**No reset happens at 10 seconds.**

---

### The easiest way to remember

```
FIXED WINDOW
"Count requests inside this box."

|--------------------|
       max 5
|--------------------|
       RESET
|--------------------|
       max 5
```

```
LEAKY BUCKET
"Put requests in a queue and drain continuously."

↓↓↓↓ requests
┌───────────┐
│ A B C D E │
└─────┬─────┘
      ↓
      A       t=0
      B       t=2
      C       t=4
      D       t=6
      E       t=8
```

So **your observation is correct at the superficial level**: if both are described merely as _“5 requests per 10 seconds,”_ they look the same.

But **Fixed Window controls the number admitted per fixed time box**, while **Leaky Bucket controls the rate at which queued traffic leaves the bucket**. That's the conceptual difference you should remember.