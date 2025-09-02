# 🚀 Project Roadmap ( ~2-3 Weeks )

## ⚠

 Although AI usage is generally encouraged at Pratishthan, this project specifically prohibits the use of AI tools. This is because the exercise aims to promote individual learning, improve documentation comprehension, strengthen conceptual understanding, and develop implementation capabilities. We trust that you will complete this exercise with integrity and abstain from using AI.

## 🎯 Objective of the project

 The central objective here is to solve a small real world problem, the solution should involve functionalities like asynchronous and parallel processing(use of tool like rabbitmq), caching techniques, and relational database persistence.
 The total timeline for this exercise is 3 weeks, so choose your problem statement wisely.


## Size of the project

   There is no constraint on the size of the project as long as it meets the objective and can be done in the above mentioned timeline

## Must use tech stack

- Loopback 3, oe-cloud
- RabbitMq
- Redis
- An RDBMS databse (like postgres)

## Steps and Timeline

- **Day 1** ETC - 1 day
  - Problem statement decision 
- **Day 2** ETC - 2 days
  - Design
        This involves explaning the problem statement, services involved, their interactions and db design
- **Day 4** ETC - 10 day
  - Implementation
        Adherence to standard coding practices is essential during this process
- **Day 5** ETC - 2 day
  - Unit cases
        Writing of unit test case are essential part of development, to ensure bug free code.

## Resources

    Refer to index.md     

## example

   Enhanced example: Dark Store Replenishment POC using RabbitMQ and Redis

   Imagine an organisation that operates multiple dark stores to serve specific nearby areas. Each dark store holds minimal stock and is replenished by a central warehouse. Your POC should demonstrate how low-stock detection and replenishment requests happen asynchronously and reliably using RabbitMQ, while read-heavy operations (e.g., checking store inventory for orders) are accelerated using Redis as a cache. A relational database (e.g., Postgres) remains the system of record.

   Goals of the POC (must-haves):
   -   Use RabbitMQ for async events and background processing (e.g., stock-low alerts, replenishment jobs).
   -   Use Redis for caching hot reads (e.g., store inventory lookups, product availability checks).
   -   Persist authoritative data in RDBMS (Loopback 3/oe-cloud backed models).
   -   Demonstrate idempotency, retries, and failure handling in consumers.

   See the accompanying flow diagram: `payments/dark-store-flow.drawio`.

### Actors and services

   -   Store API (Loopback 3/oe-cloud): CRUD for stores, products, inventory; exposes endpoints to place an order and to adjust inventory.
   -   Inventory Worker: Background worker subscribed to RabbitMQ queues; processes low-stock events and creates replenishment requests.
   -   Warehouse Worker: Consumes replenishment requests, updates shipment status, and confirms receipt back to the store.
   -   Redis: Cache layer for product availability per store (cache-only; no idempotency keys or locks).
   -   Postgres (or equivalent RDBMS): Source of truth for orders, inventory, store-product mappings, and shipment records.
   -   RabbitMQ: Event backbone with topic exchange(s) and durable queues.

### Suggested data model (minimal)

   -   store(id, name, service_area)
   -   product(id, sku, name)
   -   inventory(store_id, product_id, on_hand, reserved, reorder_threshold)
   -   order(id, store_id, status, created_at)
   -   order_item(order_id, product_id, qty)
   -   replenishment_request(id, store_id, product_id, qty, status)

### RabbitMQ topology

   Exchanges (topic):
   -   inventory.events
       -   Routing keys: `stock.low`, `stock.replenished`
   -   order.events
       -   Routing keys: `order.created`, `order.cancelled`
   -   warehouse.events
       -   Routing keys: `replenishment.created`, `shipment.dispatched`, `shipment.received`

   Queues (durable):
   -   q.inventory.low-stock -> bound to `inventory.events:stock.low`
   -   q.warehouse.replenish -> bound to `warehouse.events:replenishment.created`
   -   q.inventory.stock-updates -> bound to `inventory.events:stock.replenished`

   Consumer responsibilities:
   -   Inventory Worker consumes `q.inventory.low-stock` to open replenishment requests.
   -   Warehouse Worker consumes `q.warehouse.replenish` to dispatch shipments and emit `shipment.dispatched`.
   -   Store API (or a light worker) consumes `shipment.received` to update inventory and cache.

### Redis usage (cache-only)

   -   Read cache: `inv:{storeId}:{productId} -> {onHand, reserved, threshold}` with a short TTL.

### Core flows

   1. Order placement (fast path with cache)
      -   Client calls Store API: POST /orders with items and target store.
      -   API reads availability from Redis; on cache miss, read DB, backfill Redis with short TTL.
      -   If sufficient stock, reserve items (DB transaction + optionally update cache), publish `order.events:order.created`.
      -   If reservation pushes on_hand near or below threshold, publish `inventory.events:stock.low` with a correlation id.

   2. Low stock -> replenishment (async path)
      -   Inventory Worker consumes `stock.low`, creates a `replenishment_request` in DB if not already open (idempotent via DB constraints), then publishes `warehouse.events:replenishment.created`.
      -   Warehouse Worker consumes `replenishment.created`, plans quantity, writes shipment record, publishes `warehouse.events:shipment.dispatched`.
      -   Upon delivery confirmation (simulated in POC), publish `inventory.events:stock.replenished` which updates DB on_hand and invalidates Redis keys for affected products/store.

   3. Cache invalidation
      -   Any inventory write (reservation, replenishment) invalidates or updates the relevant `inv:{storeId}:{productId}` key.

### Reliability & observability

   -   Use durable queues, persistent messages, dead-letter exchanges for poison messages.
   -   Consumers must be idempotent (use DB unique constraints/upserts; no Redis-based idempotency).
   -   Include basic metrics/logs: counts of events processed, retry counts, cache hit rate.

### Non-goals for the POC

   -   Real payment integration, full auth, multi-warehouse optimization. Keep scope tight to showcase async processing + caching.

### Milestones (mapping to timeline)

   -   Design: Model schema, queue topology, cache keys, sequence diagrams.
   -   Implementation: Store API, workers, RabbitMQ bindings, Redis cache, DB migrations.
   -   Tests: Unit tests for consumers/producers, cache behavior, and DB writes.

## Services

    This can have following services
    1.  Store API service
    Exposes order placement and inventory endpoints; produces `order.created`, `stock.low`; consumes `shipment.received` and `stock.replenished`.

    2.  Inventory Worker service
    Subscribes to `q.inventory.low-stock`; creates replenishment requests and publishes `replenishment.created`.

    3.  Warehouse Worker service
    Subscribes to `q.warehouse.replenish`; simulates dispatch and delivery; publishes `shipment.dispatched` and `stock.replenished`.

    4.  Support services
    Redis (cache-only) and Postgres (persistence).

   The flow diagram for the above can be found at `payments/dark-store-flow.drawio`.

 `payments/dark-store-flow.png`
