A client sends a request, the network drops the response, and now the client does not know whether the operation happened. Retrying feels natural, but a blind retry can charge a customer twice or create a duplicate order.

**Idempotency** is the property that makes retries safe: running an operation multiple times has the same intended effect as running it once.

# Natural Idempotency vs Engineered Idempotency

Some operations are naturally idempotent because they set a final state.

```
UPDATE users
SET status = 'ACTIVE'
WHERE id = 123;
`````

Running this statement several times leaves the user in the same state.

Other operations are not naturally idempotent because they change the value again or create something new each time.

```
UPDATE inventory
SET stock = stock + 10
WHERE item_id = 1;
```

Each retry adds 10 more units. To make this safe, attach a stable operation ID:

```
INSERT INTO inventory_changes (operation_id, item_id, delta)
VALUES ('shipment_789', 1, 10)
ON CONFLICT (operation_id) DO NOTHING;
```

The operation ID turns "add 10 units" into "apply shipment `shipment_789` once."

This distinction matters:

- **Natural idempotency:** The operation itself sets a final state.
- **Engineered idempotency:** The system records a stable operation ID and uses it to detect duplicates.

Payments, order creation, email delivery, webhook processing, and job submission usually need engineered idempotency.

Good keys are:

- **Client-generated:** The key exists before the first request is sent.
- **Stable across retries:** The same operation always uses the same key.
- **Unique for different operations:** Two different payment attempts do not share a key.
- **Scoped:** The server checks the key within a user, account, tenant, endpoint, or operation type.
- **Bound to the request:** The server rejects the same key if the request body changes.
- **Stored durably:** The key survives restarts and failover.


> [!Question] 1. How idempotency works here in this table with simple example
![[Pasted image 20261003092739.png]]

Think of this table as a **database-backed memory of a request**.

The important idea is:

> **Same idempotency key + same scope → treat repeated requests as the same operation.**

Let's use a simple **Create Order** example.

### 1. Client sends the request

```
POST /orders
Idempotency-Key: abc123

{
  "productId": "P10",
  "quantity": 2
}
```

Your application creates a key:

```
scope = "create-order:user-101"
key   = "abc123"
```

Initially, the DB might contain:

|scope|key|request_hash|status|response_status|response_body|resource_id|
|---|---|---|---|---|---|---|
|create-order:user-101|abc123|hash(X)|PROCESSING|NULL|NULL|NULL|

Here:

```
PRIMARY KEY (scope, key)
```

means you **cannot insert another row with**

```
create-order:user-101 + abc123
```

---

## 2. First request creates the order

Suppose the application does:

```
POST /orders
Idempotency-Key: abc123
```

It checks:

```
SELECT *
FROM idempotency_keys
WHERE scope = 'create-order:user-101'
  AND key = 'abc123';
```

No row exists.

So it inserts:

```
abc123 → PROCESSING
```

Then creates the actual order:

```
Order ID = ORD-5001
```

Then updates the idempotency record:

```
status          = COMPLETED
response_status = 201
response_body   = {
                    "orderId": "ORD-5001",
                    "status": "CREATED"
                  }
resource_id     = "ORD-5001"
completed_at    = ...
```

Now the table looks roughly like:

|key|status|response_status|resource_id|
|---|---|---|---|
|abc123|COMPLETED|201|ORD-5001|

---

# 3. Network problem happens

This is where idempotency becomes useful.

The server successfully created:

```
ORD-5001
```

but the response never reached the client.

The client thinks:

> "I don't know whether my order was created. I'll retry."

It sends **the exact same request**:

```
POST /orders
Idempotency-Key: abc123

{
  "productId": "P10",
  "quantity": 2
}
```

Your application checks the table.

It finds:

```
abc123 → COMPLETED
          ORD-5001
```

So it **does NOT create another order**.

Instead it returns the previously stored response:

```
201 Created

{
  "orderId": "ORD-5001",
  "status": "CREATED"
}
```

So:

```
First request
      ↓
Create order ORD-5001
      ↓
Save result against abc123
      ↓
Response lost
      ↓
Client retries
      ↓
Find abc123
      ↓
Return stored result
      ↓
NO second order
```

That's the core of idempotency.

---

# 4. Why `request_hash`?

This is an important part.

Suppose the client tries:

```
Idempotency-Key: abc123

{
  "productId": "P10",
  "quantity": 5
}
```

Notice:

```
same key
different request
```

The original request was:

```
{
  "productId": "P10",
  "quantity": 2
}
```

So the server can calculate:

```
hash(original request) = XYZ
hash(new request)      = ABC
```

But the key already belongs to `XYZ`.

Therefore:

```
abc123 + XYZ → valid retry
abc123 + ABC → conflict
```

Typically return something like:

```
409 Conflict
```

Conceptually:

```
Idempotency-Key: abc123

First request:
    hash = XYZ
    → accepted

Second request:
    hash = XYZ
    → same request → return previous result

Third request:
    hash = ABC
    → different request → reject
```

This prevents someone from accidentally reusing an idempotency key for a completely different operation.

---

# 5. What is `scope` for?

Suppose the key is only:

```
abc123
```

Different users could theoretically generate/use the same key.

For example:

```
User A → abc123
User B → abc123
```

You don't necessarily want those to collide.

So you can make the effective identity:

```
(scope, key)
```

For example:

```
user-101 + abc123
user-202 + abc123
```

These are different records:

|scope|key|status|
|---|---|---|
|user-101|abc123|COMPLETED|
|user-202|abc123|COMPLETED|

That's why the primary key is:

```
PRIMARY KEY (scope, key)
```

---

# 6. What is `resource_id`?

This tells you **what was actually created/changed**.

For example:

```
resource_id = ORD-5001
```

So you can think:

```
Idempotency key
       ↓
"Which operation was this?"
       ↓
abc123
       ↓
"What resource did it produce?"
       ↓
ORD-5001
```

This is particularly useful when the response isn't enough or when you need to associate the idempotency record with a database entity.

---

# 7. What is `locked_until`?

This handles a slightly more complicated situation.

Imagine two identical requests arrive **at almost exactly the same time**:

```
Request A ───────→ Server
Request B ───────→ Server
```

Both might check:

```
SELECT ...
WHERE scope = 'user-101'
AND key = 'abc123';
```

and both see:

```
NO ROW
```

Without proper concurrency handling, both could proceed and create:

```
Request A → ORD-5001
Request B → ORD-5002   ❌
```

So the application can create a `PROCESSING` record and use locking/ownership information such as:

```
status = PROCESSING
locked_until = 09:20
```

The second request sees:

```
abc123 → PROCESSING
```

and knows:

> "Someone else is currently processing this idempotency key."

It can wait/retry, or return an appropriate response depending on the API design.

`locked_until` is especially useful for **recovering from abandoned/stuck processing**.

For example:

```
09:00 → Request A starts
09:00 → status = PROCESSING
09:01 → Request A crashes
09:05 → locked_until expires
```

Now another request can potentially take over processing according to your recovery rules.

---

# 8. What do the other columns mean?

### `status`

Usually something like:

```
PROCESSING
COMPLETED
FAILED
```

It tells you where the original operation is.

### `response_status`

Stores HTTP status:

```
201
200
400
409
500
```

So a retry can reproduce the previous response.

### `response_body`

Stores the actual response:

```
{
  "orderId": "ORD-5001",
  "status": "CREATED"
}
```

That's why the retry doesn't have to execute the business operation again.

### `created_at`

When the idempotency record was created.

### `completed_at`

When processing finished.

---

# The complete picture

Think of the table as:

```
                  idempotency_keys
                         │
                         │
          ┌──────────────┴──────────────┐
          │                             │
     "Have I seen                  "What happened
      this request?"                last time?"
          │                             │
     scope + key                 status + response
          │                             │
          └──────────────┬──────────────┘
                         ↓
                  Prevent duplicate
                    side effects
```

For our example:

```
Client
  │
  │ POST /orders
  │ Key = abc123
  ↓
API
  │
  │ INSERT idempotency record
  ↓
DB
  │
  │ PROCESSING
  ↓
Create Order
  │
  │ ORD-5001
  ↓
Update idempotency record
  │
  │ COMPLETED
  │ response = ORD-5001
  ↓
Client
```

If client retries:

```
Client
  │
  │ POST /orders
  │ Key = abc123
  ↓
API
  │
  │ lookup abc123
  ↓
DB
  │
  │ COMPLETED
  │ ORD-5001
  ↓
API
  │
  │ return stored response
  ↓
Client
```

**No second order is created.**

One subtle but very important point: **the table by itself does not magically make an API idempotent.** The application must implement the lookup/insert/locking/update logic correctly, and the database transaction/concurrency strategy matters. The `PRIMARY KEY (scope, key)` is one of the mechanisms that makes the design safe under concurrent requests.


> [!Question] 2. when we talk about lease right? but it isn't required because the order creation and idempotency table update would be atomic operations?

Assume we have **one PostgreSQL database** containing two tables:

```
PostgreSQL DB
│
├── idempotency_keys
│
└── orders
```

The important point is:

- `idempotency_keys` → remembers the request
- `orders` → contains the actual order

And the API is:

```
POST /orders
```

with:

```
Idempotency-Key: ABC123
```

---

## 1. First request comes

Client sends:

```
POST /orders
Idempotency-Key: ABC123

product = Laptop
```

### Step 1 — API checks `idempotency_keys`

```
SELECT *
FROM idempotency_keys
WHERE scope = 'create-order'
AND key = 'ABC123';
```

Nothing exists.

So API inserts into **`idempotency_keys`**:

```
┌─────────────────────────────────────────┐
│ idempotency_keys                        │
├───────────┬────────────┬───────────────┤
│ key       │ status     │ locked_until  │
├───────────┼────────────┼───────────────┤
│ ABC123    │ PROCESSING │ 10:05         │
└───────────┴────────────┴───────────────┘
```

This means:

> "I'm currently processing request ABC123."

---

## 2. API creates the actual order

Now the API inserts into the **`orders` table**:

```
┌──────────────────────────┐
│ orders                   │
├──────────┬───────────────┤
│ order_id │ product       │
├──────────┼───────────────┤
│ ORD-100  │ Laptop        │
└──────────┴───────────────┘
```

Then API updates **`idempotency_keys`**:

```
status          = COMPLETED
response_status = 201
resource_id     = ORD-100
response_body   = {"orderId":"ORD-100"}
```

Final state:

```
idempotency_keys
ABC123 → COMPLETED → ORD-100

orders
ORD-100 → Laptop
```

Everything is fine.

---

# Now the interesting part: SERVER CRASH

Suppose the API crashes **after inserting into `orders` but before updating `idempotency_keys`**.

Timeline:

```
Request ABC123
     │
     ▼
idempotency_keys
ABC123 → PROCESSING
     │
     ▼
orders
ORD-100 created
     │
     ▼
💥 SERVER CRASH
     │
     X
Never updated idempotency_keys to COMPLETED
```

So the database now looks like:

### `idempotency_keys`

```
ABC123 → PROCESSING
locked_until = 10:05
```

### `orders`

```
ORD-100 → Laptop
```

This is the problem.

The system **doesn't know from the idempotency row that the order was successfully created**.

---

# Now another request with SAME key arrives

At 10:02:

```
POST /orders
Idempotency-Key: ABC123
```

API checks **`idempotency_keys`**:

```
ABC123
status = PROCESSING
locked_until = 10:05
```

Current time:

```
10:02
```

Since:

```
10:02 < 10:05
```

the lease has not expired.

So the API says:

> "Someone is still processing ABC123. Don't start another operation."

It does **not** insert another row into `orders`.

It can return:

```
409 Conflict
```

or some "still processing, retry later" response.

---

# Now same request comes at 10:07

Client retries again:

```
POST /orders
Idempotency-Key: ABC123
```

API checks **`idempotency_keys`**:

```
locked_until = 10:05
current time = 10:07
```

The lease has expired.

So API can say:

> "The previous server that owned ABC123 is probably dead. I'll take ownership."

It updates **`idempotency_keys`**:

```
ABC123
status = PROCESSING
locked_until = 10:12
```

Now it needs to safely continue the operation.

It checks the **`orders` table**:

```
SELECT *
FROM orders
WHERE ...;
```

Suppose it finds:

```
ORD-100 → Laptop
```

That means:

> "Ah, the previous request actually created the order before crashing."

So it **doesn't create another order**.

Instead it updates **`idempotency_keys`**:

```
ABC123
status          = COMPLETED
resource_id     = ORD-100
response_status = 201
```

Now everything is consistent again.

---

# What if the first request crashed BEFORE creating the order?

Then after the crash:

### `idempotency_keys`

```
ABC123 → PROCESSING
locked_until = 10:05
```

### `orders`

```
(no order for ABC123)
```

At 10:07 another request takes over the expired lease.

It checks `orders`:

```
No order exists
```

So it creates:

```
ORD-101 → Laptop
```

Then updates:

```
idempotency_keys

ABC123 → COMPLETED → ORD-101
```

---

# What about a DIFFERENT idempotency key?

While ABC123 is processing, suppose another client sends:

```
POST /orders
Idempotency-Key: XYZ999
```

The API checks **`idempotency_keys`**:

```
ABC123 → PROCESSING
XYZ999 → doesn't exist
```

So it creates:

```
idempotency_keys

ABC123 → PROCESSING
XYZ999 → PROCESSING
```

Then it can create another order:

```
orders

ORD-100 → Laptop
ORD-101 → Phone
```

There is no conflict because:

```
ABC123 ≠ XYZ999
```

They represent **two different logical operations**.

---

## The whole thing in one picture

```
                    CLIENT
                      │
                      │ POST /orders
                      │ Idempotency-Key = ABC123
                      ▼
                ┌─────────────┐
                │ ORDER API   │
                └──────┬──────┘
                       │
                       │ 1. Check / update
                       ▼
             ┌──────────────────────┐
             │ PostgreSQL           │
             │                      │
             │ idempotency_keys     │
             │                      │
             │ ABC123 → PROCESSING  │
             │          lease 10:05 │
             └──────────┬───────────┘
                        │
                        │ 2. Create actual order
                        ▼
             ┌──────────────────────┐
             │ PostgreSQL           │
             │                      │
             │ orders               │
             │                      │
             │ ORD-100 → Laptop     │
             └──────────────────────┘
```

If the server crashes:

```
idempotency_keys
ABC123 → PROCESSING → lease expires

orders
ORD-100 → Laptop
```

Then a retry with **ABC123** can take the expired lease and inspect the **orders table** to determine whether the business operation already happened.

### One crucial correction to the mental model

The lease **doesn't make the order creation safe**.

It only answers:

> **"Who is currently allowed to work on this idempotency key?"**

You still need a safe way to determine:

> **"Did the previous attempt already create the order?"**

That's why a real implementation often also stores the idempotency key/request ID with the business record, or has another unique business constraint.

> [!Question] 3.The flow is [create/update/retrieve from idempotency table -> order table -> update idempotency table] lets say server crashed after creating order and updating order table and unable to update idempotency table. next request leases with same key by using locked_until, below two question:
> 1.  i know with idempotent key they can come to know which record is in idempotency table, and retrieve it and see it, but when it proceed to check order table on which basis it will go and check it that order is created or not what would ideally be the key?
> 2. so lease time you mentioned almost 5 mins, which means client remains on same screen for 5 mins and retry again, normally user exits the screen?

Yes — these are exactly the two things that need to be made explicit. And there is an important correction to the earlier example: **the idempotency key itself should ideally be carried into the `orders` table** if you want to reliably reconcile a crashed request.

## 1. On what basis does the retry check the `orders` table?

Suppose the first request was:

```
Idempotency-Key = ABC123
```

The API creates:

### `idempotency_keys`

```
scope          key       status
---------------------------------------
create-order   ABC123    PROCESSING
```

Then it creates:

### `orders`

Ideally, you store the idempotency key (or an equivalent request/business ID) here too:

```
order_id   product   idempotency_key
------------------------------------
ORD-100    Laptop    ABC123
```

Now the server crashes before updating `idempotency_keys`.

Later, the retry comes with:

```
Idempotency-Key = ABC123
```

The API finds:

```
idempotency_keys
ABC123 → PROCESSING → lease expired
```

It takes over the lease.

Then it can ask the **orders table**:

```
SELECT *
FROM orders
WHERE idempotency_key = 'ABC123';
```

If it finds:

```
ORD-100 | Laptop | ABC123
```

it knows:

> "The previous attempt already created this order."

So it does **not** create another one.

It can then update:

```
idempotency_keys
ABC123 → COMPLETED
resource_id = ORD-100
```

### So the important relationship is:

```
                  ABC123
                    │
          ┌─────────┴─────────┐
          ↓                   ↓
 idempotency_keys           orders
          │                   │
 ABC123 → PROCESSING     ORD-100 → ABC123
```

This is why I would prefer a design where the business record has some **unique operation/request identifier** associated with the idempotency key.

For example:

```
ALTER TABLE orders
ADD COLUMN idempotency_key TEXT;

CREATE UNIQUE INDEX
ON orders(scope, idempotency_key);
```

Then you get an additional safety mechanism:

```
ABC123 → ORD-100
```

can exist only once.

### Even better mental model

The idempotency table answers:

> **"What happened to request ABC123?"**

The order table answers:

> **"Did ABC123 already create an order?"**

They are related through the same operation identifier.

---

# 2. Does the user really wait 5 minutes?

No. **The user normally should NOT sit on the screen for five minutes.**

This is where we should separate **lease duration** from **client retry behavior**.

Suppose:

```
locked_until = now + 5 minutes
```

That does **not** mean:

> "Tell the user to wait five minutes."

It means:

> "Give the current server/process up to five minutes to finish before another worker is allowed to assume it has died."

For example:

```
10:00:00  Request A starts
          lease = 10:05:00

10:00:02  A creates order

10:00:03  💥 A crashes before completing idempotency record
```

The user might see:

```
"Something went wrong. Please try again."
```

They may immediately leave the screen.

---

## What happens when they come back?

Suppose they come back at:

```
10:01
```

and press **Retry**.

The new request still sends:

```
Idempotency-Key = ABC123
```

The server sees:

```
ABC123
PROCESSING
locked_until = 10:05
```

It knows another owner supposedly still has the lease.

So it could return:

```
"Request is still being processed. Please try again."
```

But this raises a practical design question:

> **Do we really want to make the user wait until 10:05?**

Usually, **no**.

You can design the system differently depending on the business operation.

For an order/payment operation, you'd generally want stronger recovery/reconciliation rather than simply making the user wait for the entire lease.

---

# The 5 minutes is not necessarily a good universal value

This is important.

I previously used **5 minutes as an illustrative example**, not as a recommended universal setting.

The appropriate lease depends on your operation.

For example:

```
Normal operation duration
        ↓
       2 sec
```

You might use something like:

```
lease = 30 sec
```

rather than 5 minutes.

For a slow operation:

```
Normal operation
       ↓
     30 sec
```

perhaps:

```
lease = 2 min
```

The point is:

```
lease > expected processing time
```

while still being short enough to recover from crashes reasonably quickly.

---

# And there is another pattern: background retry

A real system doesn't necessarily rely on the **human** to retry.

Imagine:

```
Client
   │
   │ POST /orders
   ▼
Order API
   │
   ├── create idempotency record
   │
   ├── process order
   │
   └── 💥 crash
```

The user gets an error.

But you could have another mechanism:

```
Scheduler / worker
        │
        │ periodically finds
        │ expired PROCESSING records
        ▼
idempotency_keys
        │
        ▼
ABC123 → expired
        │
        ▼
recovery worker
        │
        ▼
check orders
        │
        ├── order exists → mark COMPLETED
        │
        └── doesn't exist → retry creation
```

So **the client doesn't necessarily have to sit there and retry**.

---

## One more important thing about the user's second question

There are actually **three different retries** you should distinguish:

### A. Browser/mobile app automatically retries

```
Network timeout
   ↓
client automatically retries
```

This could happen seconds later.

### B. User manually presses Retry

```
Error shown
   ↓
user presses Retry
```

Could be 10 seconds later, 5 minutes later, or tomorrow.

The important thing is that they reuse:

```
ABC123
```

for the **same logical operation**.

### C. Backend recovery retries

```
expired lease
   ↓
worker detects it
   ↓
worker reconciles/retries
```

This doesn't involve the user at all.

---

## The key thing to remember

Don't think:

> `locked_until = 5 minutes` → **user waits 5 minutes**

Think:

> `locked_until = 5 minutes` → **the server gives the current owner a 5-minute lease before another server/worker can assume ownership.**

And separately:

```
Idempotency-Key
       │
       ├── idempotency_keys
       │       ↓
       │   tracks request state
       │
       └── orders
               ↓
           ideally stores the same
           operation/request identifier
               ↓
           lets us determine whether
           the order was already created
```

That relationship is the part that makes the crash-recovery example actually concrete.


---
---

# Handling In-Progress Requests

Retries can arrive at the same time. A user double-clicks, a mobile client retries aggressively, or two workers process the same job.

When a duplicate request arrives while the first request is still running, the server has a few options:

![[Pasted image 20261004105005.png]]

For long-running operations, returning an operation resource is often cleaner than holding the connection open:

```
{
  "id": "payment_attempt_123",
  "status": "processing",
  "retry_after_seconds": 2
}
```

# External Side Effects

The hardest idempotency bugs happen when the operation calls another system.

Example payment flow:

1. Reserve idempotency key.
2. Create local payment attempt.
3. Call payment provider.
4. Provider charges the card.
5. Local service crashes before saving the provider result.

On retry, the service must not charge again. Better designs reduce this risk:

- Pass an idempotency key to the external provider when the provider supports it.
- Store a local payment attempt before calling the provider.
- Store the provider's charge ID as soon as it is known.
- Recover by querying the provider using its request ID or saved call details.
- Use a workflow engine for long-running, multi-step operations when appropriate.
- Keep side effects behind database uniqueness checks where possible.

No database transaction can include every external API. The design needs a recovery path for this case: "the external action happened, but our local state did not finish updating."

# HTTP Method Behavior

![[Pasted image 20261004105948.png]]


> [!Question] Tell me if creating idempotent key as UUID, is it responsibility of a client to pass it a day after as same idempotency key as you said earlier. lets say if it is a responsibility of client and it tried today from web and tommorow retrying from andoid mobile now how android mobile know that from which idempotent key i have to retry?

Yes — **your confusion is valid.** The important answer is:

> **If the client generates the idempotency key, the client is responsible for remembering and reusing it for retries.**

But your example exposes an important limitation: **a completely different client/device cannot magically know yesterday's idempotency key.**

### Your exact example

Today:

`Web browser`
    `↓`
`User clicks "Create Order"`
    `↓`
`Web generates UUID-A`
    `↓`
`Idempotency-Key: UUID-A`
    `↓`
`Backend`

Suppose the request actually created the order, but the response was lost.

Tomorrow:

`Android mobile ↓ "Retry"`

How does Android know UUID-A?

**It doesn't.**

Unless the system has deliberately made that key available to Android.

---

## So what happens in a real production system?

Usually, you **don't design cross-device retry around the client-generated idempotency key**.

There are two common approaches.

### Approach 1 — Client keeps the key

This works when retries happen from the **same client/session**.

`Web`
  `│`
  `├── UUID-A → server`
  `│`
  `├── save UUID-A locally`
  `│`
  `└── retry → UUID-A`

For example, mobile app can store:

`pendingOrder {`
    `orderDetails: ...`
    `idempotencyKey: UUID-A`
`}`

If the app crashes and restarts, it can still retry with UUID-A.

---

### Approach 2 — Server gives you an operation/order reference

This is more appropriate for your **Web → next day Android** example.

The system can have a business-level operation ID:

`User starts order`
      `↓`
`Server creates operation`
      `↓`
`operationId = OP-123`
      `↓`
`User can see "Order attempt OP-123"`

Then Android can retrieve the pending operation:

`Android`
   `↓`
`GET /orders/pending`
   `↓`
`Server`
   `↓`
`OP-123`

Now Android knows which operation it is dealing with.

It can then continue/reconcile that operation.

---

### The key correction to what I said earlier

I made it sound like:

> "If the user comes back tomorrow, just reuse the same idempotency key."

That's **only true if the client still has that key**.

If today's request came from Web and tomorrow's retry comes from Android, **Android does not inherently know the UUID**.

So the better mental model is:

                    Idempotency Key
                         UUID-A
                           │
                           ↓
                  identifies ONE operation
                           │
             ┌─────────────┴─────────────┐
             ↓                           ↓
       Web retry                     Android retry
       can reuse UUID-A              cannot know UUID-A
       if it saved it                unless system gives it

And this is why production systems often combine **idempotency keys + persistent business/operation identifiers + database constraints**, rather than expecting a new device to somehow reconstruct an old UUID.


# Best Practices

Use this checklist when building retry-safe endpoints, consumers, and calls to other services.

- Use stable operation IDs for operations with side effects.
- Require idempotency keys for retryable endpoints that can create side effects, such as payment creation and job submission.
- Scope keys by tenant, caller, endpoint, or operation type.
- Store request hashes and reject mismatches.
- Reserve keys atomically.
- Store the original response when client consistency matters.
- Keep business state and duplicate-detection state in the same transaction when possible.
- Pass idempotency keys to providers that support them.
- Treat message consumers as duplicate-tolerant by default.
- Document key retention windows.
- Test concurrent retries, timeouts, process crashes, provider failures, and broker redelivery.