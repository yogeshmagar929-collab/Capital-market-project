# Capital-market-project
Capital market Domain project
# Capital Market Order Management System

A production-style **Java Spring Boot** application covering the full trade lifecycle in capital markets — from order placement to settlement.

---

## Architecture Overview

```
Order Placement → Validation → Order Matching → Trade Execution → Clearing → Settlement
```

### Base Class Hierarchy

```
BaseEntity          (id, createdAt, updatedAt)
    └── AuditableEntity  (createdBy, updatedBy, version, isDeleted, tenantId)
            ├── Order
            ├── Trade
            └── Settlement

BaseService<T,ID>   (findById, findAll, findAllPaged, save, deleteById, count)
    ├── OrderService
    ├── TradeService
    └── SettlementService

BaseController      (ok, created, paged, list, notFound, badRequest, noContent)
    ├── OrderController
    └── TradeAndSettlementController
```

---

## Tech Stack

| Layer        | Technology                        |
|--------------|-----------------------------------|
| Language     | Java 17                           |
| Framework    | Spring Boot 3.2                   |
| ORM          | Spring Data JPA / Hibernate       |
| Database     | H2 (dev) / MySQL (prod)           |
| Build        | Maven                             |
| Testing      | JUnit 5, Spring Boot Test         |
| Utilities    | Lombok                            |

---

## Project Structure

```
src/main/java/com/capitalmarket/
├── base/
│   ├── BaseEntity.java           ← id, createdAt, updatedAt
│   ├── AuditableEntity.java      ← + audit trail, soft delete
│   ├── BaseService.java          ← generic CRUD with logging
│   └── BaseController.java       ← standard response helpers
│
├── model/
│   ├── Order.java                ← extends AuditableEntity
│   ├── Trade.java                ← extends AuditableEntity
│   └── Settlement.java           ← extends AuditableEntity
│
├── service/
│   ├── OrderService.java         ← extends BaseService<Order, String>
│   ├── TradeService.java         ← extends BaseService<Trade, String>
│   └── SettlementService.java    ← extends BaseService<Settlement, String>
│
├── controller/
│   ├── OrderController.java      ← extends BaseController
│   └── TradeAndSettlementController.java
│
├── repository/
│   ├── OrderRepository.java
│   ├── TradeRepository.java
│   └── SettlementRepository.java
│
├── dto/
│   └── PlaceOrderRequest.java
│
├── response/
│   ├── ApiResponse.java          ← standard JSON envelope
│   └── PagedResponse.java        ← paginated response wrapper
│
├── exception/
│   ├── GlobalExceptionHandler.java ← catches all, returns clean JSON
│   ├── ResourceNotFoundException.java
│   ├── BusinessRuleException.java
│   └── OrderValidationException.java
│
├── config/
│   ├── JpaAuditingConfig.java    ← enables @CreatedBy, @CreatedDate
│   └── DataInitializer.java      ← seeds demo data on startup
│
└── enums/
    ├── OrderStatus.java
    ├── OrderType.java
    ├── OrderSide.java
    └── TradeStatus.java
```

---

## Quick Start

```bash
# Clone
git clone https://github.com/your-username/capital-market-system.git
cd capital-market-system

# Run
mvn spring-boot:run

# H2 Console (dev database browser)
open http://localhost:8080/h2-console
# JDBC URL: jdbc:h2:mem:capitalmarketdb
```

---

## API Reference

All responses follow the standard envelope:
```json
{
  "success": true,
  "message": "Found 3 records",
  "data": [ ... ],
  "timestamp": "2024-01-15T09:30:00"
}
```

### Orders  `/api/v1/orders`

| Method | Endpoint                         | Description                    |
|--------|----------------------------------|--------------------------------|
| POST   | `/api/v1/orders`                 | Place a new order              |
| GET    | `/api/v1/orders?page=0&size=20`  | Get all orders (paginated)     |
| GET    | `/api/v1/orders/{id}`            | Get order by ID                |
| GET    | `/api/v1/orders/trader/{id}`     | Orders by trader               |
| GET    | `/api/v1/orders/symbol/{sym}`    | Orders by symbol               |
| GET    | `/api/v1/orders/status/{status}` | Orders by status               |
| DELETE | `/api/v1/orders/{id}/cancel`     | Cancel order (pre-execution)   |
| POST   | `/api/v1/orders/match/{symbol}`  | Run matching engine for symbol |

### Trades  `/api/v1/trades`

| Method | Endpoint                         | Description          |
|--------|----------------------------------|----------------------|
| GET    | `/api/v1/trades`                 | All trades (paged)   |
| GET    | `/api/v1/trades/{id}`            | Trade by ID          |
| GET    | `/api/v1/trades/symbol/{sym}`    | Trades by symbol     |
| GET    | `/api/v1/trades/status/{status}` | Trades by status     |
| GET    | `/api/v1/trades/buyer/{id}`      | Trades by buyer      |

### Settlements  `/api/v1/settlements`

| Method | Endpoint                              | Description              |
|--------|---------------------------------------|--------------------------|
| POST   | `/api/v1/settlements/clear/{tradeId}` | CCP clears a trade       |
| POST   | `/api/v1/settlements/settle/{tradeId}`| Settle a cleared trade   |
| POST   | `/api/v1/settlements/pipeline`        | Run full EOD pipeline    |
| GET    | `/api/v1/settlements`                 | All settlements (paged)  |
| GET    | `/api/v1/settlements/trade/{tradeId}` | Settlement by trade      |

---

## Example: Full Lifecycle

```bash
# 1. Place a BUY order
curl -X POST http://localhost:8080/api/v1/orders \
  -H "Content-Type: application/json" \
  -d '{"symbol":"RELIANCE","side":"BUY","type":"LIMIT","quantity":100,"price":2500.00,"traderId":"ALICE"}'

# 2. Place a matching SELL order
curl -X POST http://localhost:8080/api/v1/orders \
  -H "Content-Type: application/json" \
  -d '{"symbol":"RELIANCE","side":"SELL","type":"LIMIT","quantity":100,"price":2500.00,"traderId":"BOB"}'

# 3. Run matching engine
curl -X POST http://localhost:8080/api/v1/orders/match/RELIANCE

# 4. Get the trade ID from step 3, then clear it
curl -X POST http://localhost:8080/api/v1/settlements/clear/{tradeId}

# 5. Settle the trade
curl -X POST http://localhost:8080/api/v1/settlements/settle/{tradeId}

# OR run the full pipeline in one shot:
curl -X POST http://localhost:8080/api/v1/settlements/pipeline
```

---

## Order Status Lifecycle

```
NEW → VALIDATED → MATCHED → EXECUTED
        ↓              ↓
    REJECTED       CANCELLED
```

## Trade Status Lifecycle

```
EXECUTED → CLEARED → SETTLED
                ↓
             FAILED
```

---

## Key Design Patterns

**DRY via inheritance** — `BaseEntity`, `AuditableEntity`, `BaseService`, `BaseController` eliminate all boilerplate repetition across entities and layers.

**Soft delete** — financial records are never physically deleted. `AuditableEntity.softDelete()` sets `isDeleted = true` for full regulatory audit trail.

**Optimistic locking** — `@Version` on `AuditableEntity` prevents two concurrent updates from overwriting each other (critical in high-frequency trading scenarios).

**Standard response envelope** — every API response uses `ApiResponse<T>`, so the frontend always knows the shape of the data.

**Typed exceptions** — `ResourceNotFoundException (404)`, `BusinessRuleException (422)`, `OrderValidationException (400)` give precise HTTP status codes with `GlobalExceptionHandler`.

---

## Running Tests

```bash
mvn test
```

Tests cover: order placement, LIMIT validation, cancel authorization, price-time matching, lifecycle from placement to settlement, and error cases.
