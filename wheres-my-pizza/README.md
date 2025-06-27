# wheres-my-pizza

## Learning Objectives

- Message Queue Systems
- RabbitMQ Integration
- Concurrent Programming
- Microservices Architecture

## Abstract

In this project, you will build a distributed restaurant order management system. Using Go, you will create several microservices that communicate asynchronously via a RabbitMQ message broker, with order data persisted in a PostgreSQL database. This system will simulate a real-world restaurant workflow, from an order being placed via an API, to it being cooked by a kitchen worker, and finally its status being tracked. This project teaches a fundamental lesson in modern software engineering: think about the architecture first. Before writing a single line of code, you must design how services will interact, how data will flow, and how the system can scale.

## Context

> Good architecture makes the system easy to understand, easy to develop, easy to maintain, and easy to deploy. The ultimate goal is to minimize the lifetime cost of the system and to maximize programmer productivity.
>
> — Robert C. Martin (Uncle Bob)

Have you ever ordered a pizza through a delivery app and watched its status change from "Order Placed" to "In the Kitchen" and finally "Out for Delivery"? What seems like a simple status tracker is actually a complex dance between multiple independent systems. The web app where you place your order isn't directly connected to the tablet in the kitchen.

This is the power of microservices and message queues. The challenge is to create a reliable order processing system that can handle a high volume of orders without slowing down. A single, monolithic application would quickly become a bottleneck. Instead, we distribute the work. The `Order Service` takes your order, the `Kitchen Service` cooks it, and a `Notification Service` keeps you updated. They don't talk to each other directly; they pass messages through a central mailroom, RabbitMQ. This ensures that even if the kitchen is busy, the order service can still take new orders.

In this project, you will build the core of such a system. You will learn how to design services that have a single responsibility and how to orchestrate their collaboration to create a robust and scalable application.

**Message Queue Patterns**

A smart way to solve this type of problem is using message queue patterns. This approach views the system as a set of interacting services, where each service processes a specific type of message and passes the result further down the chain.

**Work Queue Pattern**
- One producer sends tasks to a queue
- Multiple consumers wait for the task to arrive, but only one receives it
- Each task is processed by exactly one consumer
- Provides load distribution among workers

**Publish/Subscribe Pattern**
- One publisher sends messages to all subscribers
- Multiple consumers receive copies of messages
- Used for notifications and state synchronization

**Routing Pattern**
- Messages are routed based on routing key
- Allows creating complex processing schemes
- Provides flexibility in defining recipients

## System Architecture Overview

Your application will consist of four main services, a database, and a message broker. They interact as follows:

```
                                +------------------+
                                |   PostgreSQL DB  |
                                | (Order Storage)  |
                                +--+-------------+-+
                                   ^             ^
           (Writes & Reads)        |             |        (Writes & Reads)
                                   |             |
+------------+        +----------+ v             v +---------------+
| HTTP Client|------->|  Order   |               | Kitchen       |
| (e.g. curl)|        |  Service |               | Service       |
+------------+        +----------+               +--+------------+
                         |                         ^
                         | (Publishes New Order)   | (Publishes Status Update)
                         v                         |
                   +-----+-------------------------+-----+
                   |                                     |
                   |         RabbitMQ Message Broker     |
                   |                                     |
                   +-------------------------------------+
                              |                      |
                              | (Status Updates)     | (Status Updates)
                              v                      v
                        +-----+-----------+    +-----+-----------+
                        | Notification    |    | Tracking        |
                        | Service         |    | Service         |
                        +-----------------+    +-----------------+
```

## Database Schema

### Orders Table
**Purpose**: Primary storage for all restaurant orders with complete order information
**Used by**: Order Service (insert), Kitchen Workers (status updates), Tracking Service (queries)

```sql
create table orders (
    "id"                uuid          primary key default gen_random_uuid(),
    "created_at"        timestamptz   not null    default now(),
    "updated_at"        timestamptz   not null    default now(),
    "number"            text          unique not null,
    "customer_name"     text          not null,
    "customer_type"     text          default 'regular',
    "type"              text          not null check (type in ('dine-in', 'takeout', 'delivery')),
    "table_number"      integer,
    "delivery_address"  text,
    "total_amount"      decimal(10,2) not null,
    "priority"          integer       default 1,
    "status"            text          default 'received',
    "processed_by"      text,
    "completed_at"      timestamptz
);
```

### Order Items Table
**Purpose**: Stores individual items within each order for detailed order composition
**Used by**: Order Service (insert items when creating orders), API queries for order details

```sql
CREATE TABLE order_items (
    "id"          uuid          primary key default gen_random_uuid(),
    "created_at"  timestamptz   not null    default now(),
    "order_id"    uuid          references orders(id),
    "name"        text          not null,
    "quantity"    integer       not null,
    "price"       decimal(8,2)  not null
);
```

### Order Status Log Table
**Purpose**: Audit trail for all status changes throughout order lifecycle
**Used by**: All services (insert status changes), Tracking Service (status history queries)

```sql
create table order_status_log (
    "id"          uuid          primary key default gen_random_uuid(),
    "created_at"  timestamptz   not null    default now(),
    order_id      uuid          references orders(id),
    "status"      text,
    "changed_by"  text,
    "changed_at"  timestamp     default current_timestamp,
    "notes"       text
);
```

### Workers Table
**Purpose**: Registry and monitoring of all kitchen workers and their current status
**Used by**: Kitchen Workers (registration and heartbeat), Tracking Service (worker monitoring)

```sql
create table workers (
    "id"                uuid        primary key default gen_random_uuid(),
    "created_at"        timestamptz not null    default now(),
    "name"              text        unique not null,
    "type"              text        not null,
    "status"            text        default 'online',
    "last_seen"         timestamp   default current_timestamp,
    "orders_processed"  integer     default 0
);
```

## RabbitMQ Configuration

### Exchanges and Queues Setup

**Exchange Types Explained:**

**Topic Exchange (`orders_topic`)**: Routes messages to queues based on pattern matching with routing keys. Used for flexible order routing where different order types and priorities can be routed to specialized queues.

**Fanout Exchange (`notifications_fanout`)**: Broadcasts all messages to every queue bound to it, ignoring routing keys. Used for notifications where all subscribers need to receive status updates regardless of their specific interests.

```
Exchanges:
├── orders_topic (type: topic, durable: true)
│   └── Routing Keys (examples):
│       ├── kitchen.dine-in.high
│       ├── kitchen.takeout.medium
│       └── kitchen.delivery.low
│
└── notifications_fanout (type: fanout, durable: true)
    └── Broadcasts to all subscribers

Queues:
├── kitchen_queue (durable: true, x-max-priority: 10)
│   └── Bound to orders_topic with routing key: kitchen.order
│   └── Read by: General kitchen workers
├── kitchen_dine_in_queue (durable: true, x-max-priority: 10)
│   └── Bound to orders_topic with routing key: kitchen.dine-in.*
│   └── Read by: Dine-in specialized workers
├── kitchen_takeout_queue (durable: true, x-max-priority: 10)
│   └── Bound to orders_topic with routing key: kitchen.takeout.*
│   └── Read by: Takeout specialized workers
├── kitchen_delivery_queue (durable: true, x-max-priority: 10)
│   └── Bound to orders_topic with routing key: kitchen.delivery.*
│   └── Read by: Delivery specialized workers
└── notifications_queue (durable: true, auto-delete: false)
    └── Bound to notifications_fanout
    └── Read by: Notification subscribers, customer apps
```

### Message Formats

#### Order Message
**Purpose**: Contains complete order information for kitchen processing
**Sent to**: Kitchen queues (kitchen_queue, kitchen_dine_in_queue, etc.)
**Read by**: Kitchen Workers
**Routing**: Through orders_topic exchange using routing keys like "kitchen.delivery.high"

```json
{
  "order_number": "ORD_20241216_001",
  "customer_name": "John Doe",
  "customer_type": "vip",
  "order_type": "delivery",
  "table_number": null,
  "delivery_address": "123 Main St, City",
  "items": [
    {
      "name": "Margherita Pizza",
      "quantity": 1,
      "price": 15.99
    }
  ],
  "total_amount": 15.99,
  "priority": 10
}
```

#### Status Update Message
**Purpose**: Notifies all interested parties about order status changes
**Sent to**: notifications_fanout exchange
**Read by**: Notification Subscribers
**Routing**: Broadcast to all queues bound to notifications_fanout exchange

```json
{
  "order_number": "ORD_20241216_001",
  "old_status": "received",
  "new_status": "cooking",
  "changed_by": "chef_mario",
  "timestamp": "2024-12-16T10:32:00Z",
  "estimated_completion": "2024-12-16T10:42:00Z"
}
```

## Logging Format

This structured logging format must be used by all services. Consistent logging is vital for debugging, monitoring, and auditing the system.

### JSON Log Format

All services must implement structured logging in JSON format.

**Mandatory Core Fields:**
The following keys must always be present in every log entry:
*   `timestamp`: The time the log entry was created.
*   `level`: The log level (e.g., INFO, ERROR).
*   `service`: The name of the service emitting the log (e.g., `order-service`, `kitchen-worker`).
*   `action`: A concise, machine-readable string describing the event (e.g., `order_received`, `db_error`).
*   `message`: A human-readable description of the event.
*   `hostname`: The hostname or unique identifier of the module emitting the log.
*   `request_id`: A unique identifier for correlating requests/operations across multiple services.

**Format Example:**
```json
{
  "timestamp": "2024-12-16T10:30:15.123Z",
  "level": "INFO",
  "service": "order-service",
  "version": "abcdef12345",
  "hostname": "order-service-789abc",
  "request_id": "a1b2c3d4e5f6",
  "action": "order_received",
  "message": "Order received and queued for processing",
  "worker_name": "chef_mario",
  "order_number": "ORD_20241216_001",
  "duration_ms": 45,
  "details": {
    "customer_name": "John Doe",
    "order_type": "delivery",
    "priority": 10,
    "total_amount": 15.99
  }
}
```

**Error Object Shape:**
For `ERROR` level logs, an `error` object must be included with the following structure:
```json
{
  "timestamp": "2024-12-16T10:35:00.000Z",
  "level": "ERROR",
  "service": "kitchen-worker",
  "version": "abcdef12345",
  "hostname": "kitchen-worker-xyz789",
  "request_id": "a1b2c3d4e5f6",
  "action": "db_query_failed",
  "message": "Failed to retrieve order from database",
  "error": {
    "type": "sql.ErrNoRows",
    "msg": "query failed: no rows in result set",
    "stack": "internal/db/order.go:120"
  }
}
```

### Log Levels Usage

*   **ERROR:** Database connection failures, RabbitMQ connection drops, order validation failures, message publish/consume errors, system crashes.
*   **WARN:** Message retry attempts, database query timeout warnings, configuration issues, order processing delays, worker disconnection/reconnection events.
*   **INFO:** Order lifecycle events (received, cooking started, completed), worker status changes, service startup/shutdown, normal business operations, performance milestones.
*   **DEBUG:** Detailed message content, database query execution details, internal processing steps, performance metrics, development and troubleshooting information.

### Log Location and Format Rules:
*   All logs must be emitted as single-line JSON (no pretty printing) to `stdout`. This is crucial for containerized environments where log collectors typically consume `stdout`.
*   Logs must not contain Personally Identifiable Information (PII), such as `customer_address`, payment information, or other sensitive data.
*   All log messages must be UTF-8 encoded, and newlines within log fields must be properly escaped.

### Required Log Events:

- **Order received/processed/completed** (INFO level)
- **Worker connected/disconnected** (INFO level)
- **Status changes** (INFO level)
- **Error conditions and recoveries** (ERROR/WARN level)
- **Performance metrics** (INFO/DEBUG level)

## Configuration Management

A clear configuration strategy is essential for deploying and running the services in different environments.

### Configuration Files

The system should be able use configuration files for database and RabbitMQ connections.

```yaml
# Database Configuration
database:
  host: localhost
  port: 5432
  user: restaurant_user
  password: restaurant_pass
  database: restaurant_db

# RabbitMQ Configuration  
rabbitmq:
  host: localhost
  port: 5672
  user: guest
  password: guest
  vhost: /
  exchanges:
    orders_topic:
      type: topic
      durable: true
    notifications_fanout:
      type: fanout
      durable: true
  queues:
    kitchen_queue:
      durable: true
      max_priority: 10
    kitchen_dine_in_queue:
      durable: true
      max_priority: 10
    kitchen_takeout_queue:
      durable: true
      max_priority: 10
    kitchen_delivery_queue:
      durable: true
      max_priority: 10
    notifications_queue:
      durable: true
      auto_delete: false
```

## Resources

- [RabbitMQ Documentation](https://www.rabbitmq.com/documentation.html)
- [RabbitMQ Docker Image](https://hub.docker.com/_/rabbitmq)
- [Go AMQP Client](https://github.com/rabbitmq/amqp091-go)
- [PostgreSQL Go Driver (pgx)](https://github.com/jackc/pgx)

## General Criteria

- Your code MUST be written in accordance with [gofumpt](https://github.com/mvdan/gofumpt). If not, you will automatically receive a score of `0`.
- Your program MUST compile successfully.
- Your program MUST NOT crash unexpectedly (any panics: `nil-pointer dereference`, `index out of range`, etc.). If this happens, you will receive `0` points during defense.
- Only built-in Go packages, `pgx/v5` and the official AMQP client (`github.com/rabbitmq/amqp091-go`) are allowed. If other packages are used, you will receive a score of `0`.
- RabbitMQ server MUST be running and available for connection.
- PostgreSQL database MUST be running and accessible for all services
- All RabbitMQ connections must handle reconnection scenarios
- Implement proper graceful shutdown for all services
- All database operations must be transactional where appropriate
- The project MUST compile with the following command in the project root directory:

```sh
$ go build -o restaurant-system .
```

## Mandatory Part

### Baseline

By default, your program should implement a basic order processing system using RabbitMQ work queue pattern to distribute orders among cooks.

## Support

If you get stuck, test your code with the example inputs from the project. You should get the same results. If not, re-read the description again. Perhaps you missed something, or your code is incorrect.

Make sure both PostgreSQL and RabbitMQ servers are running and accessible. Check the connection strings and verify proper configuration of database schema and message queues.

Test each component individually before integrating them together. Use the provided SQL scripts to verify database schema and the RabbitMQ management interface to monitor queue status.

If you're still stuck, review the logging output for error details, check service dependencies, and ensure all required environment variables are set correctly.

## Guidelines from Author

Before diving into code, it's crucial to step back and think about your system architecture. This project illustrates a fundamental principle of good software design - your architectural choices often determine the clarity and efficiency of your code.

Start with questions: How will components interact? Which message exchange patterns best fit the task? What delivery guarantees are needed? How will you handle failures and ensure data consistency? Only after you've carefully thought through these questions should you proceed to API design and code writing.

This approach may seem like extra work initially, but it pays off. Well-chosen architecture can make your code simpler, more readable, and often more efficient. It's like choosing the right tools before starting work - with the right foundation, the rest of the work becomes much easier.

Pay special attention to the data flow between services. Design your database schema first, then define your message formats, and finally implement the service logic. This order ensures consistency and reduces the need for major refactoring later.

Remember that in programming, as in many other things, thoughtful preparation is the key to success. Spend time on proper architecture, and you'll find that the coding process becomes smoother and more enjoyable.

Good system architecture is the foundation of clear, efficient code. It often simplifies programming more than clever algorithms can. Invest time in architectural design first. This approach usually leads to more maintainable and understandable programs, regardless of their size or complexity.

## Author

This project has been created by:

Yelnar Moldabekov

Contacts:

- Email: [mranle91@gmail.com](mailto:mranle91@gmail.com)
- [GitHub](https://github.com/ymoldabe/)