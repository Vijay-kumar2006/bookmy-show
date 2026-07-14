# Async Order Flow (Payment Queue)

## Section 1: Why Async?

Synchronous payment calls are catastrophic at 500,000 users. Payment gateways are inherently slow (often taking 800ms - 2000ms to respond).
If we process payments synchronously, the API thread and the database connection remain locked waiting for the payment gateway.
**The Math:**
Using the formula: `Connections held = (% non-payment RPS × avg_query_time_s) + (% payment RPS × payment_hold_time_s)`
Assume peak traffic is 25,000 RPS. 
If 20% of this traffic (5,000 RPS) consists of users hitting the "Pay" endpoint, and the payment gateway takes 1.5 seconds:
`Connections held for payment alone = 5,000 * 1.5 = 7,500 connections.`
Our Postgres pool (PgBouncer) maximum is 500 connections. At 5,000 RPS, the pool would collapse in a fraction of a second, causing the entire API to return `503 Service Unavailable` for all users, including those just trying to view the homepage. Asynchronous queues isolate this slow, third-party dependency from our critical fast-path.

## Section 2: The Queue Message Format

```json
{
  "bookingId": "a1b2c3d4-e5f6-7a8b-9c0d-1e2f3a4b5c6d",
  "userId": "u9x8y7z6-...",
  "totalAmount": 150.00,
  "paymentToken": "tok_visa_12345",
  "seatIds": [101, 102],
  "eventId": 42,
  "idempotencyKey": "req_88bbhh33"
}
```
**Field Explanations:**
- `bookingId`: UUID generated upfront. Used to update the specific booking record to `confirmed` or `failed`.
- `userId`: To notify the user via websocket/email upon success, and to verify they own the booking.
- `totalAmount`: Passed directly to the payment gateway to authorize the charge.
- `paymentToken`: The tokenized credit card or payment intent from the frontend.
- `seatIds`: Included so the worker can finalize the seats (update status from `held` to `booked`) without needing a DB lookup to figure out which seats belong to this booking.
- `eventId`: Useful for analytics or partition routing without extra DB joins.
- `idempotencyKey`: Crucial for safely retrying the payment charge against Stripe/Razorpay if the worker crashes mid-process.

## Section 3: The Worker Logic

On receipt of a message, the worker executes:
1. **Validation**: Check DB if `bookingId` status is still `pending`. If not, ack the message and drop (it was already processed).
2. **Idempotent Charge**: Call the Payment Gateway API using `totalAmount`, `paymentToken`, and `idempotencyKey`.
3. **Success Path**:
   - If gateway returns Success:
   - `BEGIN` transaction.
   - `UPDATE bookings SET status = 'confirmed', payment_reference = '...' WHERE id = bookingId`.
   - `UPDATE seats SET status = 'booked', held_until = NULL WHERE id IN (seatIds)`.
   - `COMMIT`.
   - Delete message from SQS (ACK).
   - Emit event to notify the user via WebSocket/Email.
4. **Failure Path**:
   - If gateway returns Hard Decline (insufficient funds):
   - `BEGIN` transaction.
   - `UPDATE bookings SET status = 'failed' WHERE id = bookingId`.
   - `UPDATE seats SET status = 'available', held_by = NULL, held_until = NULL WHERE id IN (seatIds)`.
   - `COMMIT`.
   - Delete message from SQS (ACK).
   - Emit failure event to user.
5. **DLQ Handling**:
   - If DB is down, or worker crashes abruptly, the message remains un-ACKed, becomes visible again, and eventually moves to the DLQ after `maxReceiveCount` is hit for manual intervention.

## Section 4: Edge Cases

**Edge Case 1: Server crashes after SQS publish but before API responds**
- *Scenario:* The API enqueues the message successfully but dies before returning `200 OK` to the user's browser.
- *Resolution:* The user's browser sees a 500 error. However, the worker *will* process the payment and confirm the booking. To prevent a poor UX, the frontend should immediately start polling or listen on a WebSocket for `bookingId` status if it gets a 5xx on payment submission. The user still gets their tickets.

**Edge Case 2: Payment gateway returns a timeout**
- *Scenario:* We call Stripe, but Stripe's API hangs and drops the connection. We don't know if they charged the user.
- *Resolution:* We do *not* fail the booking. We throw an exception in the worker, leaving the SQS message un-ACKed. SQS will retry it. On the retry, the worker sends the exact same `idempotencyKey`. Stripe recognizes the key; if it succeeded previously, it returns the previous success response instead of charging again. We then successfully complete the DB updates.

## Section 5: SQS Configuration

- **Visibility Timeout:** `30 seconds`. The worker usually takes 1-2 seconds. If a worker crashes, we want the message to be picked up by another worker relatively quickly. However, it must be longer than the maximum possible timeout of the Payment Gateway (e.g., 10 seconds).
- **Max Receive Count:** `3`. If a message fails 3 times, it implies a systematic failure (e.g., persistent DB crash, malformed data, or severe gateway outage). Pushing it to a Dead Letter Queue (DLQ) after 3 tries prevents poison-pill messages from endlessly looping and burning compute resources.
