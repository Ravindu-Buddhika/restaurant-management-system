# Monolithic vs Microservices Architecture — Concept Summary

## 1. What is Monolithic Architecture?

**Core idea:** All the logic, modules, and features of a system (POS, Inventory, Self-Checkout, Online Ordering) are written inside **one single project (codebase)** and share **one central database**.

**How this applies to your scenario:**
- The backend is a single Spring Boot application.
- There is one central database.
- All client apps — the Tauri POS app, the Kiosk app, and the React web view — are built and run separately, but they all talk to the same Spring Boot backend's APIs.

## 2. What is Microservices Architecture?

**Core idea:** Each service in the system (e.g., POS Service, Inventory Service, Payment Service, Online Order Service) is built as a **fully independent project**, with its own separate Git repository and its own separate database (database-per-service).

**How services communicate:**
Since each service has its own database, they sync data over the network — either through direct **REST API (HTTP) calls**, or through **event brokers / message queues** like Kafka or RabbitMQ.

## 3. Monolithic vs Microservices — Comparison

| Feature | Monolithic Architecture | Microservices Architecture |
|---|---|---|
| **Complexity** | Much lower; easy to manage. | Much higher (needs API Gateways, Service Discovery, etc.). |
| **Database Transactions** | Easy to maintain data integrity using standard ACID transactions, since there's one DB. | Since each service has its own DB, rollback requires the Saga Pattern. |
| **Development Speed** | Faster initially, since it's one repo. | Multiple teams can work independently, which speeds up parallel development. |
| **Scaling** | The entire system has to be scaled together. | Only the specific service that needs it can be scaled independently. |

## 4. How Does a Monolith Scale to 10 Servers?

The process when scaling a monolith for heavy traffic:

```
                        ┌──> Server 1 (Spring Boot App) ──┐
                        ├──> Server 2 (Spring Boot App) ──┤
[User Requests] ──> [Load Balancer]                       ├──> [Single Central DB]
                        ├──> ...                          │
                        └──> Server 10 (Spring Boot App) ─┘
```

- **Running 10 instances:** The same Spring Boot code runs as 10 identical copies across 10 servers.
- **Load Balancer:** A load balancer in front distributes incoming traffic across these 10 servers.
- **Failover:** If one server goes down, the load balancer routes traffic to the other 9, so the system never goes fully down.
- **Stateless Backend:** Since a request can land on any server, **JWT (JSON Web Tokens)** are used to handle login/session state instead of relying on server memory.

## 5. How Do You Prevent the Database From Crashing?

Two industry-standard techniques to prevent the central database from crashing when thousands of connections hit it at once:

**a) HikariCP (Connection Pooling)**
- Instead of letting the backend open thousands of connections directly to the DB, a **pool of 10–20 pre-created connections** is maintained.
- Incoming requests reuse these pooled connections and are queued when all are busy, keeping the load controlled.

**b) Read Replicas (Primary / Replica DBs)**
- The database is split into two roles:
  - **Primary DB:** Handles only write/update operations — placing orders, updating stock, etc.
  - **Read Replicas (Secondary DBs):** Handle read operations — viewing menus, searching orders — which typically make up **~90% of total traffic**.
- This significantly reduces the load on the primary DB and prevents it from crashing.

## 6. Conclusion

> **"Start with a Monolith, move to Microservices only when you need to scale."**

Even large companies like Uber, Amazon, and Netflix started out as monoliths.

For your restaurant system — built in phases (Phase 1: POS, Phase 2: Self-Checkout, Phase 3: Online Ordering) — a **Modular Monolith** approach is the most practical and successful way to move forward phase by phase.
