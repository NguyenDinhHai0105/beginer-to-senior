# ACID properties in RDBMS (Example: PostgreSQL)

ACID is the set of four properties that ensure correctness and reliability of transactions in a relational database management system (RDBMS): Atomicity, Consistency, Isolation, and Durability.

## 1. Atomicity
- Description: A transaction is executed as an indivisible unit: either it fully succeeds (COMMIT) or it is fully rolled back (ROLLBACK).
- PostgreSQL: Use BEGIN/COMMIT/ROLLBACK. SAVEPOINT is available to partially rollback within a transaction.

Example:
```sql
BEGIN;
INSERT INTO accounts(id, balance) VALUES (1, 100);
UPDATE accounts SET balance = balance - 50 WHERE id = 1;
-- If an error occurs:
-- ROLLBACK;
COMMIT;
```

## 2. Consistency
- Description: After a transaction, the database must satisfy all constraints, triggers, and business rules.
- PostgreSQL: Enforces constraints (PRIMARY KEY, FOREIGN KEY, UNIQUE, CHECK) and triggers at commit time.

Example (constraints):
```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email TEXT UNIQUE NOT NULL
);

CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  user_id INT REFERENCES users(id) ON DELETE CASCADE
);
```
Consistency ensures a transaction cannot commit if it violates these constraints.

## 3. Isolation
- Description: Concurrent transactions should not observe each other's intermediate states; results should be equivalent to some serial execution order.
- PostgreSQL: Supports isolation levels: READ UNCOMMITTED (name-only; effectively READ COMMITTED), READ COMMITTED (default), REPEATABLE READ, SERIALIZABLE.

Set isolation level example:
```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- or
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```
Note: SERIALIZABLE is the strongest level. PostgreSQL uses Serializable Snapshot Isolation and may abort transactions (serialization failure) requiring retries.

## 4. Durability
- Description: Once a transaction commits, its changes persist even after a system crash.
- PostgreSQL: Uses WAL (Write-Ahead Logging). Related settings: fsync, synchronous_commit, wal_level. WAL records changes before confirming commit to the client (depending on synchronous_commit).

Check or adjust settings:
```sql
SHOW wal_level;
SHOW synchronous_commit;
-- Configure in postgresql.conf or via ALTER SYSTEM, e.g.: synchronous_commit = on
```

## Practical combination
- Atomicity + Durability: COMMIT ensures data is written to WAL; WAL allows recovery after a crash.
- Consistency: Enforced by constraints and triggers to prevent invalid data at commit.
- Isolation: Choose a suitable level (READ COMMITTED for performance, SERIALIZABLE for maximum safety) and handle retries when necessary.

## Tips for PostgreSQL
- Enforce constraints at the DB level (not only in the application) to guarantee consistency.
- Use SAVEPOINT to rollback parts of a transaction when needed.
- Tune synchronous_commit according to durability vs performance requirements.
- When using SERIALIZABLE, be prepared to retry transactions after serialization failures.

---
This short document explains ACID concepts and how PostgreSQL implements each property, with SQL examples to try.
