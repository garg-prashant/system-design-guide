# System Design: Uber

## Problem

Design a ride-sharing system: match riders with drivers, track location, handle ride lifecycle (request, accept, in-progress, complete), and support surge pricing.

## Requirements

**Functional:** Rider requests ride (pickup/drop); nearby drivers see request (or get matched); driver accepts; track location during ride; complete ride and payment; surge pricing by area/time.

**Non-functional:** Low latency for matching and ETA; real-time location updates; high availability; handle geographic load (hotspots).

## High-Level Design

```
  Riders/Drivers (mobile) → API Gateway → Matching service, Trip service, Location service
                                      → Geospatial index (drivers by location)
                                      → Trip state machine, Pricing
```

- **Location:** Drivers send location (lat, lng) periodically; store in geospatial index (e.g. grid, quadtree, or GeoHash) for “drivers near point.”
- **Request ride:** Rider pickup/drop; matching service queries geospatial index for nearby available drivers; send request to drivers (push); first accept wins or assign best.
- **Trip lifecycle:** Created → driver_accepted → driver_arrived → in_progress → completed. State in DB; events to billing and analytics.
- **Surge pricing:** Pricing service uses demand (active requests) and supply (available drivers) per zone; multiplier; stored and shown to rider/driver.

## Key Components

- **Location service:** Ingest driver locations; update geospatial index (e.g. Redis Geo, or custom grid). Query: “drivers within R km of (lat, lng).”
- **Matching service:** Get nearby drivers; filter by type, rating; send request to N drivers; accept flow with locking (one driver gets the trip).
- **Trip service:** CRUD trip; state transitions; persist; notify rider/driver.
- **Pricing service:** Zone-based demand/supply; surge multiplier; fare calculation (distance, time, surge).
- **Notification:** Push to drivers (new request); push to rider (driver accepted, ETA, etc.).

## Data Model

- **drivers:** driver_id, current_lat, current_lng, status (available/busy), updated_at. Index: geospatial.
- **trips:** trip_id, rider_id, driver_id, pickup, drop, status, created_at, completed_at, fare.
- **zones:** zone_id, polygon or grid; surge_multiplier, updated_at.

## Scale

- Millions of drivers; location updates every few seconds → high write rate. Geospatial index must support range queries and updates; shard by region. Trip and pricing reads/writes at request rate; DB + cache.

## Trade-offs

- **Matching:** Push request to nearby drivers (race to accept) vs assign centrally. Central assign: better fairness and ETA; push: simpler, driver choice.
- **Location storage:** Store only “current” in index vs history. History for replay and analytics; separate pipeline for analytics.
- **Surge:** Dynamic pricing improves supply; explain clearly to avoid trust issues.

## Failure & Operations

- **Location ingestion:** Backpressure; drop old updates if overloaded; prioritize matching queries.
- **Matching:** Timeout if no accept; retry or expand radius; idempotency so double-accept doesn’t create two trips.
- **Monitoring:** Match latency, ETA accuracy, trip completion rate, surge algorithm health.
