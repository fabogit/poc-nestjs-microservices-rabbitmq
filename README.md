# poc-nestjs-microservices-rabbitmq

> An enterprise microservices monorepo orchestrating Orders, Billing, and Authentication services using NestJS, RabbitMQ message queues, and a MongoDB replica set.

**Inception (First Commit):** `2024-03-29` • **Batch / Horizon:** `batch-2024-h1` • **Architecture Domain:** `Distributed Message Queue / RabbitMQ Microservices`

---

## 📌 Architecture & Overview

`nestJs-microservices-rabbitMQ` demonstrates a distributed, event-driven microservices architecture structured as a monorepo. It isolates business domains into autonomous microservices that communicate asynchronously using RabbitMQ message queues and synchronously using HTTP/JWT validation.

The system topology incorporates 3 autonomous microservices and a shared library:
1. **Orders Service (`apps/orders`):** Public-facing HTTP REST API (port 3000) that accepts order placements, verifies authentication tokens with `apps/auth`, persists orders in MongoDB, and dispatches `order_created` events to the `BILLING` RabbitMQ queue.
2. **Billing Service (`apps/billing`):** Asynchronous background worker consuming events from the `BILLING` queue, performing billing verification, and completing manual AMQP message acknowledgments.
3. **Auth Service (`apps/auth`):** Security microservice (port 3001) managing user registration, credential validation, Passport local/JWT strategies, and cookie-based identity tokens.
4. **Common Library (`libs/common`):** Shared infrastructure layer encapsulating `RmqModule`, `RmqService` (with manual acknowledgment helpers), database connection providers, and base schemas.

```
+---------------------------------------------------------------------------------------+
|                            nestJs-microservices-rabbitMQ                              |
|                                                                                       |
|  [ HTTP Client / Postman ]                                                            |
|         |                                                                             |
|         | POST /orders (Bearer JWT Cookie)                                            |
|         v                                                                             |
|  +--------------------+       Sync HTTP Auth Guard (Validate User)                    |
|  | Orders Service     +----------------------------------------+                      |
|  | (Port: 3000)       |                                        |                      |
|  +---------+----------+                                        v                      |
|            |                                        +--------------------+            |
|            | 1. Persist Order                       | Auth Service       |            |
|            v                                        | (Port: 3001)       |            |
|  +--------------------+                             +---------+----------+            |
|  | MongoDB Replica    |                                       |                       |
|  | Set (Primary/Sec)  |                                       v                       |
|  +--------------------+                             +--------------------+            |
|            |                                        | Users Collection   |            |
|            | 2. Publish 'order_created'             +--------------------+            |
|            v                                                                          |
|  +--------------------+                                                               |
|  | RabbitMQ Broker    |                                                               |
|  | (Queue: 'BILLING') |                                                               |
|  +---------+----------+                                                               |
|            |                                                                          |
|            | 3. Consume Message (Manual ACK)                                          |
|            v                                                                          |
|  +--------------------+                                                               |
|  | Billing Service    | (Processes invoice, executes rmqService.ack(context))         |
|  | (Background Worker)|                                                               |
|  +--------------------+                                                               |
+---------------------------------------------------------------------------------------+
```

* **Context:** Distributed microservices reference architecture evaluating RabbitMQ transport decoupling, manual AMQP message acknowledgment lifecycles, and MongoDB replica set high availability.
* **Key Patterns:**
  * **Asynchronous Message Queuing:** Offloading long-running billing operations from the critical HTTP request path via RabbitMQ broker queues.
  * **Manual Acknowledgment (ACK) Semantics:** Disabling auto-ack to ensure messages are re-queued if a billing worker crashes mid-transaction.
  * **Shared Monorepo Infrastructure:** Reusable `RmqModule` providing configurable connection management via `amqp-connection-manager`.
  * **High-Availability Database Topology:** Containerized MongoDB 8.0 cluster configured with Primary, Secondary, and Arbiter nodes.

---

## 🛠 Tech Stack

* **Microservices Framework:** NestJS 8.4 (`@nestjs/microservices`, `@nestjs/core`, `@nestjs/common`)
* **Message Broker:** RabbitMQ 4 (AMQP 0-9-1 via `amqplib` 0.9.1 and `amqp-connection-manager` 4.1.14)
* **Database & Cluster:** MongoDB 8.0 Replica Set (Primary, Secondary, Arbiter via Bitnami), Mongoose 9.2.2
* **Security & Identity:** `@nestjs/jwt` 8.0.1, `@nestjs/passport` 8.2.2, `passport-jwt`, `passport-local`, `bcrypt`
* **Orchestration & Tooling:** Docker Compose, pnpm, Postman Collection

---

## ✨ Features & Microservice Roles

* **Orders Service:** HTTP endpoint receiving orders with payload validation (`class-validator`) and RabbitMQ event emission.
* **Billing Worker:** Consumer service listening on the `BILLING` queue with explicit context acknowledgment.
* **Authentication Subsystem:** User registration, credential verification, and JWT cookie parsing.
* **Complete Multi-Node Cluster:** Production-like `docker-compose.yml` defining full service topologies with network bridges.

### Inter-Service Communication Interfaces

| Source | Target | Protocol | Pattern / Queue | Payload / Event |
|---|---|---|---|---|
| Client | `orders` | `HTTP POST` | `/orders` | Order creation request (`CreateOrderDto`) |
| Client | `auth` | `HTTP POST` | `/auth/login` | User credentials (`email`, `password`) |
| `orders` | `auth` | `HTTP GET` | `/auth/user` | Synchronous cookie validation check |
| `orders` | `billing` | `AMQP` | Queue: `BILLING` | Event: `order_created` (`Order` payload) |

---

## 🚀 Getting Started

### Prerequisites

* Docker & Docker Compose
* Node.js `>= 18.0.0`
* pnpm (`npm install -g pnpm`)

### Installation

```bash
# Clone the repository
git clone https://github.com/fabogit/poc-nestjs-microservices-rabbitmq.git
cd poc-nestjs-microservices-rabbitmq

# Install monorepo dependencies
pnpm install
```

### Environment Configuration

Configure environment files based on provided templates:

```bash
cp apps/auth/.env.auth.template apps/auth/.env
cp apps/orders/.env.orders.template apps/orders/.env
cp apps/billing/.env.billing.template apps/billing/.env
```

### Running the Complete Cluster via Docker Compose

```bash
# Launch RabbitMQ, MongoDB Replica Set, Auth, Orders, and Billing services
docker compose up --build
```

### Testing the Order & Billing Pipeline

Execute the preconfigured Postman collection [`Nestjs-rabbitMQ.postman_collection.json`](./Nestjs-rabbitMQ.postman_collection.json) or run cURL:

```bash
# 1. Register a user
curl -X POST http://localhost:3001/auth/users \
  -H "Content-Type: application/json" \
  -d '{"email": "developer@test.com", "password": "Password123!"}'

# 2. Place an order (triggers RabbitMQ event to Billing)
curl -X POST http://localhost:3000/orders \
  -H "Content-Type: application/json" \
  -d '{"name": "MacBook Pro", "price": 2499, "phoneNumber": "+1234567890"}'
```

---

## 📄 License & Author

Developed by [@fabogit](https://github.com/fabogit).
