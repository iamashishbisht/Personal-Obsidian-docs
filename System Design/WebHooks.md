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



