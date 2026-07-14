# Constraints Analysis

## Constraint 1: 5 lakh concurrent users at 12:00:00 noon
- **What is the peak RPS?** Assuming 500,000 users perform around 3 API calls within the first 60 seconds (refreshing, fetching event details, locking seats), the peak RPS is `(500,000 * 3) / 60 = 25,000 RPS`. 
- **What component hits its limit first at this RPS?** The database connection pool. Relational databases like PostgreSQL can handle high throughput, but connections are expensive. If average query time is fast, pgBouncer can multiplex, but holding connections for slow transactions (like payment processing) will instantly exhaust the pool.

## Constraint 2: Zero acceptable double-bookings
- **What does a double-booking mean technically?** Two or more rows in the `booking_seats` table referencing the same `seat_id` for an overlapping `held_until` timeframe, or a state where two bookings think they have successfully reserved the identical seat.
- **What mechanisms prevent this?** 
  1. *Optimistic Locking*: Using a `version` column on the `seats` table where the update succeeds only if the version hasn't changed.
  2. *Pessimistic Locking*: `SELECT FOR UPDATE` to lock the seat row in the DB until the transaction completes.
  3. *Distributed Locks*: Using Redis `SETNX` to acquire a lock on the seat ID before inserting a booking.

## Constraint 3: $2,000/month AWS budget
- **What does $2,000 actually buy?**
  - ~6 EC2 `t3.xlarge` instances behind an ALB (~$600)
  - 1 Multi-AZ RDS `r6g.xlarge` (~$500)
  - 3-node ElastiCache (Redis) `r6g.large` cluster (~$400)
  - SQS, ALB, NAT Gateways, and Data Transfer (~$500)
- **Which of your design choices must change if the budget were $500/month?**
  - We could not afford a large Multi-AZ RDS instance or a multi-node Redis cluster. 
  - *Change 1*: We would have to rely heavily on a Virtual Waiting Room (CloudFront + Edge functions) to forcefully limit the number of users reaching the backend to match our severely reduced DB capacity.
  - *Change 2*: We would use a smaller DB and perhaps drop Redis for a simpler in-memory application cache, sacrificing some horizontal scalability and consistency for cost.