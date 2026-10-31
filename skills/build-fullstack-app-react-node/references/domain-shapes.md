# Domain Shapes Reference

Both shapes share one rule that matters more than any schema detail: **the server never trusts client-sent prices, quantities, or availability.** The client sends *what* was selected (`{ id, qty }` or `{ resourceId, startsAt, endsAt }`); the server looks up the real price and re-checks availability from the database inside the same transaction that commits the order/booking. A client that sent `price: 0.01` must get charged the real price, full stop — this is the one piece of business logic that's worth being paranoid about in every variant of this app.

## Shape 1: Stock-count (e-commerce / catalog)

A product has a `stock` integer. Checkout decrements it. The concurrency hazard is two buyers racing for the last unit — **a single `UPDATE` with a guard clause solves this atomically**, no explicit row lock needed:

```sql
CREATE TABLE IF NOT EXISTS products (
  id          TEXT PRIMARY KEY,
  name        TEXT NOT NULL,
  category    TEXT NOT NULL,
  price       NUMERIC(10, 2) NOT NULL CHECK (price >= 0),
  stock       INTEGER NOT NULL CHECK (stock >= 0),
  description TEXT NOT NULL,
  sort_order  INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE IF NOT EXISTS orders (
  id          TEXT PRIMARY KEY,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  status      TEXT NOT NULL DEFAULT 'Processing' CHECK (status IN ('Processing', 'Cancelled')),
  -- shipping/contact fields, subtotal/shipping/tax/total NUMERIC(10,2)
  ...
);

-- Name/price/etc. are copied at purchase time so later catalog edits
-- never rewrite order history.
CREATE TABLE IF NOT EXISTS order_items (
  order_id   TEXT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  product_id TEXT NOT NULL REFERENCES products(id),
  name       TEXT NOT NULL,
  price      NUMERIC(10, 2) NOT NULL,
  qty        INTEGER NOT NULL CHECK (qty > 0),
  PRIMARY KEY (order_id, product_id)
);
```

```js
// Atomic check-and-decrement: no row is returned if there isn't enough stock.
// Sort ids before looping so concurrent multi-item orders always take locks in
// the same order and can't deadlock against each other.
const { rows } = await db.query(
  'UPDATE products SET stock = stock - $2 WHERE id = $1 AND stock >= $2 RETURNING *',
  [id, qty],
);
if (rows.length === 0) {
  // Either the id doesn't exist, or there wasn't enough stock — look it up to say which.
  throw new HttpError(409, `Sorry, only N left in stock.`);
}
```

Cancelling an order restores stock the same way, but needs an explicit lock because it's a multi-row read-then-write rather than a single atomic `UPDATE`: lock the order row with `SELECT ... FOR UPDATE` first (so two cancel requests on the same order can't both restore stock), then lock the affected product rows **in id order** — the same order used when placing orders, which is what prevents a deadlock between a concurrent "place order" and "cancel order" on overlapping products.

## Shape 2: Time-range (booking / reservation)

A resource (court, chair, van, room, provider) is booked for `[starts_at, ends_at)`. The hazard is two bookings on the *same resource* with *overlapping* ranges — and this is **not** safe to handle with a row lock, because the two conflicting rows don't exist yet when the conflict check runs. "Query for overlapping bookings, see none, insert" is a real race: two concurrent requests can both pass the query before either inserts.

**The fix is a database-enforced exclusion constraint**, so the conflict is impossible regardless of what the application code does:

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;  -- standard Postgres contrib extension; Neon supports it

CREATE TABLE IF NOT EXISTS resources (
  id          SERIAL PRIMARY KEY,
  name        TEXT NOT NULL,
  category    TEXT NOT NULL,
  price       NUMERIC(10, 2) NOT NULL,
  details     JSONB NOT NULL DEFAULT '{}',  -- domain-specific fields, see table below
  description TEXT
);

CREATE TABLE IF NOT EXISTS bookings (
  id             SERIAL PRIMARY KEY,
  booking_number TEXT UNIQUE NOT NULL,
  resource_id    INTEGER NOT NULL REFERENCES resources(id),
  user_id        TEXT REFERENCES "user"(id) ON DELETE SET NULL,  -- omit if no auth
  starts_at      TIMESTAMPTZ NOT NULL,
  ends_at        TIMESTAMPTZ NOT NULL,
  status         TEXT NOT NULL DEFAULT 'confirmed' CHECK (status IN ('confirmed', 'cancelled')),
  customer_name  TEXT NOT NULL,
  customer_email TEXT NOT NULL,
  total_price    NUMERIC(10, 2) NOT NULL,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  CONSTRAINT valid_range CHECK (ends_at > starts_at),
  EXCLUDE USING gist (
    resource_id WITH =,
    tstzrange(starts_at, ends_at) WITH &&
  ) WHERE (status = 'confirmed')
);
```

The `EXCLUDE` clause: for the same `resource_id`, no two `confirmed` rows may have overlapping ranges. A conflicting `INSERT` fails with SQLSTATE `23P01` — handle that specific code to return a friendly 409:

```js
try {
  await client.query('BEGIN');
  // ... look up resource, compute price, INSERT INTO bookings ...
  await client.query('COMMIT');
} catch (err) {
  await client.query('ROLLBACK');
  if (err.code === '23P01') throw new HttpError(409, 'That time slot was just booked. Please pick another.');
  throw err;
}
```

Cancelling just flips `status` to `'cancelled'` — the partial-index `WHERE (status = 'confirmed')` means a cancelled slot is immediately rebookable, no stock to restore.

### Mapping domains onto the `resources`/`bookings` schema

| Domain | `resources` row | Slot model | Extra `details` JSONB fields |
|---|---|---|---|
| Tennis courts | one court | fixed grid (hourly, 7am–10pm) | `surface`, `indoor` |
| Doctor/dentist appointments | a provider | provider-defined windows, variable duration | `specialty`, `appointmentTypes: [{name, minutes}]` |
| Salon/spa slots | a stylist/chair | fixed grid tied to service duration | `services: [{name, minutes, price}]` |
| Equipment/vehicle rental | one unit | continuous multi-day spans | `make`, `model`, `dailyRate` |
| Meeting rooms | one room | fixed grid (30-min increments) | `capacity`, `amenities` |

Don't build a separate schema per domain — only `computePrice()`, the slot-generation logic for the booking UI, and `details` change.

**Fixed-grid UIs** (tennis courts, salons, meeting rooms): generate the day's slots in the frontend (every hour/30min from open to close) and mark each available/booked by checking the resource's existing bookings for that day, fetched with an overlap query:

```sql
SELECT starts_at, ends_at FROM bookings
WHERE resource_id = $1 AND status = 'confirmed'
  AND starts_at < $3 AND ends_at > $2   -- $2 = window start, $3 = window end
ORDER BY starts_at;
```

## Simulated status, either shape

A real fulfillment/scheduling backend (shipping carriers, staff marking an appointment complete) is out of scope for a first build. Instead, store only the states the server actually controls (`Processing`/`Cancelled`, or `confirmed`/`cancelled`) and **derive** cosmetic intermediate stages from elapsed time on the client/shared code, compressed to minutes so it's demoable:

```js
// src/shared/status.js — imported by both the browser and the server.
export const STAGES = [
  { status: 'Processing', after: 0 },
  { status: 'Shipped', after: 2 * 60_000 },
  { status: 'Out for delivery', after: 5 * 60_000 },
  { status: 'Delivered', after: 10 * 60_000 },
];

export function withStatus(order) {
  if (order.status === 'Cancelled') return order;
  const age = Date.now() - new Date(order.createdAt).getTime();
  return { ...order, status: STAGES.findLast((s) => age >= s.after).status };
}
```

Only allow cancellation while the *derived* status is still the first stage — once it's "Shipped" or later, treat it as committed, the same way a real fulfillment system would.
