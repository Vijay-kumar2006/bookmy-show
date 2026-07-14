# Architecture Updates (Post-Roast)

## Update 1: Per-User Seat Hold Limit
 
**Triggered by:** Panel Question 3 - "What stops one user from holding 200 seats?"
 
**What changed:**
Added a per-user seat hold counter in Redis using the key: `holds:{userId}:count`.
When a user attempts to lock a seat:
1. API executes a Lua script that increments `holds:{userId}:count`.
2. If the count exceeds 8 (max tickets per transaction), the script returns an error and rejects the lock.
3. The counter has a TTL matching the seat hold TTL (10 minutes).
4. When a booking succeeds or fails, the counter is decremented.
 
**Why this is necessary:**
The original design prevented double-bookings for a *single* seat, but left a massive loophole where malicious bots could iterate through thousands of `seat_id`s, acquiring a lock for every single one and freezing the entire event's inventory for 10 minutes without paying.
 
**What it costs:**
Additional Redis memory (one key per active user) and a slight increase in latency for the lock acquisition step (executing a Lua script instead of a basic `SETNX`), but this is negligible in a Redis cluster.
 
**What it still doesn't solve:**
It does not prevent a sophisticated attacker from creating hundreds of fake user accounts to bypass the per-user limit. Solving that requires WAF-level bot mitigation or phone number verification during account creation.

---

## Update 2: SQS Dead-Letter/Circuit Breaker Alerting
 
**Triggered by:** Panel Question 1 & internal reflection on queue reliance - "What if the worker fails entirely?"
 
**What changed:**
Added a CloudWatch Alarm triggered when `SQS Queue Depth > 10,000` or `DLQ Depth > 100`. 
This alarm triggers an automated SNS alert to the on-call engineering team and dynamically scales up the ECS Payment Worker cluster. 
 
**Why this is necessary:**
The original design assumed the payment worker would always gracefully handle load. However, if the payment gateway (e.g., Stripe) experiences a total outage, our SQS queue will silently balloon with pending orders, eventually pushing everything to the DLQ after 3 retries, leading to thousands of failed user bookings without immediate engineering awareness.
 
**What it costs:**
Minimal AWS CloudWatch metrics pricing and the cost of potentially over-provisioning ECS workers during a transient spike.
 
**What it still doesn't solve:**
If the payment gateway is hard-down for hours, auto-scaling our workers won't fix it. The queue will eventually overflow or age out, requiring complex manual reconciliation or mass refunds/cancellations.