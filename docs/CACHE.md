# Cache Layer Design

## Required Cache Entries

### 1. Event Details
- **Redis Key Format:** `event:{event_id}`
- **TTL Value:** 5 minutes (300 seconds). Event details (name, venue, start time) rarely change, but a 5-minute TTL provides a safety net if someone corrects a typo in the event name without explicitly triggering an invalidation.
- **Invalidation Trigger:** Whenever the event details are updated via the admin dashboard, or if the event status changes (e.g., transitions to `cancelled`).
- **Strategy:** Cache-aside. We delete the key on update. The next read repopulates it.

### 2. Seat Availability Count
- **Redis Key Format:** `availability:{event_id}:{category}`
- **TTL Value:** 30 seconds. At 5L concurrent users, seat availability changes hundreds of times per second. A 30-second TTL prevents DB bombardment while keeping the UI reasonably fresh. Users understand that "Available: 15" might be slightly stale until they click.
- **Invalidation Trigger:** We do *not* invalidate this on every single seat hold/book, because doing so at 25,000 RPS would render the cache useless (cache churn). 
- **Strategy:** TTL-only expiry. Every 30 seconds, the cache expires, and one request will query the DB `COUNT(id) FROM seats WHERE event_id = X AND category = Y AND status = 'available'` to refresh it.

### 3. Static Seat Map Layout
- **Redis Key Format:** `seatmap:{event_id}`
- **TTL Value:** 24 hours (86,400 seconds). The physical layout of the venue (sections, rows, total seats) does not change once the event is published.
- **Invalidation Trigger:** Admin explicitly edits the venue/seat configuration (extremely rare after `on_sale`).
- **Strategy:** Cache-aside. Event-driven invalidation.

### What We Will NOT Cache
- **Individual Seat Status:** We will NOT cache the exact status (available/held/booked) of individual seats in Redis for read purposes. 
- *Why:* The risk of caching individual seat status is that users will make decisions based on stale data. If a seat is held but the cache says 'available', the user clicks it, tries to lock it, and gets rejected. This leads to immense frustration. We serve the granular seat status directly from the DB (using indexed queries) or via WebSockets for real-time updates.

## The Invalidation Strategy

Our overall approach is the **Cache-Aside Pattern (Delete on Write)**.

*Why not Write-Through?*
Write-through (updating the Redis key in-place) introduces race conditions. If Request A and Request B update the DB in that order, but due to network jitter, Request B updates Redis *before* Request A, the cache now holds stale data permanently until TTL expiry. Deleting the key forces the next read to fetch the guaranteed latest state from the database.

**Code-Level Logic (Event Update):**
```python
# When event status changes
def update_event_status(event_id, new_status):
    # 1. Update DB (Source of Truth)
    db.execute(
        "UPDATE events SET status = %s WHERE id = %s", 
        [new_status, event_id]
    )
    
    # 2. Delete Redis Key
    redis.delete(f"event:{event_id}")
    
# Next read will:
def get_event(event_id):
    cached = redis.get(f"event:{event_id}")
    if cached:
        return cached
        
    # Cache miss
    event = db.query("SELECT * FROM events WHERE id = %s", [event_id])
    redis.set(f"event:{event_id}", event, ex=300)
    return event
```
