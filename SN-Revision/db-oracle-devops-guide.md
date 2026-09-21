# Database / Oracle Troubleshooting — DevOps Interview Guide

> Goal: not to think like a DBA, but to answer *"why is the app unable to get data, or why is the DB slow/failing?"*

```
Application → Connection → Database → SQL Query → Tables/Data
```

---

## Level 1 — Basics

### 1. What is a Database?
Stores application data in an organized way (tables, rows, columns).

```
CUSTOMER            ORDERS
---------            ---------
customer_id          order_id
name                 customer_id
email                amount, status
```

**Interview answer:** A database is a system used to persist, organize and retrieve application data. Oracle is an RDBMS that stores data in tables and uses SQL to access and manipulate it.

### 2. What is SQL?
Structured Query Language — how you talk to the DB.
```sql
SELECT * FROM employees;
INSERT INTO employees VALUES (101, 'John');
UPDATE employees SET name = 'David' WHERE id = 101;
DELETE FROM employees WHERE id = 101;
```
For DevOps interviews: focus on SELECT, UPDATE, transactions, locks, sessions — not deep query writing.

### 3. What is a Database Session?
Represents a connection between a client/app and the DB. Each session runs SQL and participates in transactions.
```
Application
 ├── Session 101
 ├── Session 102
 └── Session 103
```

### 4. What is a Transaction?
A logical unit of work — all-or-nothing.
```
Transfer ₹1000: A -₹1000, B +₹1000 → must both succeed or both rollback
```
- `COMMIT;` → makes changes permanent
- `ROLLBACK;` → undoes uncommitted changes

---

## Level 2 — Troubleshooting

### 5. What is a Lock?
Session 1 updates a row without committing → holds a lock. Session 2 wanting the same row must wait.
```
Session 1 → holds lock → Row 101
Session 2 → wants same row → WAITING
```

### 6. What is a Blocking Session?
A session holding a lock another session needs.
```
Session 101 (BLOCKER) → holds lock → Table/row ← needs lock ← Session 102 (BLOCKED)
```
Effect: user request → app → DB query → waiting for lock → timeout → app error.

### 7. Blocking vs Deadlock
- **Blocking:** A holds a lock, B waits. Resolves once A commits/rolls back.
- **Deadlock:** A holds Resource1, needs Resource2. B holds Resource2, needs Resource1 → circular wait, neither can proceed.

**Interview answer:** Blocking occurs when one session holds a resource required by another. A deadlock occurs when two or more sessions wait on resources held by each other, creating a circular dependency.

### 8. What is a Stuck Session?
A long-running, uncommitted transaction holding locks → other sessions pile up waiting → application appears stuck.

### 9. As a DevOps Engineer, How Do You Troubleshoot? *(memorize this)*
```
1. Check application symptoms
2. Check DB connectivity
3. Check active sessions
4. Check blocking sessions
5. Identify the SQL/query
6. Identify the blocker
7. Check transaction/lock details
8. Coordinate with DBA/application owner
9. Take approved action
10. Monitor recovery
```

### 10. Finding Sessions in Oracle
```sql
SELECT sid, serial#, username, status
FROM v$session;
```
- **SID** — Session ID
- **SERIAL#** — used with SID to uniquely identify a session for operations like kill

### 11. Finding Blocking Sessions
```sql
SELECT sid, serial#, blocking_session, status, event
FROM v$session
WHERE blocking_session IS NOT NULL;
```
```
SID 102 → blocked by 101
SID 103 → blocked by 101
```
Session 101 becomes the one to investigate.

### 12. What Do You Check on the Blocker?
```sql
SELECT sid, serial#, username, machine, program
FROM v$session WHERE sid = 101;

SELECT sql_id, sql_text FROM v$sql WHERE sql_id = '<SQL_ID>';
```
Ask: how long running? what's it waiting on? uncommitted transaction? expected app behavior? known service? matching incident?

### 13. Should You Kill the Session?
**Not automatically.** This is a key interview trap.

Don't say: *"I find the blocking session and kill it."*

Say instead: *"I first identify the blocker, understand the transaction and impact, and coordinate with the DBA/application owner. If confirmed stale and approved, the DBA terminates it."*

Why: killing a prod session can cause rollback storms, app errors, failed transactions, extra load, business impact.

### 14. What Does the DBA Do?
```
DevOps → detects symptom → identifies DB-related issue → provides evidence → DBA investigates internals → DBA takes DB-level action
```
Strong answer: *"SID 101 has held a lock for 30 minutes, blocking 15 sessions. Here's the SQL ID and connection info."*
Weak answer: *"Database is stuck, please check."*

### 15. If You Don't Know the Exact Query
Don't say "I don't know." Don't just repeat the question.

Say: *"Are you asking me to identify the blocking session, or to resolve the lock itself?"* — then answer based on clarification.

Fallback: *"I'd first identify the affected sessions and blocking relationship, collect SID, SQL ID, wait event and transaction info, then work with the DBA for the DB-level action."*

### 16. Application Suddenly Slow — How Do You Troubleshoot?
Think in layers:
```
User → Application → Network → DB connection → Database → SQL query → Disk/CPU/Memory
```
Check:
- **App:** error rate, response time, connection pool, timeouts
- **DB:** CPU, memory, active/waiting/blocking sessions, long-running queries, wait events
- **SQL:** slow query, missing/bad index, full table scan, poor execution plan

---

## Level 3 — Performance

### 17. What is an Index?
Like a book index — instead of scanning page by page, jump directly to the row.
```sql
SELECT * FROM employees WHERE employee_id = 1001;
```
An index on `employee_id` speeds this up — but indexes cost storage, memory, and slow down INSERT/UPDATE/DELETE. Don't blindly say "add an index."

### 18. What is a Long-Running Query?
Expected 100ms, actual 10 minutes. Causes: large dataset, poor SQL, missing/bad index, bad execution plan, lock contention, resource contention, network/app behavior. Don't assume the SQL itself is bad first.

### 19. What is an Execution Plan?
Shows how Oracle's optimizer intends to run a query — index access vs full table scan, join method, etc.

**Interview answer:** The execution plan shows how the database optimizer intends to execute a SQL statement, including operations such as index access, table scans and joins.

---

## Cheat Sheet — Full Troubleshooting Flow
```
APPLICATION ISSUE
  → Is DB connectivity working?
  → Check active sessions
  → Check blocking/waiting sessions
  → Identify SID + SERIAL#
  → Identify SQL_ID / query
  → Check wait event
  → Check transaction/locks
  → Check DB resource utilization
  → Coordinate with DBA
  → Take approved action
  → Monitor recovery
```

## 10 Terms to Know Cold

| Term | Meaning |
|---|---|
| Database | Stores application data |
| SQL | Language used to interact with DB |
| Session | Connection between client and DB |
| Transaction | Logical unit of database work |
| COMMIT | Make transaction changes permanent |
| ROLLBACK | Undo uncommitted changes |
| Lock | Controls concurrent access to data |
| Blocking | One session waits for another |
| Deadlock | Sessions wait on each other cyclically |
| Index | Data structure that speeds up lookups |

---

## Level 4 — Real Interview Scenarios (practice these out loud)

1. "Production DB session is stuck — walk me through it."
2. "Application is unable to connect to Oracle — what do you check?"
3. "Queries suddenly became slow — how do you approach it?"
4. "One query is blocking 50 sessions — what's your action?"
5. "DB CPU is at 100% — where do you look?"
6. "Connection pool is exhausted — what's happening and what do you check?"
7. "Application is getting a DB timeout — how do you isolate app vs DB vs network?"
8. "How do you work with the DBA during an incident?"

---

## Extra: SQL / RDBMS Basics (fill the gaps)

1. RDBMS vs NoSQL — structured schema + relationships vs flexible/unstructured.
2. Primary key vs foreign key — PK uniquely identifies a row; FK links to another table's PK.
3. Normalization — organizing tables to reduce redundancy (1NF: atomic columns, 2NF: no partial dependency, 3NF: no transitive dependency).
4. DELETE vs TRUNCATE vs DROP — DELETE removes rows (can rollback, triggers fire), TRUNCATE removes all rows fast (minimal logging, no rollback in most DBs), DROP removes the whole table structure.
5. ACID — Atomicity, Consistency, Isolation, Durability.
6. Clustered vs non-clustered index — clustered physically orders table data by the key (one per table); non-clustered is a separate lookup structure pointing to rows (many allowed).
7. JOIN types — INNER (matching rows only), LEFT (all left + matches), RIGHT (all right + matches), FULL (all rows, matched where possible).
8. WHERE vs HAVING — WHERE filters rows before grouping, HAVING filters groups after GROUP BY.

## DevOps-Specific Add-ons

- **Connection pooling** — reuses DB connections instead of opening new ones per request; exhaustion is a common "app can't reach DB" root cause.
- **Replication lag** — delay between primary and replica data; monitor it or reads can return stale data.
- **RDS Multi-AZ vs Read Replica** — Multi-AZ is for failover/HA (synchronous standby, not for read scaling); Read Replica is for scaling reads (asynchronous, can be promoted).
- **Point-in-time recovery (PITR)** — restore DB to a specific timestamp using backups + transaction logs.
- **Disk space exhaustion** — DB can't write, transactions fail; monitor free storage via CloudWatch/Prometheus and alert before it hits 0.
