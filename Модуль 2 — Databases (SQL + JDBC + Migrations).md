# Модуль 2 — Databases (SQL + JDBC + Migrations) — Questions 15–29

> Вопросы 15–29 из *Questions for Java Internship transfer.pdf* (SQL, транзакции, JDBC, миграции схемы). Ранее отдельным файлом: `database-questions-15-29.md` / `other/1 database-questions-15-29.md`.

## Оглавление

15. [Which JOIN types do you know?](#q15)
16. [What are Constraints in SQL?](#q16)
17. [What is a transaction?](#q17)
18. [What is ACID?](#q18)
19. [Which isolation levels do you know?](#q19)
20. [What is the difference between Function and Procedure?](#q20)
21. [What is the difference between Pessimistic and Optimistic locks?](#q21)
22. [What is the difference between Explain and Explain Analyze?](#q22)
23. [What are the Triggers in SQL?](#q23)
24. [What is the normalization process? Which normal forms do you know? What is denormalization? Why do we need to denormalize DB?](#q24)
25. [What do we need to do to execute a query using JDBC?](#q25)
26. [What types of statements in JDBC do you know? What is the difference between them?](#q26)
27. [Why do we need DB Migrations?](#q27)
28. [What DB Migration tools do you know?](#q28)
29. [What are the differences between Liquibase and Flyway?](#q29)

---

<a id="q15"></a>

## 15. Which JOIN types do you know?

### Question Restatement
What SQL `JOIN` types exist, how do they differ in which rows they keep when keys match or do not match, and when would you use each?

### 1. Definition
A **JOIN** combines rows from two or more tables (or derived tables) based on a related column—usually a **foreign key = primary key** condition in the `ON` clause.

### 2. Core Idea
JOINs answer: *for each row on the left/right, do we require a match on the other side, or do we keep non-matching rows with `NULL` placeholders?*  
That choice defines **INNER** vs **OUTER** vs **CROSS**.

### 3. JOIN Types (with behavior)

| JOIN | Keeps unmatched rows? | Typical use |
|------|------------------------|-------------|
| **INNER JOIN** | No — only rows where predicate matches | Default “related data only” |
| **LEFT [OUTER] JOIN** | Yes — all rows from **left**, right columns `NULL` if no match | “All customers, even without orders” |
| **RIGHT [OUTER] JOIN** | Yes — all rows from **right**, left columns `NULL` if no match | Same as LEFT if you swap tables (often rewritten) |
| **FULL [OUTER] JOIN** | Yes — all rows from **both** sides, `NULL` where no match | Reconciliation, rare in OLTP |
| **CROSS JOIN** | N/A — Cartesian product (every left × every row right) | Small dimension tables, combinatorial reports (dangerous if large) |

**Self-JOIN**: table joined to itself (aliases `t1`, `t2`) — still uses INNER/LEFT/etc.

**NATURAL JOIN** (know the name): joins on **all** same-named columns — easy to write, risky (schema changes break silently).

### 4. How Matching Works
- **`ON` condition** filters which pairs of rows combine.
- For OUTER joins, unmatched rows are preserved with `NULL`s for the non-preserved side.
- **Multiple predicates** in `ON` vs moving filters to `WHERE` can change results for OUTER joins (classic trap: filtering outer side in `WHERE` turns it into INNER behavior).

### 5. Performance
- Join order, indexes on join keys, and statistics matter.
- Missing index on FK/PK join columns → nested loops become expensive.
- Large `CROSS JOIN` → explosive row count.

### 6. Code Examples
```sql
-- INNER: only matching pairs
SELECT o.id, c.name
FROM orders o
INNER JOIN customers c ON o.customer_id = c.id;

-- LEFT: all orders even if customer missing (data issue)
SELECT o.id, c.name
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.id;
```

### 7. Common Pitfalls
- Confusing **LEFT** vs **RIGHT** (rename tables instead of memorizing “right”).
- Putting outer-table filter in `WHERE` so outer join effectively becomes inner.
- Duplicate rows from **one-to-many** joins without `DISTINCT` or aggregation when you expected one row per parent.

### 8. Interview Deep Questions
- Difference INNER vs LEFT with example?
- Why is `CROSS JOIN` dangerous?
- `ON` vs `WHERE` for LEFT JOIN?
- How does DB pick join algorithm (hash / merge / nested loops) at high level?

### 9. Summary (Short speaking answer)
SQL provides **INNER** (matches only), **LEFT/RIGHT/FULL OUTER** (keep non-matching side(s) with `NULL`s), and **CROSS** (Cartesian product). In practice **INNER** and **LEFT** are most common. **RIGHT** is often rewritten as **LEFT** by swapping tables. **FULL** is rarer. **Self-joins** reuse the same table with aliases. Always mind **one-to-many** duplication and **predicate placement** with outer joins.

**SHORT (30 sec answer)**  
**INNER JOIN** returns only matching rows. **LEFT/RIGHT/FULL OUTER** keep rows from one or both tables when there is no match, filling the other side with `NULL`. **CROSS JOIN** is every combination of rows. Most day-to-day work is **INNER** and **LEFT JOIN**.

<a id="q16"></a>

## 16. What are Constraints in SQL?

### Question Restatement
What are **constraints** in relational databases, which types exist, and what do they guarantee at the table/column level?

### 1. Definition
**Constraints** are declarative rules enforced by the database so data stays **valid** and **relationships** stay consistent. Violations cause the statement to **fail** (or be rejected), depending on DB settings.

### 2. Core Idea
They move correctness from application-only checks into the database: **single source of truth** for rules that must always hold, even if multiple apps write data.

### 3. Types of Constraints

| Constraint | What it enforces |
|-------------|------------------|
| **PRIMARY KEY (PK)** | Uniqueness + **NOT NULL**; one PK per table (typically); clustered index on many DBs. |
| **FOREIGN KEY (FK)** | Referential integrity: values in child column(s) must exist in parent PK/unique key (or NULL if allowed). |
| **UNIQUE** | No duplicate values in column(s); allows multiple `NULL` in some DBs (e.g. PostgreSQL), not in others (e.g. older SQL Server rules—know it varies). |
| **NOT NULL** | Column cannot store `NULL`. |
| **CHECK** | Boolean expression per row (e.g. `age >= 0`). |
| **DEFAULT** | Value inserted when column omitted (not always classified as “constraint” in all docs, but related). |

**Composite constraints**: PK/UNIQUE/FK on **multiple columns** together.

### 4. How They Work Internally (high level)
- DB maintains **indexes** for PK/UNIQUE to enforce uniqueness quickly.
- FK checks use indexes on parent PK and often index on child FK column.
- On `INSERT`/`UPDATE`/`DELETE`, engine validates constraints before committing row changes (timing of FK checks can have modes like `DEFERRABLE` in PostgreSQL—advanced).

### 5. Practical Usage
- PK on every table (surrogate key or natural key).
- FK to model relations and optional `ON DELETE CASCADE/RESTRICT`.
- CHECK for domain rules (non-negative price).

### 6. Code Examples
```sql
CREATE TABLE customers (
    id          BIGSERIAL PRIMARY KEY,
    email       VARCHAR(255) NOT NULL UNIQUE,
    age         INT CHECK (age >= 0)
);

CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL REFERENCES customers(id)
);
```

### 7. Common Pitfalls
- Missing FK → orphan rows in application data.
- Overusing `CHECK` for cross-row rules (harder—often triggers or app logic).
- Assuming `UNIQUE` allows only one `NULL` everywhere (DB-specific).

### 8. Interview Deep Questions
- PK vs UNIQUE?
- What happens on delete of parent row if FK exists?
- Can CHECK reference other tables? (Usually no in standard SQL—triggers instead.)

### 9. Summary (Short speaking answer)
Constraints are database-enforced rules: **PRIMARY KEY** identifies rows uniquely, **FOREIGN KEY** preserves references between tables, **UNIQUE** forbids duplicates, **NOT NULL** forbids missing values, and **CHECK** validates domain conditions. They protect data integrity at the engine level and complement application validation. Understanding FK delete/update actions (`CASCADE`, `RESTRICT`) is essential for correct schema design.

**SHORT (30 sec answer)**  
Constraints are rules the DB enforces: **PRIMARY KEY** (unique, not null), **FOREIGN KEY** (valid references), **UNIQUE**, **NOT NULL**, and **CHECK** (row-level conditions). They keep data consistent even when many clients write to the database.

<a id="q17"></a>

## 17. What is a transaction?

### Question Restatement
What is a **database transaction**, how do **commit** and **rollback** work, and why do we group operations into transactions?

### 1. Definition
A **transaction** is a **logical unit of work** executed as a single atomic step from the database’s perspective: either **all** changes succeed (**COMMIT**) or **none** do (**ROLLBACK**).

### 2. Core Idea
Transactions bundle multiple SQL statements so the database never ends up in a **half-updated** state (e.g., money debited but not credited).

### 3. How It Works Internally (conceptual)
- Transaction gets a **start** boundary (`BEGIN` / implicit in some APIs).
- Changes go to the transaction’s **private view** (often via undo/redo logs) until commit.
- **COMMIT** makes changes **durable** and visible to other transactions (subject to isolation rules).
- **ROLLBACK** discards uncommitted work using undo information.
- **ACID** properties formalize guarantees (see Q18).

### 4. Isolation and Concurrency
- Concurrent transactions interleave; isolation level defines what anomalies are allowed (Q19).
- Locks or MVCC implement isolation depending on DB engine.

### 5. APIs in Applications
- JDBC: `setAutoCommit(false)`, `commit()`, `rollback()`.
- Spring `@Transactional`: declarative transaction boundaries.

### 6. Code Examples (SQL)
```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

```sql
BEGIN;
DELETE FROM orders WHERE id = 123;
-- oops
ROLLBACK;
```

### 7. Common Pitfalls
- Long-running transactions holding locks → blocking and deadlocks.
- Forgetting commit in autocommit-off code.
- Mixing business logic with huge transactions.

### 8. Interview Deep Questions
- Difference implicit vs explicit transaction?
- What is savepoint?
- How does rollback interact with exceptions in Spring?

### 9. Summary (Short speaking answer)
A transaction groups database operations into one atomic unit: **commit** persists all changes; **rollback** cancels them. This prevents partial updates and is the foundation for reliable business workflows. Implementation uses logging and concurrency control. Application servers map transactions to connection boundaries and frameworks like Spring manage them declaratively.

**SHORT (30 sec answer)**  
A transaction is a sequence of SQL operations treated as **one atomic unit**: **COMMIT** applies everything, **ROLLBACK** undoes everything since the start. It prevents inconsistent half-finished updates and works together with isolation and durability guarantees.

<a id="q18"></a>

## 18. What is ACID?

### Question Restatement
What does **ACID** mean for databases, and how does each letter apply to real transaction behavior?

### 1. Definition
**ACID** is a mnemonic for four guarantees of **reliable transactions**: **Atomicity**, **Consistency**, **Isolation**, **Durability**.

### 2. Each Property Explained

| Letter | Meaning | Practical meaning |
|--------|---------|-----------------|
| **A — Atomicity** | All-or-nothing | Either every statement in the transaction commits, or none do. Crash between steps → rollback. |
| **C — Consistency** | Valid state → valid state | All **constraints**, triggers, and rules remain satisfied after commit. (Also “business invariants” in broader sense.) |
| **I — Isolation** | Concurrent transactions don’t corrupt each other’s reads/writes | Degree depends on **isolation level** (Q19): may allow some anomalies for performance. |
| **D — Durability** | After commit, survives crashes | Typically via **write-ahead log (WAL)**: log persisted before commit ack. |

### 3. Why It Matters
Without ACID, distributed money transfer, inventory, and booking systems would corrupt under failures and concurrency.

### 4. Trade-offs
- Stronger isolation → fewer anomalies, often more locking / lower throughput.
- Durability → disk/fsync cost.

### 5. Common Pitfalls
- Confusing **Consistency** with only “application validation” — DB constraints are part of it.
- Thinking **Isolation** means “no bugs ever” — levels trade off vs performance.

### 6. Interview Deep Questions
- Give an example violating each property.
- Which property does WAL primarily support?
- Isolation vs serializability?

### 7. Summary (Short speaking answer)
**Atomicity** ensures all-or-nothing execution. **Consistency** ensures committed data obeys all database rules and invariants. **Isolation** controls how concurrent transactions see each other’s in-progress work. **Durability** ensures committed transactions survive crashes through logging and storage policies. Together they define trustworthy transactional processing.

**SHORT (30 sec answer)**  
**ACID**: **Atomicity** (all-or-nothing), **Consistency** (valid data after commit), **Isolation** (controlled concurrency visibility), **Durability** (committed data survives crashes). These are the standard guarantees of transactional databases.

<a id="q19"></a>

## 19. Which isolation levels do you know?

### Question Restatement
What **transaction isolation levels** exist in SQL, what anomalies can each allow, and what do they mean in practice?

### 1. Definition
**Isolation** defines how much one transaction’s uncommitted or concurrent changes are visible to another. Weaker isolation improves performance but allows **anomalies**.

### 2. Standard SQL Isolation Levels (ANSI)

| Level | Dirty read | Non-repeatable read | Phantom read |
|-------|------------|---------------------|--------------|
| **READ UNCOMMITTED** | Possible | Possible | Possible |
| **READ COMMITTED** | Not | Possible | Possible |
| **REPEATABLE READ** | Not | Not | Possible* |
| **SERIALIZABLE** | Not | Not | Not |

\* **Phantom reads**: In PostgreSQL `REPEATABLE READ` uses snapshot isolation and prevents phantoms for many workloads; exact behavior is **DB-specific** — important interview honesty.

### 3. Anomalies Explained
- **Dirty read**: read another transaction’s **uncommitted** change (rolled back later → bad data read).
- **Non-repeatable read**: same row read twice, **values differ** because another transaction committed an update between reads.
- **Phantom read**: same query twice, **different set of rows** (inserts/deletes committed by others).

### 4. Real Engines
- **Oracle** default is often **READ COMMITTED** (statement-level snapshot).
- **PostgreSQL** default **READ COMMITTED**; **REPEATABLE READ** is snapshot-based.
- **MySQL InnoDB** default **REPEATABLE READ** with next-key locking reducing phantoms for locking reads.
- **SQL Server** offers all four.

### 5. Choosing a Level
- Most OLTP apps use **READ COMMITTED** + correct SQL.
- Use **SERIALIZABLE** when correctness under concurrency is critical and contention is acceptable (or use explicit locking).

### 6. Common Pitfalls
- Assuming identical behavior across databases at the same named level.
- Fixing race conditions only in Java while DB is READ COMMITTED without idempotent constraints.

### 7. Interview Deep Questions
- What is snapshot isolation?
- Difference **REPEATABLE READ** MySQL vs PostgreSQL?
- How to prevent lost updates?

### 8. Summary (Short speaking answer)
Isolation levels trade **correctness vs performance**. **READ UNCOMMITTED** allows dirty reads (rare in production). **READ COMMITTED** prevents dirty reads but allows non-repeatable and phantom reads. **REPEATABLE READ** prevents non-repeatable reads for the snapshot/locking model of the engine. **SERIALIZABLE** provides the strongest isolation and avoids these anomalies (may serialize transactions). Always note **implementation differs by database**.

**SHORT (30 sec answer)**  
The classic levels are **READ UNCOMMITTED**, **READ COMMITTED**, **REPEATABLE READ**, and **SERIALIZABLE**. Higher levels prevent more anomalies (dirty / non-repeatable / phantom reads) but may reduce concurrency. Real databases implement them differently (MVCC vs locks), so names are not always identical behavior.

<a id="q20"></a>

## 20. What is the difference between Function and Procedure?

### Question Restatement
In databases (and sometimes in general programming), what is the difference between a **function** and a **stored procedure**?

### 1. Definition (SQL / stored routines — typical interview focus)
- **Function (stored function)**: named routine that **returns a value** (scalar or table in some DBs) and is designed to be used **inside expressions** (e.g. `SELECT`, `WHERE`) when the DB allows it.
- **Procedure (stored procedure)**: named routine that usually **performs actions** (`INSERT`/`UPDATE`/DDL in some systems) and may return data via **OUT parameters** or result sets, depending on the engine.

### 2. Core Differences

| Aspect | Function | Procedure |
|--------|----------|-----------|
| **Return value** | Yes — `RETURN` value / result | Often “no scalar return”; may use `OUT` params / result sets |
| **Use in SQL expressions** | Often allowed (`SELECT fn()`) | Usually called with `CALL` / `EXEC` |
| **Side effects** | Many DBs **restrict** non-read-only side effects in functions (portability varies) | Full procedural logic, multiple statements, transactions sometimes controlled inside |
| **Transaction control** | Often cannot `COMMIT` inside (DB-specific) | May allow transaction control in some engines |

### 3. Examples (PostgreSQL-style illustration)
```sql
CREATE FUNCTION add_one(x INT) RETURNS INT AS $$
  SELECT x + 1;
$$ LANGUAGE sql;

CREATE PROCEDURE archive_old_orders() AS $$
BEGIN
  INSERT INTO orders_archive SELECT * FROM orders WHERE created_at < now() - interval '1 year';
  DELETE FROM orders WHERE created_at < now() - interval '1 year';
END;
$$ LANGUAGE plpgsql;
```

### 4. “Function vs Procedure” in Programming Languages (if interviewer broadens)
- **Function**: returns a value, ideally pure in FP discussions.
- **Procedure** (subroutine): sequence of steps; may return nothing (`void`).

### 5. Common Pitfalls
- Porting procedures between Oracle / SQL Server / PostgreSQL — syntax and capabilities differ.
- Using functions for heavy side effects where procedures are clearer.

### 6. Interview Deep Questions
- Can a function call a procedure and vice versa?
- Why limit side effects in functions?
- Difference `RETURNS TABLE` function vs procedure returning cursor?

### 7. Summary (Short speaking answer)
In SQL stored routines, a **function** primarily **returns a value** and is often usable inside queries; engines often restrict side effects for predictability. A **procedure** is a **procedural block** invoked with `CALL`/`EXEC`, suited to multi-step work, side effects, and sometimes transaction control—behavior is **database-specific**, so naming the engine matters in a deep interview.

**SHORT (30 sec answer)**  
A **function** returns a value and is usually meant for use in expressions (e.g. in `SELECT`). A **procedure** is a stored routine focused on executing one or more statements (often via `CALL`), with `OUT` parameters or result sets instead of a single return value. Exact rules differ by database (especially side effects and transactions inside functions).


<a id="q21"></a>

## 21. What is the difference between Pessimistic and Optimistic locks?

### Question Restatement
In databases (and sometimes in JPA), what is the difference between **pessimistic** and **optimistic** locking, and when do we use each?

### 1. Definition
- **Pessimistic locking**: the database (or API) **locks rows** (shared or exclusive) so other transactions **cannot change** them until you commit/rollback. Example: `SELECT ... FOR UPDATE` (PostgreSQL/MySQL), `UPDLOCK` (SQL Server).
- **Optimistic locking**: no DB lock while you read; before write you check that the row **has not changed** (usually via **version column** or comparing all fields). If it changed, update fails and you **retry** or report conflict.

### 2. Core Idea
- Pessimistic: assume **conflicts will happen** → block others early.
- Optimistic: assume **conflicts are rare** → cheaper reads, fail fast on write if data drifted.

### 3. How It Works Internally
- **Pessimistic**: row/page/table locks held per isolation rules; can cause **deadlocks** and **blocking** under contention.
- **Optimistic**: `UPDATE ... WHERE id = ? AND version = ?` then check `rowsAffected == 1`; else someone else incremented `version`.

### 4. Performance & Usage
| Aspect | Pessimistic | Optimistic |
|--------|-------------|------------|
| Contention | Blocks readers/writers | No lock during read |
| Throughput | Lower under high contention | Higher when conflicts rare |
| Lost updates | Prevented by lock | Prevented by version check |
| Typical use | Financial critical sections, low-contention hot rows | Web CRUD, high read / low write conflict |

### 5. JPA Note
`@Lock(LockModeType.PESSIMISTIC_WRITE)` vs `@Version` field (optimistic).

### 6. Summary (Short speaking answer)
**Pessimistic** locking acquires database locks so nobody else can modify rows until the transaction ends—strong guarantees but risk of blocking and deadlocks. **Optimistic** locking uses a version (or checksum) and only fails the update if data changed, which scales better when conflicts are uncommon. Choose pessimistic when conflicts are likely or business rules require exclusive access; optimistic for many concurrent reads and rare concurrent writes to the same row.

**SHORT (30 sec answer)**  
**Pessimistic** = lock rows in DB (`FOR UPDATE`) so others wait. **Optimistic** = no lock on read; on save, check **version**—if it changed, reject/retry. Pessimistic is safer under heavy conflict; optimistic is faster when conflicts are rare.

---

<a id="q22"></a>

## 22. What is the difference between Explain and Explain Analyze?

### Question Restatement
What is the difference between `EXPLAIN` and `EXPLAIN ANALYZE` (PostgreSQL naming; other DBs have similar concepts)?

### 1. Definition
- **`EXPLAIN`**: shows the **query plan** the optimizer **intends** to use—join order, index usage, estimated rows/cost. Does **not** execute the full query (in PostgreSQL `EXPLAIN` alone does not run the query; in some tools “Explain” may still execute depending on dialect—know your DB).
- **`EXPLAIN ANALYZE`**: **executes** the query and returns **actual** timings, rows, and sometimes buffer stats—**real** behavior, not only estimates.

### 2. Core Idea
`EXPLAIN` = cheap, prediction.  
`EXPLAIN ANALYZE` = real measurement (can be heavy on production—use with care).

### 3. Interview Trap
- MySQL: `EXPLAIN` vs `EXPLAIN ANALYZE` (8.0.18+) semantics differ slightly from PostgreSQL.
- SQL Server: **Estimated** vs **Actual** execution plans (similar idea).

### 4. Summary (Short speaking answer)
`EXPLAIN` shows the **planned** execution path and cost estimates. `EXPLAIN ANALYZE` **runs** the statement and shows **actual** row counts and timings, revealing mis-estimates and real bottlenecks. Use `EXPLAIN` for quick checks; use `EXPLAIN ANALYZE` in dev/staging or off-peak to validate performance without guessing.

**SHORT (30 sec answer)**  
`EXPLAIN` = **plan + estimates** (optimizer’s prediction). `EXPLAIN ANALYZE` = **runs the query** and shows **actual** time and rows. Use the second to find real slow steps; avoid running heavy `ANALYZE` blindly on production.

---

<a id="q23"></a>

## 23. What are the Triggers in SQL?

### Question Restatement
What is a **trigger** in SQL, when does it fire, and what are pros and cons?

### 1. Definition
A **trigger** is a stored procedure **automatically executed** by the DB when an event occurs: `BEFORE`/`AFTER` `INSERT`, `UPDATE`, `DELETE` on a table (sometimes `INSTEAD OF` on views in some engines).

### 2. Core Idea
Centralize cross-cutting rules: audit logging, derived columns, referential-style checks, replication hooks—**inside the database**.

### 3. Example (conceptual)
```sql
CREATE TRIGGER trg_orders_ai
AFTER INSERT ON orders
FOR EACH ROW
BEGIN
  INSERT INTO audit_log(table_name, row_id, action) VALUES ('orders', NEW.id, 'INSERT');
END;
```

### 4. Pitfalls
- Hidden side effects (harder to debug than app code).
- Performance: fire on every row in bulk loads.
- Testing and migrations become more complex.

### 5. Summary (Short speaking answer)
Triggers are **event-driven** routines attached to DML operations. They enforce or automate behavior at the data layer but can obscure logic and hurt bulk operation performance. Many teams prefer explicit application or outbox patterns for complex workflows, but triggers remain valid for auditing and simple invariants.

**SHORT (30 sec answer)**  
A **trigger** runs **automatically** on `INSERT`/`UPDATE`/`DELETE` (before or after). Useful for **audit**, defaults, or enforcing rules in one place. Downsides: **hidden logic**, harder testing, performance on **bulk** operations.

---

<a id="q24"></a>

## 24. What is the normalization process? Which normal forms do you know? What is denormalization? Why do we need to denormalize DB?

### Question Restatement
Explain **normalization**, common **normal forms**, **denormalization**, and why we sometimes **denormalize** on purpose.

### 1. Normalization
Process of organizing tables to **reduce redundancy** and **anomalies** (insert/update/delete anomalies) by splitting data and linking with keys.

### 2. Normal forms (interview checklist)
- **1NF**: atomic column values (no repeating groups in a single cell).
- **2NF**: 1NF + no **partial** dependency of non-key attributes on part of a composite key.
- **3NF**: 2NF + no **transitive** dependency (non-key columns depend only on key, not on other non-key columns).
- **BCNF** (Boyce-Codd): stricter variant of 3NF for tricky cases.
- Higher forms (4NF, 5NF) appear in advanced design—know names briefly.

### 3. Denormalization
**Deliberately** introducing redundancy (e.g. storing `customer_name` on `orders`) to **speed up reads** or simplify queries.

### 4. Why denormalize?
- **Read performance**: fewer joins, precomputed aggregates.
- **Reporting / analytics** workloads.
- **Trade-off**: faster SELECT, harder UPDATE (must keep copies consistent), more storage.

### 5. Summary (Short speaking answer)
**Normalization** structures data to minimize redundancy and anomalies; common targets are **1NF, 2NF, 3NF**, and **BCNF**. **Denormalization** adds controlled redundancy to optimize read-heavy or reporting queries at the cost of update complexity and consistency discipline. In microservices, denormalized read models are common in CQRS/event-driven designs.

**SHORT (30 sec answer)**  
**Normalization** splits tables to remove redundancy (1NF → 2NF → 3NF/BCNF). **Denormalization** duplicates data on purpose for **faster reads** or simpler queries, accepting harder updates and consistency work.

---

<a id="q25"></a>

## 25. What do we need to do to execute a query using JDBC?

### Question Restatement
What steps does JDBC take from Java code to run a SQL query?

### 1. Steps (typical flow)
1. **Load JDBC driver** (often automatic via `DriverManager` / SPI in modern JDBC).
2. **Obtain `Connection`** — URL, user, password (`DriverManager.getConnection` or `DataSource` from pool in production).
3. **Create `Statement`, `PreparedStatement`, or `CallableStatement`** with SQL.
4. For **`PreparedStatement`**: bind parameters (`setInt`, `setString`, …) — **always** for user input (SQL injection safety).
5. **Execute**: `executeQuery` (SELECT), `executeUpdate` (INSERT/UPDATE/DELETE), or `execute` for either/multiple results.
6. **Process `ResultSet`** (iterate rows, read columns).
7. **Close** in `finally` or **try-with-resources**: `ResultSet` → `Statement` → `Connection` (pooled connections return to pool, not really “closed”).

### 2. Modern practice
Use **`DataSource`** (HikariCP), **`PreparedStatement`**, try-with-resources, and avoid string concatenation for SQL.

### 3. Summary (Short speaking answer)
JDBC requires a **driver**, a **connection**, a **statement** with bound parameters, **execution**, iterating **`ResultSet`**, and **closing** resources (prefer try-with-resources). Production apps use **connection pools** and `PreparedStatement` for security and performance.

**SHORT (30 sec answer)**  
Load driver → get **Connection** (usually from a **pool**) → create **PreparedStatement** with SQL → **bind** parameters → **executeQuery** / **executeUpdate** → read **ResultSet** → **close** resources (try-with-resources).

---

<a id="q26"></a>

## 26. What types of statements in JDBC do you know? What is the difference between them?

### Question Restatement
What JDBC statement types exist and how do they differ?

### 1. Types
| Type | Use |
|------|-----|
| **`Statement`** | Static SQL **without** parameters (rarely appropriate for dynamic values). |
| **`PreparedStatement`** | **Parameterized** SQL (`?` or named); **precompiled** by DB, **bind** values safely. |
| **`CallableStatement`** | Calls **stored procedures/functions**; extends `PreparedStatement`. |

### 2. Differences
- **`Statement`**: SQL injection risk if concatenating strings; no parameter binding.
- **`PreparedStatement`**: **best default** for CRUD; plan caching; binds types correctly.
- **`CallableStatement`**: `{call procName(?,?)}` style for procedures.

### 3. Summary (Short speaking answer)
Use **`PreparedStatement`** for almost all application SQL with bound parameters. Use **`CallableStatement`** for stored procedures. Avoid raw **`Statement`** for user-influenced SQL. This improves security (no injection), clarity, and often performance through plan reuse.

**SHORT (30 sec answer)**  
**Statement** = raw SQL (avoid for dynamic input). **PreparedStatement** = **parameters**, safe and efficient. **CallableStatement** = **stored procedures**. Prefer **PreparedStatement** by default.

---

<a id="q27"></a>

## 27. Why do we need DB Migrations?

### Question Restatement
Why do teams use database migrations instead of applying SQL manually?

### 1. Core reasons
- **Version control** for schema: same history as code (Git).
- **Reproducible** environments: dev/staging/prod get the **same** DDL in order.
- **Automation** in CI/CD: deploy app + schema together.
- **Audit trail**: who changed what, when.
- **Rollback strategy** (where supported) or forward-only fixes with discipline.

### 2. Without migrations
Manual `ALTER` on prod → drift, human error, untraceable state.

### 3. Summary (Short speaking answer)
Migrations turn database schema changes into **repeatable, ordered, reviewable** scripts so every environment stays aligned and deployments are automated and auditable. They are essential for team workflows and safe releases.

**SHORT (30 sec answer)**  
Migrations **version** the schema like code: same ordered changes everywhere, **CI/CD** friendly, **reviewable**, and they prevent **manual drift** between environments.

---

<a id="q28"></a>

## 28. What DB Migration tools do you know?

### Question Restatement
Which tools are commonly used for database migrations in Java ecosystems?

### 1. Common tools
- **Flyway** — SQL-first, simple versioning (`V1__`, `R__` repeatable).
- **Liquibase** — XML/YAML/JSON/SQL **changelog** model, supports complex refactorings.
- **Hibernate `ddl-auto`** — dev only; **not** a replacement for controlled migrations in production.
- Others: **Atlas**, **db-migrate**, vendor tools (Redgate, etc.).

### 2. Summary (Short speaking answer)
In Java shops, **Flyway** and **Liquibase** dominate. Both track applied changes and run pending scripts in order. Hibernate schema generation is useful for prototyping but production schema evolution should use a dedicated migration tool.

**SHORT (30 sec answer)**  
Best-known: **Flyway** and **Liquibase**. Both apply versioned scripts to keep databases in sync. Avoid relying on Hibernate **`ddl-auto`** for production schema evolution.

---

<a id="q29"></a>

## 29. What are the differences between Liquibase and Flyway?

### Question Restatement
How does **Liquibase** differ from **Flyway**?

### 1. Comparison

| Aspect | Flyway | Liquibase |
|--------|--------|-----------|
| **Authoring** | Primarily **SQL files** with naming convention | **Changelog** (XML/YAML/JSON/SQL) with changesets |
| **Style** | Minimal, convention-based | More **metadata** and abstractions (preconditions, rollback tags) |
| **Flexibility** | Simple pipeline | Rich for complex enterprise rules |
| **Learning curve** | Lower for SQL-first teams | Steeper if using XML abstractions |

### 2. When to pick
- **Flyway**: teams want **plain SQL** files in repo, simple CI.
- **Liquibase**: need **structured** changelogs, cross-database abstractions, or complex change orchestration.

### 3. Summary (Short speaking answer)
**Flyway** is lean and SQL-centric: versioned scripts by filename. **Liquibase** uses explicit **changesets** in changelog files with more features for preconditions and multi-DB support. Both solve the same problem—controlled, repeatable migrations—choose based on team taste and complexity needs.

**SHORT (30 sec answer)**  
**Flyway** = simple **versioned SQL** files. **Liquibase** = **changelog** (often XML/YAML) with **changesets** and more enterprise features. Same goal: migrate DB safely and repeatably; Flyway is lighter, Liquibase more structured.

