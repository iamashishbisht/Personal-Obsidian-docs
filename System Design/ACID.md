- **Atomicity:** A transaction is an all-or-nothing affair; if any part fails, the entire operation is rolled back.
- **Consistency:** A transaction must move the database from one valid state to another while adhering to all defined rules and constraints. e.g primary, foreign, checks, not null.
- **Isolation:** Concurrent transactions execute independently without interfering with or exposing intermediate states to one another.
- **Durability:** Once a transaction is committed, its changes are permanent and will survive any subsequent system crash or power loss

## Isolation

Isolation controls what transactions can see when they run at the same time.

This matters because production systems rarely do one thing at a time. Two users may check out at the same time. Two workers may process jobs at the same time. Two API requests may update the same account at the same time.

Without isolation, one transaction could read another transaction's half-finished work, or two transactions could both make decisions using old data.

Common concurrency problems:

|Problem|What Happens|
|---|---|
|Dirty read|A transaction reads data another transaction has not committed|
|Non-repeatable read|A transaction reads the same row twice and sees different committed values|
|Phantom read|A repeated query returns a different set of matching rows|
|Lost update|Two transactions overwrite each other's changes|
|Write skew|Two transactions read overlapping data and make writes that violate a rule together|

> [!Question] 1. what does this word mean? Non-repeatable read

A **non-repeatable read** (also known as a **fuzzy read**) is ==a database concurrency anomaly that occurs when a transaction reads the same row twice and gets two different values because another concurrent transaction modified and committed changes to that row in between the reads==.

How It Happens

- **First Read:** Transaction A reads a database row (e.g., checking an item's stock as `10` or a product's price as `$20`).
- **Concurrent Update:** Transaction B modifies that same row (e.g., updating stock to `5` or price to `$25`) and **commits** the change.
- **Second Read:** Transaction A reads the exact same row a second time later in the same transaction and sees the new value (`5` or `$25`), invalidating its initial assumptions.

Key Characteristics & Prevention

- **Committed Data:** Unlike a _dirty read_ (where a transaction reads uncommitted data that might roll back), a non-repeatable read involves data that has been officially committed by another transaction.
- **Isolation Levels:** This happens commonly under the `READ COMMITTED` isolation level (the default in databases like PostgreSQL, MySQL, and Oracle).
- **How to Prevent It:** Upgrade the transaction isolation level to **`REPEATABLE READ`** or **`SERIALIZABLE`**, which ensures that any row read by a transaction is locked or version-controlled (via MVCC) to maintain a stable value for the duration of the transaction.

> [!Question] 2. why the name says non-repeatable read.

The name **"non-repeatable read"** simply means that ==**you cannot successfully repeat the exact same read operation** and guarantee the same result==.

It describes the **failure** of a core expectation: if you read a piece of data right now, and then "repeat" that exact same read request a few seconds later within the same transaction, you _should_ get the identical result.

When this anomaly occurs, the read is **"non-repeatable"** because the data changed out from under you.

A Real-World Analogy

Imagine you look at a flight booking screen and see **1 seat left** (First Read). You spend two minutes typing in your credit card details. Right before hitting purchase, the system refreshes to double-check the seat count (Repeated Read), but someone else bought it a second ago, so it now says **0 seats left**.

Because you could not _repeat_ your initial finding, the first read was **non-repeatable**.

The Opposite: "Repeatable Read"

To make the concept clearer, look at what it fixes. When a database provides a **"Repeatable Read"** isolation level, it promises you the exact opposite: **"We guarantee that if you repeat a read, you will always see the exact same data as the first time, no matter what other users are doing."**

> [!Question] 3. So why non-repeatbale read is a problem? in a real world another transaction will commit the changes and its ok right?

It feels logical that a database should always show the absolute latest, most accurate data. However, in the middle of a business transaction, seeing data change halfway through can cause **data corruption, incorrect calculations, and financial errors**.

A **non-repeatable read** is a problem because a single transaction expects a **consistent snapshot** of the world while it makes decisions. If the ground moves beneath it, the logic breaks.

Here are three real-world scenarios where letting a parallel transaction change data midway creates a massive problem:

### 1. The Financial Audit Nightmare (Incorrect Totals)

Imagine a bank running an automated end-of-day report to calculate total assets.

- **Step 1:** The report reads Account A, which has **$100**.
- **Step 2:** A customer quickly transfers $50 from Account A to Account B, and that transaction **commits**.
- **Step 3:** The report reads Account B. Because of the non-repeatable read, it sees the updated balance (Account B now has an extra **$50**).
- **The Problem:** The report counted the $50 in Account B, but _missed_ it in Account A because it read Account A before the transfer. The bank's final report now shows **$50 of magic money** that doesn't actually exist.

### 2. The Smart Contract / E-Commerce Loophole (Price Exploits)

Imagine a system calculating a bulk discount for an enterprise customer.

- **Step 1:** The system checks the unit price of a laptop: **$1,000**. It decides to grant a 10% discount based on this price.
- **Step 2:** A store manager pushes an emergency update changing the laptop price to **$2,000** to match a supplier spike, and it **commits**.
- **Step 3:** The system generates the final invoice using the new price ($2,000) but applies the original discount logic calculated from the old price.
- **The Problem:** The customer gets charged an incorrect, mismatched amount because the core variables changed mid-calculation.

### 3. The Inventory "Phantom" Checkout

Imagine an app preparing a shipping manifest for a warehouse worker.

- **Step 1:** The system reads the inventory table and sees **1 Xbox left** in Aisle 4. It assigns the worker to go pick it up.
- **Step 2:** A customer online cancels their order, or a separate inventory adjustment updates the count to **2 Xboxes**, and **commits**.
- **Step 3:** The system prints a label verifying the item count, but now reads **2**.
- **The Problem:** The internal logic of the software breaks because `If Count == 1` was true at the start of the code execution, but false by the end of it, leading to system crashes or wrong labels being printed.

The Takeaway

While it is "ok" and correct that the other transaction committed its changes to the database, it is **not ok for _your_ active transaction to see them mid-flight**. A transaction must be an isolated, predictable block of logic. If it starts with one version of reality, it must finish with that same version of reality.

> [!Question] 4. lets say if there is repeatable read, then in this case it can lock the db rows for reading right, no commits will be possible?

**It depends entirely on how your database is designed.**

Your intuition describes the **pessimistic locking** approach, but modern databases have a brilliant way around this so that **commits are still allowed, and reads never block writes.**

Databases achieve a `REPEATABLE READ` isolation level using one of two completely different strategies:

### Strategy 1: The Modern Way (MVCC / Snapshots) — No Locking! ❌🔒

Most modern databases—like **PostgreSQL**, **MySQL (InnoDB)**, and **Oracle**—do **not** lock rows for reading. Instead, they use a technology called [Multi-Version Concurrency Control (MVCC)](https://www.databricks.com/blog/concurrency-control). [[1](https://dev.to/franckpachot/isolation-levels-part-vii-repeatable-read-l11), [2](https://planetscale.com/blog/mysql-isolation-levels-and-how-they-work), [3](https://www.mydbops.com/blog/postgresql-transaction-isolation-levels-guide), [4](https://www.databricks.com/blog/concurrency-control)]

- **How it works:** When your transaction starts, the database takes a virtual **"snapshot"** of the data at that exact millisecond. [[1](https://www.mydbops.com/blog/postgresql-transaction-isolation-levels-guide), [2](https://dev.to/franckpachot/isolation-levels-part-vii-repeatable-read-l11), [3](https://www.linkedin.com/posts/krishna-kumar-mahto_postgresql-database-mvcc-activity-7462380237436239874-o_o4)]
- **When a write happens:** If another user updates a row and commits it, the database doesn't overwrite the old data directly. It creates a _new version_ of that row in the background. [[1](https://planetscale.com/blog/mysql-isolation-levels-and-how-they-work), [2](https://www.youtube.com/watch?v=IRRYaIAM1Zw&t=235)]
- **The Result:** Other transactions can freely commit updates all day long. However, whenever _your_ transaction reads data, it is automatically routed to the older versions from your initial snapshot. **Zero locking occurs, and performance remains extremely high.** [[1](https://www.linkedin.com/posts/krishna-kumar-mahto_postgresql-database-mvcc-activity-7462380237436239874-o_o4), [2](https://www.youtube.com/watch?v=29qBvRGMnp4&t=232), [3](https://www.mydbops.com/blog/postgresql-transaction-isolation-levels-guide)]

### Strategy 2: The Old / Strict Way (Pessimistic Locking) — Yes, it locks! 🛅

Older databases (or specific systems like **SQL Server** depending on how they are configured) use [Strict Two-Phase Locking (2PL)](https://medium.com/@dipayansanyal/database-isolation-why-your-reads-cant-always-be-repeatable-52095d1ac244) to achieve a repeatable read. [[1](https://medium.com/@dipayansanyal/database-isolation-why-your-reads-cant-always-be-repeatable-52095d1ac244), [2](https://www.databricks.com/blog/concurrency-control)]

- **How it works:** The moment your transaction reads a row, the database places a **Shared Lock (S-Lock)** on it.
- **The Rule:** Under `REPEATABLE READ`, this read-lock is held **until your entire transaction finishes** (commits or rolls back).
- **The Result:** In this specific scenario, **your intuition is 100% correct**. If another transaction tries to update or commit changes to those exact rows, it will be completely blocked and forced to wait until your transaction releases the lock. [[1](https://learn.microsoft.com/en-us/sql/odbc/reference/develop-app/transaction-isolation-levels?view=sql-server-ver17), [2](https://www.youtube.com/watch?v=58pFDUUY-hY&vl=en), [3](https://medium.com/@dipayansanyal/database-isolation-why-your-reads-cant-always-be-repeatable-52095d1ac244), [4](https://www.ibm.com/docs/en/informix-servers/14.10.0?topic=level-repeatable-read-isolation)]

Summary Comparison

|Database Engine|Uses Locks for `REPEATABLE READ`?|Can other transactions commit?|
|---|---|---|
|**PostgreSQL / MySQL**|**No** (Uses MVCC snapshots)|**Yes.** Concurrent updates are allowed without blocking.|
|**Traditional SQL Server / Informix**|**Yes** (Uses Shared Locks)|**No.** Other transactions must wait for your transaction to finish.|

Are you building an application right now where you are worried about **concurrency performance**, or are you studying this for a **system design / database interview**? Let me know so I can tailor the next step!


# Common Database Concurrency Problems — Explained with Two Transactions

# Common Database Concurrency Problems

## 1. Dirty Read

**Definition:** A transaction reads data modified by another transaction before that transaction commits.

**Initial data:**

```
Account balance = ₹1,000
```

**Execution flow:**

```
T1                                      T2
-----------------------------------     --------------------------------
                                        BEGIN TRANSACTION
                                        UPDATE balance = ₹500
                                        (Not committed yet)

BEGIN TRANSACTION
READ balance = ₹500
(Reads T2's uncommitted value)

                                        ROLLBACK
                                        (Balance returns to ₹1,000)

T1 has read ₹500, but that value
was never committed.
```

**Why is this a problem?**

T1 may make a decision based on data that is later rolled back.

**Key point:** T1 reads an uncommitted change made by T2.

**Prevention:** Use an isolation level that prevents dirty reads, such as `READ COMMITTED`.

---

## 2. Non-Repeatable Read

**Definition:** A transaction reads the same row twice but gets different values because another transaction updates and commits that row between the reads.

**Initial data:**

```
Employee ID = 101
Salary = ₹50,000
```

**Execution flow:**

```
T1                                      T2
-----------------------------------     --------------------------------
BEGIN TRANSACTION

READ salary = ₹50,000

                                        BEGIN TRANSACTION
                                        UPDATE salary = ₹60,000
                                        COMMIT

READ salary = ₹60,000

T1 read the same row twice but
received different values.
```

**Why is this a problem?**

T1 may produce inconsistent calculations or reports because the value changed during its transaction.

**Key point:** The same existing row returns different committed values.

**Prevention:** Use `REPEATABLE READ` or `SERIALIZABLE`, or a suitable snapshot-based isolation mechanism.

---

## 3. Phantom Read

**Definition:** A transaction executes the same query twice but gets a different set of matching rows because another transaction inserts or deletes matching rows.

**Initial data:**

```
Orders table:

Order ID | Status
---------|---------
1        | PENDING
2        | PENDING

Number of pending orders = 2
```

**Execution flow:**

```
T1                                      T2
-----------------------------------     --------------------------------
BEGIN TRANSACTION

SELECT COUNT(*)
WHERE status = 'PENDING'

Result = 2

                                        BEGIN TRANSACTION
                                        INSERT order ID 3,
                                        status = 'PENDING'
                                        COMMIT

SELECT COUNT(*)
WHERE status = 'PENDING'

Result = 3

A new matching row appeared
in T1's second query.
```

**Why is this a problem?**

T1 may make a decision using an outdated count or result set.

**Key point:** The set of rows matching the query changes.

**Prevention:** Use `SERIALIZABLE` isolation or another suitable mechanism that guarantees a stable view of the query's results.

**Difference from non-repeatable read:**

- Non-repeatable read: The value of an existing row changes.
    
- Phantom read: The set of rows returned by a query changes.
    

---

## 4. Lost Update

**Definition:** Two transactions read the same old value, calculate new values independently, and one transaction overwrites the other's update.

**Initial data:**

```
Account balance = ₹1,000

T1 wants to deposit ₹100.
T2 wants to deposit ₹200.

Expected final balance = ₹1,300
```

**Execution flow:**

```
T1                                      T2
-----------------------------------     --------------------------------
READ balance = ₹1,000

                                        READ balance = ₹1,000

Calculate: 1000 + 100 = 1100

                                        Calculate: 1000 + 200 = 1200

UPDATE balance = ₹1,100
COMMIT

                                        UPDATE balance = ₹1,200
                                        COMMIT

Final balance = ₹1,200
Expected balance = ₹1,300
```

**Why did this happen?**

T2 used the old balance of ₹1,000 and overwrote T1's updated balance of ₹1,100.

T1's deposit was effectively lost.

**Key point:** Both transactions update the same data using stale values.

**Prevention options:**

1. Use an atomic database update:
    

```
UPDATE account
SET balance = balance + 100
WHERE account_id = 1;
```

T2 performs the equivalent operation with `+ 200`. Both deposits are applied to the current database value.

2. Use optimistic locking with JPA's `@Version`.
    
3. Use an appropriate pessimistic locking strategy when necessary.
    

---

## 5. Write Skew

**Definition:** Two transactions read overlapping data and update different rows based on what they read. Both transactions commit, but their combined changes violate a business rule.

**Business rule:** At least one doctor must remain on call.

**Initial data:**

```
Doctor A: ON_CALL = TRUE
Doctor B: ON_CALL = TRUE

Business rule:
At least one doctor must remain on call.
```

**Execution flow:**

```
T1                                      T2
-----------------------------------     --------------------------------
READ Doctor A = ON
READ Doctor B = ON

                                        READ Doctor A = ON
                                        READ Doctor B = ON

T1 decides Doctor A can go off
because Doctor B is still on.

                                        T2 decides Doctor B can go off
                                        because Doctor A is still on.

UPDATE Doctor A = OFF
COMMIT

                                        UPDATE Doctor B = OFF
                                        COMMIT
```

**Final data:**

```
Doctor A: OFF
Doctor B: OFF

Business rule violated:
No doctor is on call.
```

**Why did this happen?**

Both transactions checked the same initial state and made individually reasonable decisions. However, neither transaction knew that the other would also take a doctor off call.

They updated different rows, so neither directly overwrote the other's update.

**Key point:** Different rows are updated, but a shared business rule is violated.

**Prevention options:**

1. Use `SERIALIZABLE` isolation.
    
2. Lock a shared coordination row before checking the business rule.
    
3. Redesign the operation to enforce the rule through an appropriate database constraint or atomic operation.
    

**Important:** Snapshot isolation alone may permit write skew.

---

# Quick Revision Table

|Problem|Main issue|Key difference|
|---|---|---|
|Dirty read|Reading uncommitted data|The writer may roll back|
|Non-repeatable read|Same row gives different values|Existing row changes|
|Phantom read|Query returns a different set of rows|Matching rows appear or disappear|
|Lost update|One update overwrites another|Same data, stale value|
|Write skew|Combined updates violate a rule|Different rows, shared business rule|

# Isolation Level Quick Reference

| Isolation level  | Dirty read | Non-repeatable read | Phantom read |
| ---------------- | ---------- | ------------------- | ------------ |
| READ UNCOMMITTED | Possible   | Possible            | Possible     |
| READ COMMITTED   | Prevented  | Possible            | Possible     |
| REPEATABLE READ  | Prevented  | Prevented           | Possible     |
| SERIALIZABLE     | Prevented  | Prevented           | Prevented    |

**Note:** These are the standard SQL isolation guarantees. Actual database behavior can differ with snapshot-based implementations.

Lost updates and write skew need separate consideration; do not assume that choosing `REPEATABLE READ` automatically prevents both in every database.

## Complete concurrency problems table

The table below describes common SQL isolation guarantees and typical behavior. Lost-update and write-skew columns require the implementation caveat shown below.

|Isolation level|Dirty read|Non-repeatable read|Phantom read|Lost update|Write skew|
|---|---|---|---|---|---|
|`READ UNCOMMITTED`|Possible|Possible|Possible|Possible|Possible|
|`READ COMMITTED`|Prevented|Possible|Possible|Possible|Possible|
|`REPEATABLE READ`|Prevented|Prevented|Possible*|Depends on implementation|Possible under snapshot isolation|
|`SERIALIZABLE`|Prevented|Prevented|Prevented|Prevented when transactions follow the isolation rules|Prevented|

* For standard SQL guarantees, `REPEATABLE READ` does not guarantee prevention of phantoms. A particular database may provide stronger behavior.

Important: This is a conceptual guide, not a universal prediction for every database engine. For example, SQL Server's lock-based `REPEATABLE READ` differs from snapshot-based `REPEATABLE READ` in PostgreSQL. Also, applications can still introduce logical lost-update problems if they read and overwrite stale values outside a correctly protected transaction.

## Memory Trick
 
- Dirty read = Read uncommitted data.
    
- Non-repeatable read = Same row, different value.
    
- Phantom read = Same query, different set of rows.
    
- Lost update = One change overwrites another.
    
- Write skew = Two individually valid decisions break a shared rule.


> [!Question] so why can't we use serializable directlly if it prevents in most of the scenarios?

Because stronger consistency can come at the cost of throughput, latency, and transaction failures. We usually want the strongest guarantees that the business operation actually needs, not necessarily the strongest available guarantees for every operation.

Let's understand with an example.

## 1. What happens when we use SERIALIZABLE everywhere?

Imagine your Spring Boot application has 100 users transferring money or reading account details at the same time.

Two transactions are updating the same account:

```
Initial balance = ₹1,000

T1                                      T2
-----------------------------------     --------------------------------
BEGIN SERIALIZABLE

Read balance = ₹1,000

                                        BEGIN SERIALIZABLE
                                        Read balance = ₹1,000

Update balance = ₹900

                                        Wants to update balance = ₹800
                                        Must wait, conflict, or abort
                                        depending on database behavior.

COMMIT

                                        May continue or retry
                                        after resolving the conflict.
```

The database must coordinate these transactions so that their combined execution is equivalent to a valid serial order.

Notice that T2 cannot necessarily proceed immediately. Depending on the database, it might wait for T1, or one transaction might be aborted and need to retry.

If many transactions conflict, this can cause:

- More waiting: transactions hold locks or wait for conflicting operations to finish.
    
- Higher latency: API requests take longer to complete.
    
- Lower throughput: fewer transactions may finish per second when contention is high.
    
- More retries: some databases abort transactions to preserve serializability.
    
- Deadlocks: transactions can wait on each other's locks, requiring the database to abort one of them.
    

These are possible costs, not guarantees that every serializable transaction will be slow. If transactions access unrelated data, they may still run concurrently.

## 2. Do all operations need SERIALIZABLE?

Consider three examples.

Example A: Read a user's profile

```
SELECT name, email
FROM users
WHERE user_id = 101;
```

`READ COMMITTED` is often sufficient. You generally don't need the strongest isolation level just to display a profile.

Example B: Deposit money

```
UPDATE account
SET balance = balance + 100
WHERE account_id = 1;
```

An atomic update prevents the classic lost-update problem for increments. You may not need `SERIALIZABLE` for this operation alone, although other account rules may require additional protection.

Example C: Reserve the last available seat

Two requests attempt to book the same final seat.

Here you need strong concurrency protection. A conditional atomic update, a unique constraint, or appropriate locking may solve the problem without requiring every transaction in the application to use `SERIALIZABLE`.

## 3. The important point: SERIALIZABLE is not always the best solution

Suppose your business rule is:

> At least one doctor must remain on call.

You could use `SERIALIZABLE` for the transaction that checks the roster and changes a doctor's availability. That is a valid approach.

Alternatively, you could lock a shared roster row before checking and updating the doctors. That coordinates the specific operations that need protection, rather than asking the database to enforce serializable execution across the entire transaction.

Similarly, for a seat booking, you could use a database constraint or an atomic conditional update.

The better solution depends on the invariant you need to protect and how the data is accessed.

## 4. How should you decide?

|Requirement|Typical starting point|
|---|---|
|Ordinary reads and writes|`READ COMMITTED`|
|Prevent stale JPA entity updates|`@Version` optimistic locking|
|Increment a counter or balance|Atomic SQL update|
|Ensure a row exists only once|Unique constraint|
|Protect a specific shared resource|Pessimistic lock or conditional update|
|Enforce complex rules across multiple rows|`SERIALIZABLE` or carefully designed coordination|

These are starting points, not universal prescriptions. For financial transactions, for example, atomic updates alone don't necessarily enforce every rule about overdrafts, transfers, or simultaneous withdrawals.

## 5. One final misconception to avoid

`SERIALIZABLE` does not mean the database executes every transaction completely one after another.

It means the database ensures the successful concurrent execution is equivalent to some valid serial ordering. Transactions can still overlap in time, and the database may block or abort some of them to maintain that guarantee.

My recommendation: Start with your database's usual isolation level, often `READ COMMITTED`, and explicitly protect the business rules that can be broken by concurrent requests. Use `SERIALIZABLE` when its stronger guarantees are needed and the performance trade-offs are acceptable.

That gives you correctness without unnecessarily restricting concurrency across the entire application.

### How Databases Implement Isolation

Databases use a few common techniques to enforce isolation:

- **Locks** make one transaction wait when another transaction is changing the same data.
- **MVCC** lets readers see a stable snapshot while writers create newer versions of rows.
- **Range locks** protect a group of rows, such as "all seats in this row" or "all products below this price."
- **Conflict detection** lets the database stop a transaction when it cannot safely run alongside another one.
