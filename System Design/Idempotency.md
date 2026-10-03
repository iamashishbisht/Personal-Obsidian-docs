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


> [!Question] How idempotency works here in this table with simple example
![[Pasted image 20261003092739.png]]

Think of this table as a **database-backed memory of a request**.

The important idea is:

> **Same idempotency key + same scope → treat repeated requests as the same operation.**

Let's use a simple **Create Order** example.

### 1. Client sends the request

`POST /orders Idempotency-Key: abc123 { "productId": "P10", "quantity": 2 }`

Your application creates a key:

`scope = "create-order:user-101" key = "abc123"`

Initially, the DB might contain:

|scope|key|request_hash|status|response_status|response_body|resource_id|
|---|---|---|---|---|---|---|
|create-order:user-101|abc123|hash(X)|PROCESSING|NULL|NULL|NULL|

Here:

`PRIMARY KEY (scope, key)`

means you **cannot insert another row with**

`create-order:user-101 + abc123`

---

## 2. First request creates the order

Suppose the application does:

`POST /orders Idempotency-Key: abc123`

It checks:

```
SELECT *
FROM idempotency_keys
WHERE scope = 'create-order:user-101'
  AND key = 'abc123';
```

No row exists.

So it inserts:

`abc123 → PROCESSING`

Then creates the actual order:

`Order ID = ORD-5001`

Then updates the idempotency record:

`status = COMPLETED response_status = 201 response_body = { "orderId": "ORD-5001", "status": "CREATED" } resource_id = "ORD-5001" completed_at = ...`

Now the table looks roughly like:

|key|status|response_status|resource_id|
|---|---|--:|---|
|abc123|COMPLETED|201|ORD-5001|

---

# 3. Network problem happens

This is where idempotency becomes useful.

The server successfully created:

`ORD-5001`

but the response never reached the client.

The client thinks:

> "I don't know whether my order was created. I'll retry."

It sends **the exact same request**:

`POST /orders Idempotency-Key: abc123 { "productId": "P10", "quantity": 2 }`

Your application checks the table.

It finds:

`abc123 → COMPLETED ORD-5001`

So it **does NOT create another order**.

Instead it returns the previously stored response:

`201 Created { "orderId": "ORD-5001", "status": "CREATED" }`

So:

`First request ↓ Create order ORD-5001 ↓ Save result against abc123 ↓ Response lost ↓ Client retries ↓ Find abc123 ↓ Return stored result ↓ NO second order`

That's the core of idempotency.

---

# 4. Why request_hash?

This is an important part.

Suppose the client tries:

`Idempotency-Key: abc123 { "productId": "P10", "quantity": 5 }`

Notice:

`same key different request`

The original request was:

```
{
  "productId": "P10",
  "quantity": 2
}
```

So the server can calculate:

`hash(original request) = XYZ hash(new request) = ABC`

But the key already belongs to XYZ.

Therefore:

`abc123 + XYZ → valid retry abc123 + ABC → conflict`

Typically return something like:

`409 Conflict`

Conceptually:

`Idempotency-Key: abc123 First request: hash = XYZ → accepted Second request: hash = XYZ → same request → return previous result Third request: hash = ABC → different request → reject`

This prevents someone from accidentally reusing an idempotency key for a completely different operation.

---

# 5. What is scope for?

Suppose the key is only:

`abc123`

Different users could theoretically generate/use the same key.

For example:

`User A → abc123 User B → abc123`

You don't necessarily want those to collide.

So you can make the effective identity:

`(scope, key)`

For example:

`user-101 + abc123 user-202 + abc123`

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

# 6. What is resource_id?

This tells you **what was actually created/changed**.

For example:

`resource_id = ORD-5001`

So you can think:

`Idempotency key ↓ "Which operation was this?" ↓ abc123 ↓ "What resource did it produce?" ↓ ORD-5001`

This is particularly useful when the response isn't enough or when you need to associate the idempotency record with a database entity.

---

# 7. What is locked_until?

This handles a slightly more complicated situation.

Imagine two identical requests arrive **at almost exactly the same time**:

`Request A ───────→ Server Request B ───────→ Server`

Both might check:

```
SELECT ...
WHERE scope = 'user-101'
AND key = 'abc123';
```

and both see:

`NO ROW`

Without proper concurrency handling, both could proceed and create:

`Request A → ORD-5001 Request B → ORD-5002 ❌`

So the application can create a PROCESSING record and use locking/ownership information such as:

`status = PROCESSING locked_until = 09:20`

The second request sees:

`abc123 → PROCESSING`

and knows:

> "Someone else is currently processing this idempotency key."

It can wait/retry, or return an appropriate response depending on the API design.

locked_until is especially useful for **recovering from abandoned/stuck processing**.

For example:

`09:00 → Request A starts 09:00 → status = PROCESSING 09:01 → Request A crashes 09:05 → locked_until expires`

Now another request can potentially take over processing according to your recovery rules.

---

# 8. What do the other columns mean?

### status

Usually something like:

`PROCESSING COMPLETED FAILED`

It tells you where the original operation is.

### response_status

Stores HTTP status:

`201 200 400 409 500`

So a retry can reproduce the previous response.

### response_body

Stores the actual response:

```
{
  "orderId": "ORD-5001",
  "status": "CREATED"
}
```

That's why the retry doesn't have to execute the business operation again.

### created_at

When the idempotency record was created.

### completed_at

When processing finished.

---

# The complete picture

Think of the table as:

`idempotency_keys │ │ ┌──────────────┴──────────────┐ │ │ "Have I seen "What happened this request?" last time?" │ │ scope + key status + response │ │ └──────────────┬──────────────┘ ↓ Prevent duplicate side effects`

For our example:

`Client │ │ POST /orders │ Key = abc123 ↓ API │ │ INSERT idempotency record ↓ DB │ │ PROCESSING ↓ Create Order │ │ ORD-5001 ↓ Update idempotency record │ │ COMPLETED │ response = ORD-5001 ↓ Client`

If client retries:

`Client │ │ POST /orders │ Key = abc123 ↓ API │ │ lookup abc123 ↓ DB │ │ COMPLETED │ ORD-5001 ↓ API │ │ return stored response ↓ Client`

**No second order is created.**

One subtle but very important point: **the table by itself does not magically make an API idempotent.** The application must implement the lookup/insert/locking/update logic correctly, and the database transaction/concurrency strategy matters. The PRIMARY KEY (scope, key) is one of the mechanisms that makes the design safe under concurrent requests.