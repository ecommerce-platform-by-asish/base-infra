# 🚀 eCommerce Learning Guide

A step-by-step action plan to build the entire eCommerce Microservices platform. Built on the philosophy of starting simple and adding one major concept per phase.

---

## 🧰 Prerequisites & Setup

- **Tools:** Spring Boot 4+, Java 25, Gradle, Docker, Postman/curl.
- **Root Setup:** The stack uses `docker-compose.yml` to spin up PostgreSQL, Redis, and Kafka. Run `cd .github && docker compose up` before starting.
- **Fast Testing:** Use `WireMock` JSON stubs in `src/test/resources/mappings` to immediately mock downstream dependencies rather than waiting for them to be built.

---

## ⭐ Phase 1 — Product Service (Foundation)
**Goal:** Build a robust, clean REST API foundation.
**Tech:** Java 25, Spring Boot 4.0, MapStruct, Lombok, JPA/PostgreSQL.

**Action Items:**
- [ ] Implement robust RESTful CRUD endpoints for Products.
- [ ] Use `MapStruct` with `@Mapper(componentModel = "spring")` to eliminate manual data class mapping.
- [ ] Connect to PostgreSQL on port `5432`.

---

## ⭐ Phase 2 — Cart Service & Redis Caching
**Goal:** Optimize read performance and manage volatile state using Redis.
**Tech:** Redis Hash (`HSET`), Cache-Aside Pattern.

**Action Items:**
- [ ] Update Product Service to cache responses in Redis. Implement **Cache-Aside Invalidation** (evict cache on updates).
- [ ] Create **Cart Service** on Port `8082`.
- [ ] Store entire shopping carts exclusively in Redis using `HSET` (No PostgreSQL).

---

## ⭐ Phase 3 — Auth Service & API Gateway
**Goal:** Secure the platform and centralize routing.
**Tech:** JWT (RS256 asymmetric signing), Spring Cloud Gateway, Rate Limiting.

**Action Items:**
- [ ] Create **Auth Service** on Port `8080`. Implement JWT issuance via Async RSA Keys.
- [ ] Create **API Gateway** on Port `8000` to route traffic.
- [ ] Gateway verifies JWTs locally via Public Key and passes the extracted `X-User-Id` header to downstream services.

---

## ⭐ Phase 4 — Pricing Service & gRPC
**Goal:** Implement lightning-fast, synchronous internal communication.
**Tech:** gRPC, Protocol Buffers.

**Action Items:**
- [x] Create **Pricing Service** on Port `9090` exporting gRPC endpoints.
- [x] Define `.proto` contracts for calculating real-time tax and discounts.
- [x] Update Cart Service to calculate the final total by querying the Pricing Service via gRPC.

---

## ⭐ Phase 5 — Resilience Foundation (Sync Calls)
**Goal:** Prevent cascading failures before moving to distributed asynchronous architecture.
**Tech:** Resilience4j, Circuit Breakers, Retries.

**Action Items:**
- [x] Inject **Circuit Breakers** and **Retries** into all outbound gRPC/REST calls (e.g., Cart -> Pricing via \`@CircuitBreaker\`).
- [x] Consolidate Global Exception Handling and MDC trace logging.

---

## ⭐ Phase 6 — Order Orchestration (Saga Phase 1)
**Goal:** Transition to event-driven eventual consistency.
**Tech:** Kafka Topics, Saga Pattern (State Machine).

**Action Items:**
- [ ] Create **Order Service** on Port `8083` with PostgreSQL for state tracking (PENDING -> RESERVED -> PAID).
- [ ] Publish `ecommerce.orders.requested` to Kafka when a cart is checked out.
- [ ] Create **Inventory Service** (Port `8084`) that listens for the event and attempts to logically reserve stock.

---

## ⭐ Phase 7 — Payment Service (Saga Phase 2)
**Goal:** Handle failures using Compensating Transactions and Outbox.
**Tech:** Transactional Outbox Pattern, Kafka.

**Action Items:**
- [ ] Ensure Outbox Pattern is used for all DB writes + Kafka publishes in the same transaction.
- [ ] Create **Payment Service** to simulate charge requests and emit success/failure.
- [ ] If payment fails, Order Service must publish a `RELEASE_STOCK_CMD` to revert inventory.

---

## ⭐ Phase 8 — Saga Watchdogs & Recovery
**Goal:** Handle stuck asynchronous workflows and prevent distributed deadlocks.
**Tech:** Spring Scheduled Jobs.

**Action Items:**
- [ ] Build a **Saga Watchdog** scheduled job in the Order Service to auto-cancel sagas stuck in `PENDING` for over 5 minutes.

---

## ⭐ Phase 9 — Search Service & Elasticsearch
**Goal:** Implement real-time, fuzzy full-text search capability.
**Tech:** Elasticsearch, Change Data Capture (CDC).

**Action Items:**
- [ ] Create **Search Service** on Port `8085`.
- [ ] Subscribe to the `PRODUCT_UPDATED_EVT` Kafka topic to immediately update the Elasticsearch index, keeping search in sync with PostgreSQL.

---

## 🧭 Project Ports Quick-Reference

| Service         | Port | Service         | Port |
|-----------------|------|-----------------|------|
| API Gateway     | 8000 | Pricing (gRPC)  | 9090 |
| Auth Service    | 8080 | PostgreSQL      | 5432 |
| Product Service | 8081 | Redis           | 6379 |
| Cart Service    | 8082 | Kafka           | 9092 |
| Order Service   | 8083 | Elasticsearch   | 9200 |
