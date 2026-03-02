# Smart Parking Lot System Design

This document shows the low-level architecture and design considerations for a backend system managing a smart parking lot.

## Overview

The system handles:

- Automatic parking spot allocation based on vehicle size and availability.
- Recording vehicle check-in and check-out times.
- Calculating parking fees according to duration and vehicle type.
- Real-time updates of parking spot availability.
- Concurrency handling to support simultaneous entries/exits.

Designed for urban multi-floor parking structures with diverse vehicle types.

---

## Data Model

A relational database (e.g., PostgreSQL) maintains core entities.

### Tables

- **ParkingSpot**
  - `spot_id` (PK)
  - `floor`
  - `size` (ENUM: SMALL, MEDIUM, LARGE)
  - `status` (ENUM: AVAILABLE, RESERVED, OCCUPIED)

- **Vehicle**
  - `vehicle_id` (PK)
  - `license_plate` (unique)
  - `type` (ENUM: MOTORCYCLE, CAR, BUS)

- **EntryRecord**
  - `record_id` (PK)
  - `vehicle_id` (FK)
  - `spot_id` (FK)
  - `entry_time`
  - `exit_time` (nullable)
  - `fee` (nullable until exit)

- **Rate**
  - `vehicle_type` (PK)
  - `base_rate_per_hour`
  - `grace_period` (minutes)
  - `max_daily` (optional cap)

### Relationships

- `Vehicle` 1→N `EntryRecord`
- `ParkingSpot` 1→N `EntryRecord`

> **Indexes** should be created on `ParkingSpot.status` and `EntryRecord.entry_time` for efficient queries.

---

## Spot Allocation Algorithm

1. **Event Queue**: Entry events are published to a broker (e.g., Kafka).
2. **Acquire Lock & Search**: Within a transaction, query for the smallest available spot that fits the vehicle.

   ```sql
   SELECT spot_id
   FROM ParkingSpot
   WHERE status = 'AVAILABLE'
     AND size >= :vehicleSize
   ORDER BY size ASC
   LIMIT 1
   FOR UPDATE;
   ```

3. **Reserve Spot**: Update the spot status to `RESERVED` and insert an `EntryRecord` with `entry_time`.
4. **Timeout Handling**: If the vehicle fails to park within a set time, release the reservation.

> **Concurrency**: Use `SELECT … FOR UPDATE SKIP LOCKED` or optimistic locking; partition spots by floor/size for scalability.

---

## Fee Calculation Logic

Upon exit:

1. Retrieve the `EntryRecord` and compute duration.
2. Use rate table to determine pricing.
3. Apply grace periods, hourly charges, and caps.
4. Update `EntryRecord.fee` and mark spot `AVAILABLE`.

Sample logic:

```java
double calculateFee(VehicleType type, Duration duration) {
    Rate rate = rateRepo.findByType(type);
    if (duration.toMinutes() <= rate.gracePeriod) return 0;
    double hours = Math.ceil(duration.toHours());
    double fee = hours * rate.baseRate;
    return Math.min(fee, rate.maxDaily);
}
```

---

## Real-Time Availability

- **Cache**: Use Redis to mirror spot availability counts.
- **Pub/Sub**: Publish updates on entry/exit to WebSocket topics or a RESTful endpoint.
- **API**: Provide endpoints such as `/spots/available?size=CAR&floor=1`.

---

## Concurrency & Scaling

- **Transactional Boundaries**: Ensure atomic operations for spot assignment and exit processing.
- **Row-Level Locking**: Prevent race conditions when multiple vehicles compete for spots.
- **Stateless Services**: Enable horizontal scaling behind load balancers.
- **Partitioning**: Separate service responsibilities by floor or spot size.

---

## System Components

1. **API Gateway** – Routes external requests to internal services.
2. **Entry Service** – Validates incoming vehicles and assigns spots.
3. **Exit Service** – Calculates fees and frees spots.
4. **Spot Manager** – Maintains and updates availability cache.
5. **Billing Module** – Handles rate logic and invoice generation.
6. **Database** – Stores persistent data.
7. **Messaging Layer** – Handles queueing of events.
8. **Monitoring & Logging** – Tracks occupancy, anomalies, and system health.

---

## Additional Considerations

- Map vehicle sizes to spot sizes (motorcycle -> SMALL, car -> MEDIUM, bus -> LARGE).
- Implement fault tolerance with reservation timeouts and reconciliation jobs.
- Provide an admin portal for rate management and manual overrides.
- Future enhancements could include reservations and dynamic pricing.

---
