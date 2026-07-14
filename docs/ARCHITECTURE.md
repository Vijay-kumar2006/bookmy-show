# System Architecture

```text
                                  +---------------------------------------------------+
                                  |                     USER (Browser/App)            |
                                  +------------------------+--------------------------+
                                                           | (HTTPS API Calls)
                                                           v
+---------------------------------------------------------------------------------------------------------+
|                                            CloudFront CDN                                               |
| - Serves: Static assets, cached event pages (Hit path: fast return)                                     |
| - Passes through: API calls for dynamic booking, locking (Miss path)                                    |
+----------------------------------------------------------+----------------------------------------------+
                                                           |
                                                           v
+---------------------------------------------------------------------------------------------------------+
|                                     Application Load Balancer (ALB)                                     |
| - SSL termination                                                                                       |
| - Health check interval: 10s                                                                            |
| - Rate limit rule: 100 req/IP/min                                                                       |
+----------------------------------------------------------+----------------------------------------------+
                                                           |
                                                           v
+---------------------------------------------------------------------------------------------------------+
|                                  Node.js API Servers (Auto-Scale Group)                                 |
| - Reads from Redis: Cached availability, event details                                                  |
| - Writes to DB: New users, held seats, pending bookings                                                 |
| - Publishes to SQS: Async payment processing messages                                                   |
+---------------------------------------------------------------------------------------------------------+
       |                           |                             |                              |
       | (Cache/Lock/Counter)      | (Queue)                     | (Writes)                     | (Reads)
       v                           v                             v                              v
+---------------+    +---------------------------+    +-----------------------+    +------------------------+
| Redis Cluster |    |    SQS Payment Queue      |    |  PostgreSQL Primary   |    | PostgreSQL Read Reps(x2|
|               |    |                           |    |                       |    |                        |
| (a) Cache TTL |    | - Msg Format: JSON        |    | - Write operations    |    | - Reads: Seat avail,   |
| (b) Seat lock |    | - Visibility Timeout: 30s |    | - Updates seat status |    |   event details        |
| (c) Hold Count|    | - DLQ Path: After 3 fails |    | - Updates booking     |    |                        |
|     (Max: 8)  |    +-------------+-------------+    +-----------------------+    +------------------------+
+---------------+                  |                             ^                              ^
                                   | (Read SQS)                  |                              |
                                   v                             | (Update DB)                  |
                     +---------------------------+               |                              |
                     |   Payment Worker (ECS)    |---------------+                              |
                     |                           |                                              |
                     | 1. Read SQS message       |                                              |
                     | 2. Call Payment Gateway   |                                              |
                     | 3. Update DB (Confirmed)  |                                              |
                     | 4. Publish to SNS         |                                              |
                     | 5. Delete msg from SQS    |                                              |
                     +-------------+-------------+
                                   |
                                   | (Publish SNS)
                                   v
                     +---------------------------+
                     |            SNS            |
                     | Triggered on confirmation |
                     +-------------+-------------+
                                   |
                                   v
                     +---------------------------+
                     |    SES Email + SMS        |
                     +---------------------------+

+-------------------------------------------------------------------+
|                       CloudWatch Alarms                           |
| - Alert: SQS Queue Depth > 10,000 (Triggers ECS Auto-Scale up)    |
| - Alert: SQS DLQ Depth > 100 (Triggers PagerDuty On-Call)         |
+-------------------------------------------------------------------+
```

## Design Annotations (Part A References & Part B Updates)

- **← Redis SETNX lock** *(see CONCURRENCY.md: 500K RPS demand makes Postgres row locking unscalable, exhausting 500 connection pool at ~2840 RPS).*
- **← Async queue (SQS)** *(see QUEUE.md: sync payment at 5L users = pool exhaustion in under a second; async isolates slow payment gateway).*
- **← Read replica (x2)** *(see SCHEMA.md & CACHE.md: Seat availability reads separate from write transactions to prevent read-heavy traffic from starving the primary).*
- **← UUID generation** *(see SCHEMA.md: ID generated in Node.js API server to enable idempotent payment queueing).*
- **← Cache-aside** *(see CACHE.md: Event-driven invalidation from API to Redis to prevent stale cache during high contention).*
- **[UPDATE] ← Redis Hold Count (Max: 8)** *(see DESIGN-UPDATES.md: Added Lua script counter per user to prevent scalping bots from holding 200+ seats, exposed by Q3).*
- **[UPDATE] ← CloudWatch Alarms** *(see DESIGN-UPDATES.md: Added strict queue depth monitoring to detect payment gateway failures and automatically scale workers or alert engineers).*
