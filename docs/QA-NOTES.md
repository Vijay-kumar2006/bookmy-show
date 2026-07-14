# Live Panel Q&A Notes

**Q1: What happens if Redis crashes mid-lock?**
- *Answer given:* Restated the risk. Acknowledged that a Redis crash during `SETNX` means the lock might be lost or never propagate to replicas. Walked through the design: the application falls back to PostgreSQL Optimistic Locking (the `version` column).
- *Limit of design:* If Redis goes completely down, the API falls back to hitting the DB for locks, which will instantly exhaust the 500 PgBouncer connections at peak load.
- *Change if occurs:* We would need to fail fast at the API layer (circuit breaker) to protect the DB until Redis recovers.
- *Self-evaluation:* Complete answer.

**Q2: At what concurrent users does your DB become the bottleneck?**
- *Answer given:* Acknowledged connection limits. Explained that with Postgres pessimistic locking, it breaks at ~2,840 RPS (around 50k users). With Redis handling locks and reads hitting Replicas, the primary DB only handles the final `UPDATE`/`INSERT`. At 25,000 RPS, if 10% convert to actual buys (2,500 TPS), Postgres handles this fine on an `r6g.xlarge` instance. The bottleneck shifts to network bandwidth or application CPU before the DB primary fails.
- *Self-evaluation:* Good math, solid defense.

**Q3: What stops one user from holding 200 seats?**
- *Answer given:* Acknowledged this is a real scalping threat. Walked through the current design... and realized a gap. Currently, nothing strictly limits the number of active Redis `SETNX` locks per `userId`.
- *Limit of design:* A bot could rapidly fire 200 lock requests for different `seat_id`s and hold them all for 10 minutes.
- *Change if occurs:* Must implement a per-user hold counter in Redis.
- *Self-evaluation:* Found a major flaw in my own design during the answer. Needs immediate architectural update.

**Q4: Your auto-scaling spikes your bill to $3,200 this month - what's your plan?**
- *Answer given:* Restated the strict $2,000 budget constraint. Walked through the auto-scale group triggers. 
- *Limit of design:* Currently, ECS and EC2 scale based on CPU/Queue depth without an upper bound cap.
- *Change if occurs:* We must set hard `MaxCapacity` limits on the AWS Auto Scaling Groups (ASG). If we hit the ASG limit, latency will increase and users will get 503s, but we enforce the budget constraint.
- *Self-evaluation:* Answered effectively by prioritizing the business constraint over 100% uptime.

**Q5: Why not PostgreSQL row locking instead of Redis? Defend your choice.**
- *Answer given:* Restated the choice. Walked through the math: row locks hold the connection for the duration of the API call. 500 max connections / 0.02s query time = max 25,000 theoretical RPS. However, network latency and client-side processing mean connections are held much longer. Redis pushes the lock state to memory, freeing the DB to only process rapid, final `COMMIT` statements. 
- *Self-evaluation:* Confident defense referencing CONCURRENCY.md math.
