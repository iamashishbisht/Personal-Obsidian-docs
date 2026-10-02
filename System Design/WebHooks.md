Suppose your application uses a payment provider such as [Stripe](https://stripe.com/). When a payment succeeds, your backend needs to update the order status.

One option is to repeatedly ask the payment provider whether the payment has completed.

![[Pasted image 20260930231631.png]]


# 1. What Is a Webhook?

> A webhook is an HTTP callback. One system exposes an HTTP endpoint, and another system sends an HTTP request to that endpoint whenever a particular event occurs.

![[Pasted image 20260930231845.png]]

The sending system is usually called the **provider**. The receiving system exposes the endpoint and is called the **consumer** or **receiver**.

In the payment example, Stripe is the provider and your backend is the consumer. Instead of your system asking for updates, the provider tells you when something happens.

> [!NOTE] **HMAC** stands for **Hash-based Message Authentication Code**. It uses a **single shared secret key** along with a cryptographic hash function (like SHA-256) to simultaneously verify both the **data integrity** and the **authenticity** of a message.

![[Pasted image 20261002114922.png]]

> [!Question] you meant to say if using webhooks along with HMAC we have to get the secret key first and put it in our side to verify, it won't be transmitted in network?

**Exactly! You have understood it perfectly.**

When you configure a webhook (for example, with Stripe or Github, the secret key itself is **never transmitted** inside the webhook HTTP request.

Instead, the workflow happens exactly like this:

1. Setup (Done Once)

You go into the provider's dashboard (e.g., Stripe) and generate a **Secret Key**. You copy that secret key and manually paste it into your server's secure configuration (like an environment variable). **Now both sides have the key, and it never has to travel across the internet again.** 

2. The Provider Sends the Webhook

When an event happens, the provider:

1. Takes the raw **Message Body** (the JSON payload).
2. Uses their copy of the **Secret Key** to calculate the **HMAC hash**.
3. Attaches that hash result to the HTTP request as a custom **Header** (e.g., `X-Hub-Signature` or `Stripe-Signature`).
4. Sends the request to your URL.
5. Your Side Verifies It

When your server receives the request:

1. You grab the **Message Body** and the **Header Signature** from the incoming request.
2. You take your safely stored local copy of the **Secret Key**.
3. You run the exact same calculation: `HMAC(Your Secret Key, Received Message Body)`.
4. You compare **your calculated hash** against the **Header Signature** they sent.
If they match, you know for certain that the message came from the provider and wasn't tampered with. Because the actual _secret key_ is never transmitted, a Man-in-the-Middle eavesdropper can see the JSON and see the signature header, but they can't figure out the key used to make it.


# Webhook Retries

Suppose a provider sends an event, but your server is temporarily unavailable. The provider should not discard the event. Instead, it stores it and retries later, usually with **exponential backoff**. For example, it might retry after 1 minute, then 2, 4, and 8 minutes.

Providers also add **jitter**, a small random offset on each delay, so thousands of failed deliveries do not all retry at the same moment.

Eventually, retries must stop. Most systems define a maximum number of attempts or a retry window, such as 3 days. After that, the event moves to a failed-delivery queue, where someone can inspect it or replay it manually.

# Why Duplicate Webhooks Happen

Retries introduce another problem: duplicate delivery.

Suppose your server processes a webhook successfully, but the network fails before the provider receives the HTTP response. Your server knows the event succeeded. The provider does not. From its side, the safest option is to retry, so your application receives the same event again.

That is why most webhook systems provide **at-least-once delivery**, not exactly-once delivery. Your consumer must handle duplicate events safely.

How At-Least-Once Delivery Works:

- **The Process**: The sender transmits a message and waits for an acknowledgment. If no acknowledgment arrives due to a network drop or crash, the sender tries again.
- **The Result**: The receiver gets the message successfully, but a delayed acknowledgment means it might receive the same message two or more times.
- **The Tradeoff**: You prevent data loss completely, but you must deal with duplicate data.


# Idempotency

The standard solution to duplicate delivery is idempotency: processing the same event twice has the same effect as processing it once.

Every webhook event should have a unique event ID. When your application receives an event:

1. Check whether that event ID has already been processed.
2. If not, process the event and record the ID.
3. If it has, recognize the duplicate and ignore it.

Store processed IDs in reliable storage with a unique constraint, so two concurrent deliveries of the same event cannot both slip through:

![[Pasted image 20261002121659.png]]

When a duplicate arrives, return success, because the original was already accepted:

![[Pasted image 20261002121718.png]]

This does not create true exactly-once delivery. It makes repeated deliveries safe, which is the practical goal.

> [!Question] Having a primary key constraint without handling in code in consumer can result in not having duplicate entries, but is that correct approach to have db throwing error directly?

No, relying solely on a database unique or primary key constraint to throw errors is generally **not considered a best-practice approach** for production systems.

While it successfully prevents duplicate data, treating the database as your primary line of application logic introduces several architectural flaws.

Why Relying Solely on DB Errors is Problematic

- **Performance & Resource Exhaustion**: Your consumer still performs network round-trips, forces the database to evaluate indexes, and initiates a transaction roll-back every time a duplicate arrives. Under heavy duplicate spikes, this can exhaust connection pools and degrade DB performance.
- **Poison Pills & Consumer Blocks**: If your consumer isn't explicitly catching that specific primary key violation error, it might crash, fail to acknowledge the message, and get stuck in an infinite retry loop (**blocking the queue**).
- **Obscured Monitoring**: It mixes actual, critical database errors (like disk space issues or syntax errors) with expected operational events (duplicate messages), making alerts and logs noisy and difficult to parse.

---

The Better Approach: Catch and Handle (Idempotency)

The correct approach isn't to completely remove the database constraint—**you should always keep the primary key constraint** as a final safety net. Instead, you should handle the duplicate gracefully in your code.

|Approach|How it Works|Pros / Cons|
|---|---|---|
|**Idempotent Upsert** _(Recommended)_|Use `INSERT ... ON CONFLICT DO NOTHING` (PostgreSQL) or `INSERT IGNORE` (MySQL).|⚡ **Best**: The DB handles it natively in a single round-trip without throwing an application exception.|
|**Try-Catch Block**|Wrap the insert in a `try/catch`. Catch the specific _UniqueConstraintViolation_ exception, log a warning, and safely acknowledge (`ACK`) the message.|🛠️ **Good**: Prevents the consumer from crashing or choking the queue.|
|**Distributed Cache Check**|Check a fast, in-memory store like **Redis** for the message ID before hitting the DB.|🏎️ **Fastest**: Stops the duplicate before it ever touches your main relational database.|
# Return Quickly, Process Asynchronously

Another common mistake is doing too much work inside the webhook request itself.

Imagine your server receives a payment webhook. The handler updates the database, sends an email, generates an invoice, updates analytics, and calls another service, all before responding. If that takes too long, the provider hits its timeout and assumes delivery failed. Then it retries, and you might process the event again while the first request is still running.

A more reliable pattern keeps the webhook endpoint lightweight:

1. Validate the request.
2. Store the event durably, usually in a database or queue.
3. Return a success response as quickly as possible.

Background workers then handle the expensive processing asynchronously. This separates webhook delivery from business logic, so slow downstream systems cannot delay the acknowledgment.

The order matters. Do not return `200 OK` before the event is safely stored. If your process crashes after returning success but before saving, the provider has no reason to retry, and the event is lost.

