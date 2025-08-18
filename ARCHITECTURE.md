# 🛒 eCommerce Platform — Architecture

> **Stack**: Java 25 · Spring Boot 4 · Spring Cloud Gateway · gRPC · Kafka · Redis · PostgreSQL · Elasticsearch
> **Patterns**: DDD · Saga (Orchestration) · CQRS · Event Sourcing · Outbox · Circuit Breaker · DLQ

---

## Architecture

```
                                          👤 CLIENTS
                                  ┌────────────────────────┐
                                  │  Web, Mobile API Apps  │
                                  └───────────┬────────────┘
                                              │
                                              ▼
                             🛡️ API GATEWAY (Spring Cloud)                                      👤 Auth Service
                    ┌────────────────────────────────────────────────┐                    ┌────────────────────────┐
                    │ Pattern: Rate Limit, Circuit, API Composition  │──🌐 REST (:8080)──▶│ Pattern : ID Provider  │
                    │ Storage: Redis (Token Bucket & Session Cache)  │                    │ Storage : PGSQL (ACID) │
                    │ Purpose: Central Routing & Edge JWT Validation │                    │ Purpose : Issue Tokens │
                    └────────┬───────────────────────────────┬───────┘                    └────────────────────────┘
                             │                               │
                      🌐 REST (:8081)                 🌐 REST (:8082)
                             ▼                               ▼
                   📦 Product Service                    🛒 Cart Service                            🏷️ Pricing Service 
                 ┌────────────────────────┐          ┌────────────────────────┐                  ┌────────────────────────┐
                 │ Pattern : CQRS & Cache │          │ Pattern : Ephemeral KV │──⚡ gRPC (:9090)─▶│ Pattern : Service/RPC  │
                 │ Storage : PGSQL+Redis  │          │ Storage : Redis (Hash) │                  │ Storage : In-Memory    │
                 │ Purpose : Product CRUD │          │ Purpose : Manage Carts │                  │ Purpose : Batch Prices │
                 └───────────┬────────────┘          └───────────┬────────────┘                  └────────────────────────┘
                             │                                   │  ORDER_REQUESTED
               ┌─────────────┘                                   │
          🔄 CDC (:8089)                            📨 Kafka (orders.requested)
               ▼                                                 ▼
       🔍 Search Service                         ⚙️ Order Orchestrator (Saga)
     ┌────────────────────────┐           ┌──────────────────────────────────────────────┐
     │ Pattern : CQRS (Read)  │           │ Pattern: Saga Orchestrator & Event Sourcing  │
     │ Storage : Elasticsearch│           │ Storage: PGSQL (Immutable Event Store)       │
     │ Purpose : Fast Search  │           │ Purpose: Manages 5-step distributed tx flow  │
     └────────────────────────┘           └───────────────────────┬───────────────────────┘
                                                                  │
                  ┌────────────────────────────┬──────────────────┴──────────┬─────────────────────────────┐
                  │                            │                             │                             │
    📨 Kafka (inventory.reserve)  📨 Kafka (payments.charge)   📨 Kafka (shipments.create)  📨 Kafka (notifications.send)
                  ▼                            ▼                             ▼                             ▼
        🏭 Inventory Service              💳 Payment                    🚚 Shipping                   🔔 Notification
      ┌────────────────────────┐      ┌──────────────────┐          ┌──────────────────┐          ┌────────────────────────┐
      │ Pattern : Event Outbox │      │ Pattern : Outbox │          │ Pattern : Outbox │          │ Pattern : Async Worker │
      │ Storage : PGSQL+Redis  │      │ Storage : PGSQL  │          │ Storage : PGSQL  │          │ Storage : None         │
      │ Purpose : Reserve Stock│      │ Purpose : Pay    │          │ Purpose : Ship   │          │ Purpose : Notifications│
      └────────────────────────┘      └──────────────────┘          └─────────┬────────┘          └────────────────────────┘
                                                                              │
                                                                     📨 Kafka (orders.confirmed)
                                                                              ▼
                                                                      ⭐ Review & Rating
                                                                    ┌────────────────────────┐
                                                                    │ Pattern : CQRS + Outbox│
                                                                    │ Storage : PGSQL+Elastic│
                                                                    │ Purpose : User Reviews │
                                                                    └────────────────────────┘
```

---

## Communication Rules

| Caller → Callee | Protocol | Why |
|---|---|---|
| Client → Gateway | HTTPS/REST | External entry point |
| Gateway → Services | REST | Internal routing |
| Cart → Pricing | **gRPC** | Low-latency sync call at cart view |
| Cart → Inventory | **gRPC** | Stock check before checkout |
| Order Orch → All | **Kafka** | Async, durable saga steps |
| Product → Search | **Kafka (CDC)** | Real-time index sync |

> **Rule**: gRPC for mandatory sync calls. Kafka for everything async. REST for reads.

---

## Auth & JWT Flow

```
POST /api/auth/register  →  BCrypt hash + save to PostgreSQL
POST /api/auth/login     →  Validate credentials → issue RS256-signed JWT

API Gateway (every request):
  1. Validate JWT signature (RS256 public key — no Auth Service call)
  2. Inject X-User-Id + X-User-Role headers downstream
  3. Downstream services trust these headers directly
```

**Planned**: Refresh tokens · Redis JWT blacklist for logout · OAuth2 (Social Login)

---

## gRPC Contracts (`shared-grpc-contracts`)

A standalone Gradle library. Neither client nor server — just the shared contract.

```protobuf
service PricingService {
  rpc GetBatchPrices (BatchPriceRequest) returns (BatchPriceResponse);
}

message BatchPriceRequest  { repeated string productIds = 1; }
message BatchPriceResponse { map<string, double> prices  = 1; }
```

Cart Service calls this stub → Pricing Service implements it → both compile against the same `.proto`.

---

## Saga Pattern (Order Flow)

```
Cart Service  ──[Kafka]──▶  ORDER_REQUESTED_EVT
                                    │
                          Order Orchestrator
                           ├─1▶ RESERVE_INVENTORY  ──▶ Inventory Service
                           ├─2▶ PROCESS_PAYMENT    ──▶ Payment Service
                           ├─3▶ CONFIRM_ORDER       (self)
                           ├─4▶ CREATE_SHIPMENT    ──▶ Shipping Service
                           └─5▶ SEND_NOTIFICATION  ──▶ Notification Service

On failure → compensating transactions run in reverse order
On timeout → watchdog triggers compensation after 15 min
```

---

## Redis Usage

| Use Case | Service | Key Pattern |
|---|---|---|
| Product cache | Product Service | `product:{id}` (1hr TTL) |
| Cart storage | Cart Service | `HSET cart:{userId} {productId} {qty}` |
| Rate limiting | API Gateway | Token bucket per IP |
| JWT blacklist | API Gateway *(planned)* | `jwt:blacklist:{jti}` |
| Distributed lock | Inventory *(planned)* | `lock:inventory:{itemId} NX EX 5` |
| Idempotency | Kafka consumers *(planned)* | `idempotency:{eventId}` (7d TTL) |

---

## Kafka Topics

```
ecommerce.orders.requested          # Cart → Orchestrator
ecommerce.orders.confirmed
ecommerce.orders.failed

ecommerce.inventory.reserve-cmd     # Orchestrator → Inventory
ecommerce.inventory.reserved
ecommerce.inventory.release-cmd

ecommerce.payments.charge-cmd       # Orchestrator → Payment
ecommerce.payments.charged
ecommerce.payments.charge-failed

ecommerce.shipments.create-cmd      # Orchestrator → Shipping
ecommerce.shipments.delivered

ecommerce.notifications.send-cmd    # Fire & forget
ecommerce.search.product-sync       # CDC from Debezium → Search
```

---

## Key Design Decisions

| Pattern | Why |
|---|---|
| **Database per Service** | No cross-service JOINs. Each service owns its data. |
| **gRPC for Pricing** | Binary protocol, strongly typed, 3× faster than REST for batch calls. |
| **Kafka for Saga** | Async, fault-tolerant saga steps with natural replay and DLQ support. |
| **Outbox Pattern** | Business write + event in one DB transaction — no message loss. |
| **CQRS** | PostgreSQL for writes (normalised), Redis/Elasticsearch for reads (fast). |
| **Event Sourcing (Orders)** | Order changes are immutable events — full audit trail, replayable. |
| **Redis Cart** | Cart is ephemeral. Hash structure gives O(1) add/remove per item. |
| **Stateless Gateway** | JWT validated by public key — Gateway never calls Auth Service. |

---

## Future Enhancements
* **Service Discovery**: Eureka/Consul registry for dynamic service resolution.
* **Load Balancing**: Client-side round-robin load distribution at the Gateway level.
* **Config Server**: Centralized configuration management for all microservices.
* **Observability (Tracing)**: Zipkin/Jaeger for distributed request tracing across Saga transactions.
* **Observability (Metrics)**: Prometheus + Grafana dashboards for real-time JVM and Kafka monitoring.
* **Observability (Logging)**: ELK Stack (Elasticsearch, Logstash, Kibana) for centralized log aggregation.

*Design principles: **Domain-Driven Design (DDD)** · **Event-Driven Architecture** · **12-Factor App** · **Clean Architecture***
