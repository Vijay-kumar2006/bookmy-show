# Concurrency Strategy

## Option A: PostgreSQL SELECT FOR UPDATE (Pessimistic Locking)

**How it prevents double-booking:**
The application issues a `BEGIN` statement, followed by `SELECT * FROM seats WHERE id IN (...) FOR UPDATE`. This requests a row-level lock in Postgres. If another transaction tries to read or update these same rows `FOR UPDATE`, it will block until the first transaction either `COMMIT`s or `ROLLBACK`s. Once locked, the application updates the seat status to 'held', inserts the pending booking, and commits.

**Hard Limit & Connection Pool Exhaustion:**
The database connection pool becomes the bottleneck at high RPS because connections are blocked waiting for transactions to complete. 
Formula: `Connections held = (% non-payment RPS × avg_query_time_s) + (% payment RPS × payment_hold_time_s)`
Assume `max_connections = 500` (PgBouncer), 20% of traffic is payment calls taking 800ms, and 80% is regular traffic taking 20ms.
Let `R` be the Total RPS.
`Connections held = (0.8 * R * 0.02) + (0.2 * R * 0.8) = 0.016R + 0.16R = 0.176R`
To exhaust the pool: `0.176R = 500` => `R ≈ 2840` RPS.
The DB pool exhausts at just **2,840 RPS**.

**Deadlock Risk:**
Multi-seat bookings risk deadlocks. If Transaction 1 locks Seat A then Seat B, and Transaction 2 locks Seat B then Seat A, both wait forever. 
*Mitigation*: Always sort the `seat_id`s numerically before executing the `SELECT FOR UPDATE` query.

## Option B: Redis SETNX Distributed Lock

**How it prevents double-booking:**
The application tries to acquire a lock using `SETNX seat_lock:{seat_id} {user_id} EX {ttl}`. If it returns 1, the lock is acquired. If 0, someone else is booking it. When releasing, a Lua script checks if the lock value still matches the `{user_id}` before executing `DEL`, ensuring a user doesn't delete someone else's lock if their own TTL expired.

**Failure Mid-Lock:**
If Redis fails mid-lock, we don't necessarily get a double booking *if* the DB acts as the final source of truth. However, if the lock succeeds but the app crashes before writing to the DB, the seat remains unbookable until the TTL expires. This causes a temporary "ghost booking" but not a double booking.

**TTL Value:**
TTL should be around 10 minutes (600 seconds) for a user to complete the payment flow. 
- *Too short*: Lock expires while the user is typing their credit card, someone else buys it, leading to a terrible user experience.
- *Too long*: If the user abandons the checkout, the seat is unnecessarily locked and cannot be sold.

**Lock Key Structure:**
`seat_lock:{event_id}:{seat_id}`. Including the `event_id` provides context and makes it easier to flush all locks for a specific event if the event is suddenly cancelled.

## Our Chosen Strategy

We choose **Option B (Redis SETNX Distributed Lock)** combined with asynchronous payment processing (Queue) and DB Optimistic Locking.

**Justification:**
1. **Capacity Constraint:** At 500,000 users, peak traffic approaches 25,000 RPS. Option A (Postgres row locks) completely exhausts our 500-connection PgBouncer pool at just ~2,840 RPS. Redis can easily handle 100k+ ops/sec on a cluster.
2. **Budget Constraint:** A $2,000/month budget means we cannot simply scale out RDS infinitely to throw more connections at the problem. A 3-node ElastiCache cluster is relatively cheap and absorbs the high-concurrency read/write locking workload, protecting the limited RDS connections.
3. **What it cannot handle:** Redis locks cannot handle split-brain scenarios or severe network partitions in a Redis cluster perfectly (unless we use Redlock, which adds complexity). If Redis loses a write during a failover, a lock might be lost, falling back to the DB's Optimistic Locking (version column) to catch the double-booking at the final write step.
4. **Switch Condition:** If our architecture changes to use a distributed SQL database (like CockroachDB or Spanner) that handles row locking at immense scale natively without a centralized pool bottleneck, we would switch back to Option A to reduce architectural complexity (removing Redis).
