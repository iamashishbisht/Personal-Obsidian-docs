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

> [Question] 4. lets say if there is repeatable read, then in this case it can lock the db rows for reading right, no commits will be possible?

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