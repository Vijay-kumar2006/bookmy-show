# Database Schema

```sql
-- 1. users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    phone VARCHAR(20) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL
);

-- 2. venues
CREATE TABLE venues (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    city VARCHAR(100) NOT NULL,
    capacity INTEGER NOT NULL CHECK (capacity > 0)
);

-- 3. events
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    venue_id INTEGER NOT NULL REFERENCES venues(id) ON DELETE RESTRICT,
    name VARCHAR(255) NOT NULL,
    start_time TIMESTAMP WITH TIME ZONE NOT NULL,
    status VARCHAR(20) NOT NULL CHECK (status IN ('upcoming', 'on_sale', 'sold_out', 'cancelled')),
    total_seat_count INTEGER NOT NULL CHECK (total_seat_count > 0)
);

-- 4. seats
CREATE TABLE seats (
    id SERIAL PRIMARY KEY,
    event_id INTEGER NOT NULL REFERENCES events(id) ON DELETE CASCADE,
    section VARCHAR(50) NOT NULL,
    row VARCHAR(10) NOT NULL,
    number VARCHAR(10) NOT NULL,
    price DECIMAL(10, 2) NOT NULL CHECK (price > 0),
    category VARCHAR(20) NOT NULL CHECK (category IN ('VIP', 'General', 'Premium')),
    status VARCHAR(20) NOT NULL CHECK (status IN ('available', 'held', 'booked')),
    held_until TIMESTAMP WITH TIME ZONE,
    held_by UUID REFERENCES users(id) ON DELETE SET NULL,
    version INTEGER NOT NULL DEFAULT 1,
    UNIQUE (event_id, section, row, number)
);

-- 5. bookings
CREATE TABLE bookings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    event_id INTEGER NOT NULL REFERENCES events(id) ON DELETE RESTRICT,
    status VARCHAR(20) NOT NULL CHECK (status IN ('pending', 'confirmed', 'failed', 'refunded')),
    total_amount DECIMAL(10, 2) NOT NULL CHECK (total_amount >= 0),
    payment_reference VARCHAR(255),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL
);

-- 6. booking_seats
CREATE TABLE booking_seats (
    booking_id UUID NOT NULL REFERENCES bookings(id) ON DELETE CASCADE,
    seat_id INTEGER NOT NULL REFERENCES seats(id) ON DELETE RESTRICT,
    PRIMARY KEY (booking_id, seat_id)
);

-- INDEXES
CREATE INDEX idx_seats_event_status ON seats(event_id, status);
CREATE INDEX idx_bookings_user ON bookings(user_id, created_at DESC);
CREATE INDEX idx_bookings_status_partial ON bookings(status) WHERE status = 'pending';
```

## Commentary

**Why UUID for booking.id instead of SERIAL?**
UUIDs are globally unique, which hides the total number of bookings (business intelligence protection) and makes it significantly harder for an attacker to guess order IDs. Additionally, UUIDs can be generated on the client or application server before the database insert, enabling idempotent retry patterns in our async queue.

**Why does seats have a version column?**
The `version` column is used for Optimistic Concurrency Control (OCC). During high contention, instead of taking heavy pessimistic row locks (`SELECT FOR UPDATE`), the system can check if the seat's version matches what was read. If another transaction updated the seat in the meantime, the version would increment, causing our update to fail safely (`UPDATE seats SET ... WHERE id = X AND version = Y`), preventing double-bookings.

**Why held_until instead of just holding seats at the application level?**
Holding state in the DB ensures the application remains stateless and distributed. If an application node crashes while a user is holding seats, the database inherently knows when the hold expires via `held_until`. A background sweeper or the next query can simply free up seats where `held_until < NOW()`.

**Why partial index on bookings.status?**
The vast majority of bookings will end up in a terminal state (`confirmed`, `failed`, `refunded`). The system primarily needs to aggressively query and monitor unresolved bookings (`pending`) to clean up timeouts, process asynchronous payment confirmations, or expire holds. Indexing only `pending` rows drastically reduces the index size and speeds up these critical background queries.
