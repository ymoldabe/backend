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

This is the power of microservices and message queues. The challenge is to create a reliable order processing system that can handle a high volume of orders without slowing down. A single, monolithic application would quickly become a bottleneck. Instead, we distribute the work. The `Order Service` takes your order, the `Kitchen Service` cooks it, and a `Notification Subscriber` keeps you updated. They don't talk to each other directly; they pass messages through a central mailroom, RabbitMQ. This ensures that even if the kitchen is busy, the order service can still take new orders.

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
                        | Subscriber      |    | Service         |
                        +-----------------+    +-----------------+
```

## Database Schema

### Orders Table
**Purpose**: Primary storage for all restaurant orders with complete order information
**Used by**: Order Service (insert), Kitchen Workers (status updates), Tracking Service (queries)

```sql
create table orders (
    "id"                serial        primary key,
    "created_at"        timestamptz   not null    default now(),
    "updated_at"        timestamptz   not null    default now(),
    "number"            text          unique not null,
    "customer_name"     text          not null,
    "customer_type"     text          default 'regular',
    "type"              text          not null check (type in ('dine_in', 'takeout', 'delivery')),
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
create table order_items (
    "id"          serial        primary key,
    "created_at"  timestamptz   not null    default now(),
    "order_id"    integer       references orders(id),
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
    "id"          serial        primary key,
    "created_at"  timestamptz   not null    default now(),
    "order_id"    integer       references orders(id),
    "status"      text,
    "changed_by"  text,
    "changed_at"  timestamptz   default current_timestamp,
    "notes"       text
);
```

### Workers Table
**Purpose**: Registry and monitoring of all kitchen workers and their current status
**Used by**: Kitchen Workers (registration and heartbeat), Tracking Service (worker monitoring)

```sql
create table workers (
    "id"                serial      primary key,
    "created_at"        timestamptz not null    default now(),
    "name"              text        unique not null,
    "type"              text        not null,
    "status"            text        default 'online',
    "last_seen"         timestamptz default current_timestamp,
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
│       ├── kitchen.dine_in.10
│       ├── kitchen.takeout.5
│       └── kitchen.delivery.1
│
└── notifications_fanout (type: fanout, durable: true)
    └── Broadcasts to all subscribers

Queues:
├── kitchen_queue (durable: true, x-max-priority: 10)
│   └── Bound to orders_topic with routing key: kitchen.{order_type}.{priority}
│   └── Read by: General kitchen workers
├── kitchen_dine_in_queue (durable: true, x-max-priority: 10)
│   └── Bound to orders_topic with routing key: kitchen.dine_in.*
│   └── Read by: dine_in specialized workers
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
**Routing**: Through orders_topic exchange using routing keys like "kitchen.delivery.10"

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
**Sent to**: `notifications_fanout` exchange
**Read by**: Notification Subscribers
**Routing**: Broadcast to all queues bound to `notifications_fanout` exchange

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
- `timestamp`: The time the log entry was created.
- `level`: The log level (e.g., INFO, ERROR).
- `service`: The name of the service emitting the log (e.g., `order-service`, `kitchen-worker`).
- `action`: A concise, machine-readable string describing the event (e.g., `order_received`, `db_error`).
- `message`: A human-readable description of the event.
- `hostname`: The hostname or unique identifier of the module emitting the log.
- `request_id`: A unique identifier for correlating requests/operations across multiple services.

**Format Example:**
```json
{
  "timestamp": "2024-12-16T10:30:15.123Z",
  "level": "INFO",
  "service": "order-service",
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

- **ERROR:** Database connection failures, RabbitMQ connection drops, order validation failures, message publish/consume errors, system crashes.
- **WARN:** Message retry attempts, database query timeout warnings, configuration issues, order processing delays, worker disconnection/reconnection events.
- **INFO:** Order lifecycle events (received, cooking started, completed), worker status changes, service startup/shutdown, normal business operations, performance milestones.
- **DEBUG:** Detailed message content, database query execution details, internal processing steps, performance metrics, development and troubleshooting information.

### Log Location and Format Rules:
- All logs must be emitted as single-line JSON (no pretty printing) to `stdout`. This is crucial for containerized environments where log collectors typically consume `stdout`.
- Logs must not contain Personally Identifiable Information (PII), such as `customer_address`, payment information, or other sensitive data.
- All log messages must be UTF-8 encoded, and newlines within log fields must be properly escaped.

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
      x_max_priority: 10
    kitchen_dine_in_queue:
      durable: true
      x_max_priority: 10
    kitchen_takeout_queue:
      durable: true
      x_max_priority: 10
    kitchen_delivery_queue:
      durable: true
      x_max_priority: 10
    notifications_queue:
      durable: true
      auto_delete: false
```

## Resources

- [RabbitMQ Documentation](https://www.rabbitmq.com/documentation.html)
- [RabbitMQ Docker Image](https://hub.docker.com/_/rabbitmq)
- [Go AMQP Client](https://github.com/rabbitmq/amqp091-go)
- [PostgreSQL Go Driver (pgx)](https://github.com/jackc/pgx/v5)

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

#### Database Setup:

Create all required tables as specified in the Database Schema section. Each service must connect to PostgreSQL and handle connection errors gracefully.

#### RabbitMQ Setup:

  - Queue name: `kitchen_queue`
  - Exchange: topic exchange named `orders_topic`
  - Routing key: `kitchen.{order_type}.{priority}` (e.g., `kitchen.delivery.10`). `order_type` can be `dine_in`, `takeout`, `delivery`. `priority` can be `10`, `5`, `1`.
  - Queue should be durable and survive server restarts
  - Messages should be persistent
  - Connection must handle reconnection automatically

### Order Service

#### API Endpoint

- **Endpoint:** `POST /orders`
- **Content-Type:** `application/json`

#### Incoming Request Format

```json
{
  "customer_name": "John Doe",
  "order_type": "takeout",
  "items": [
    {"name": "Margherita Pizza", "quantity": 1, "price": 15.99},
    {"name": "Caesar Salad", "quantity": 1, "price": 8.99}
  ]
}
```

#### Processing Logic

1.  **Receive HTTP POST request.**
2.  **Validate data** against rules.
3.  **Calculate `total_amount`**.
4.  **Assign `priority`** based on rules (see [Priority Assignment Rules](#priority-assignment-rules) below).
5.  **Generate `order_number`** (`ORD_YYYYMMDD_NNN`). This should be a daily sequence starting at 001, based on the UTC date. The sequence number should be managed either by a database counter or a transaction-safe query to determine the next number for the current UTC day.
6.  **Store order** in PostgreSQL `orders` and `order_items` tables within a transaction.
7.  **Log initial status** to `order_status_log`.
8.  **Publish `Order Message` to RabbitMQ**:
    - **Exchange:** `orders_topic`.
    - **Routing Key:** `kitchen.{order_type}.{priority}` (e.g., `kitchen.takeout.1`).
    - **Message Properties:** Set `delivery_mode: 2` (persistent). For priority queueing, map the priority level (`10`/`5`/`1`) to a numeric value and set it in the `priority` message property.
9.  **Return HTTP JSON response.**

#### Outgoing Response Format

```json
{
  "order_number": "ORD_20241216_001",
  "status": "received",
  "total_amount": 24.98
}
```

#### Priority Assignment Rules

The system assigns a priority level (`10`, `5`, `1`) to each order. If an order matches multiple criteria, the highest applicable priority is assigned. This priority level is used in the routing key.

| Priority | Criteria                                    |
| :------- | :------------------------------------------ |
| `10`     | Order total amount is greater than $100.    |
| `5`      | Order total amount is between $50 and $100. |
| `1`      | All other standard orders.                  |

#### Validation Rules & Edge Cases

- `customer_name`: required, 1-100 characters, no special characters except spaces, hyphens, apostrophes.
- `order_type`: required, must be one of: 'dine_in', 'takeout', 'delivery'.
- `items`: required array, minimum 1 item, maximum 20 items per order.
- `item.name`: required, 1-50 characters.
- `item.quantity`: required, integer, 1-10 per item.
- `item.price`: required, decimal, 0.01-999.99.
- `table_number`: required for `dine_in` orders, 1-100.
- `delivery_address`: required for `delivery` orders, minimum 10 characters.
- **Conflicting Fields**:
    - `dine_in` orders must NOT include `delivery_address`.
    - `delivery` orders must NOT include `table_number`.
- **Duplicate Items**: Reject order if `items` contains duplicate entries by `name` that would violate the `quantity` rule (e.g., two "Margherita Pizza" entries summing to more than 10 quantity).
- **Database Transactions**: All database operations for order creation must be transactional.

#### Log Requirements

Use structured logging format as defined in the [Logging Format](#logging-format) section.

#### Flags

- `--mode`: Service mode (required: `order-service`)
- `--port`: HTTP port for REST API (default: 3000)
- `--max-concurrent`: Maximum concurrent orders (default: 50).

#### Worked Example

```sh
# Start the Order Service
./restaurant-system --mode=order-service --port=3000 --max-concurrent=50
# Expected console output:
{"timestamp":"2024-12-16T10:30:00.000Z","level":"INFO","service":"order-service","hostname":"order-service-789abc","request_id":"startup-001","action":"service_started","message":"Order Service started on port 3000","details":{"port":3000,"max_concurrent":50}}
{"timestamp":"2024-12-16T10:30:01.000Z","level":"INFO","service":"order-service","hostname":"order-service-789abc","request_id":"startup-001","action":"db_connected","message":"Connected to PostgreSQL database","duration_ms":250}
{"timestamp":"2024-12-16T10:30:02.000Z","level":"INFO","service":"order-service","hostname":"order-service-789abc","request_id":"startup-001","action":"rabbitmq_connected","message":"Connected to RabbitMQ exchange 'orders_topic'","duration_ms":150}
```

```sh
# Place order
curl -X POST http://localhost:3000/orders \
  -H "Content-Type: application/json" \
  -d '{
        "customer_name": "John Doe",
        "order_type": "takeout",
        "items": [
          {"name": "Margherita Pizza", "quantity": 1, "price": 15.99},
          {"name": "Caesar Salad", "quantity": 1, "price": 8.99}
        ]
      }'

# Expected HTTP Response:
# {"order_number": "ORD_20241216_001", "status": "received", "total_amount": 24.98}
```

### Order Types

#### Order Type Processing

- **dine_in:**
    - Requires `table_number`.
    - Shorter cooking time (8 seconds).
- **takeout:**
    - Standard processing.
    - Medium cooking time (10 seconds).
- **delivery:**
    - Requires `delivery_address`.
    - Longer cooking time (12 seconds).

#### Worker Specialization

- Workers can be configured to only accept certain types of orders using the `--order-types` flag. If a worker receives an order it cannot process, it must negatively acknowledge the message (`basic.nack`) and requeue it so another worker can pick it up.

### Kitchen Worker

#### Incoming Message Format

- **Queue:** `kitchen_queue` (or specialized queues like `kitchen_delivery_queue`)
- **Format:** See [Order Message](#order-message) section

#### Processing Logic

1.  **Consume Message:** Consume an order message from a bound kitchen queue.
2.  **Process Order:**
    - Update the order's `status` to `cooking` in the `orders` table.
    - Log the status change in the `order_status_log` table.
    - Publish a `status_update` message to the `notifications_fanout` exchange.
3.  **Simulate Cooking:** Simulate the cooking process with a configurable duration.
4.  **Update Status to 'ready':**
    - Update the order's `status` to `ready` in the `orders` table.
    - Update the `completed_at` timestamp.
    - Increment the `orders_processed` counter for the worker in the `workers` table.
    - Log the status change in the `order_status_log` table.
    - Publish a `status_update` message to the `notifications_fanout` exchange.
5.  **Acknowledge Receipt:** Acknowledge the message to RabbitMQ (`basic.ack`).
6.  **Handle Processing Failure:** If any step in processing fails (e.g., database error, message publish error), reject the message with `basic.nack(requeue=true)` to return it to the queue for redelivery.

#### Log Requirements

Use structured logging format as defined in the [Logging Format](#logging-format) section.

#### Relevant Flags

- `--port`: HTTP port for REST API (default: 3001)
- `--worker-name`: Unique name for the worker (required).
- `--order-types`: Comma-separated list of order types the worker can process (e.g., `dine_in,takeout`).

#### Worked Example

```sh
# Start multiple workers with different specializations
$ ./restaurant-system --mode=kitchen-worker --worker-name="chef_mario" --order-types="dine_in" &
$ ./restaurant-system --mode=kitchen-worker --worker-name="chef_luigi" --order-types="delivery" &
$ ./restaurant-system --mode=kitchen-worker --worker-name="chef_anna" &

# Monitor worker status
$ curl http://localhost:3001/workers/status
[
  {"worker_name": "chef_mario", "status": "online", "orders_processed": 5, "last_seen": "2024-12-16T10:35:00Z"},
  {"worker_name": "chef_luigi", "status": "online", "orders_processed": 3, "last_seen": "2024-12-16T10:35:01Z"}
]
```

### Multiple Workers & Load Balancing

#### Worker Management

- **Registration:** When a `kitchen-worker` starts, it should register itself in the `workers` table in the database.
- **Status Tracking:** The worker's status (`online`, `offline`, `processing`) and `last_seen` should be updated regularly. 
    - `online` indicates the worker is running and ready to receive orders. 
    - `processing` should be set when the worker is actively handling an order.
    - `offline` should be set during graceful shutdown.

#### Heartbeat & Offline Detection

- Each `kitchen-worker` sends a heartbeat every `--heartbeat-interval` (default 30 seconds) by executing an `UPDATE` query on the `workers` table: `UPDATE workers SET last_seen = now(), status='online' WHERE name = <worker_name>`. This indicates the worker is alive and active.
- The `tracking-service` (or a dedicated monitoring component) should periodically check the `workers` table. A worker is considered `offline` if `now() - last_seen` is greater than `2 * --heartbeat-interval`.

#### Load Balancing

- **Round-Robin:** By default, RabbitMQ will distribute orders to available workers in a round-robin fashion.
- **Priority Queues:** If queues are configured with `x-max-priority`, RabbitMQ delivers higher-priority messages before lower-priority regardless of round-robin.
- **Worker Specialization:** Workers can be configured to only accept certain types of orders using the `--order-types` flag.

#### Redelivery Algorithm

- **Prefetch Count:** Each `kitchen-worker` should configure its RabbitMQ consumer with `basic.qos(prefetch_count=N)` (where N is the `--prefetch` flag, default 1). This limits the number of unacknowledged messages a worker can receive at a time.
- **Clean Disconnection:** If a worker disconnects cleanly (e.g., graceful shutdown), it should `basic.nack(requeue=true)` any outstanding unacknowledged deliveries. This ensures messages are returned to the queue for other workers.
- **Unclean Disconnection/Crash:** If a worker crashes or disconnects uncleanly, RabbitMQ will automatically re-queue any unacknowledged messages after a timeout, making them available to other workers.

#### Log Requirements

Use structured logging format as defined in the [Logging Format](#logging-format) section.

#### Config / Flags

- `--worker-name`: Unique worker name (required).
- `--order-types`: Comma-separated order types to process.
- `--heartbeat-interval`: Interval in seconds for sending worker heartbeats (default: 30).
- `--prefetch`: RabbitMQ prefetch count for the worker (default: 1).

#### Validation Rules & Edge Cases

- **Duplicate Worker Name**: If the `INSERT` into `workers` violates name uniqueness, log `ERROR` and exit with status code `1`.
- **Graceful Shutdown**: Upon receiving a shutdown signal (e.g., `SIGINT`, `SIGTERM`), a worker should immediately stop accepting new orders from the queue. It must then finish processing any in-flight order, set its status to `offline` in the `workers` table, attempt to `basic.nack(requeue=true)` any unacknowledged messages, and then gracefully exit. This ensures that no work is lost and the system state remains consistent.
- **Database Unreachable**: Handle scenarios where the database is temporarily unreachable during heartbeat updates or registration. Implement retry mechanisms (e.g., three attempts with exponential back-off starting at 1 second, capped at 30 seconds) for database write operations.

### Tracking Service

This service provides HTTP API endpoints for querying order status, history, and worker information. It reads from the database and does not produce or consume messages.

#### API Endpoints

- **`GET /orders/{order_number}/status`:**

    - **Purpose:** To get the current status of a specific order.
    - **Response Format:**
        ```json
        {
          "order_number": "ORD_20241216_001",
          "current_status": "cooking",
          "updated_at": "2024-12-16T10:32:00Z",
          "estimated_completion": "2024-12-16T10:42:00Z",
          "processed_by": "chef_mario"
        }
        ```

- **`GET /orders/{order_number}/history`:**

    - **Purpose:** To get the full status history of an order.
    - **Response Format:**
        ```json
        [
          {"status": "received", "timestamp": "2024-12-16T10:30:00Z", "changed_by": "order-service"},
          {"status": "cooking", "timestamp": "2024-12-16T10:32:00Z", "changed_by": "chef_mario"}
        ]
        ```

- **`GET /workers/status`:**

    - **Purpose:** To get the status of all registered kitchen workers.
    - **Response Format:**
        ```json
        [
          {"worker_name": "chef_mario", "status": "online", "orders_processed": 5, "last_seen": "2024-12-16T10:35:00Z"},
          {"worker_name": "chef_luigi", "status": "online", "orders_processed": 3, "last_seen": "2024-12-16T10:35:01Z"}
        ]
        ```

#### Flags

- `--mode`: Service mode (required: `tracking-service`)
- `--port`: HTTP port for REST API (default: 3002)

### Notification Service

This service is responsible for consuming status update messages and notifying relevant parties. It demonstrates the publish/subscribe pattern, where multiple subscribers can listen for events without the publisher (Kitchen Worker) knowing about them.

#### Incoming Message Format

- **Queue:** `notifications_queue` (bound to `notifications_fanout` exchange)
- **Format:** See [Status Update Message](#status-update-message) section

#### Processing Logic

1.  **Consume Message:** Consume a status update message from the `notifications_queue`.
2.  **Display Notification:** Print the received notification to standard output in a clear, human-readable format.
3.  **Acknowledge Receipt:** Acknowledge the message to RabbitMQ (`basic.ack`) to remove it from the queue.

#### Log Requirements

Use structured logging format as defined in the [Logging Format](#logging-format) section for events like service startup and message consumption.

#### Flags

- `--mode`: Service mode (required: `notification-subscriber`)

#### Worked Example

```sh
# Start a subscriber to listen for all notifications
./restaurant-system --mode=notification-subscriber

# Start another subscriber interested only in notifications for "John Doe"
./restaurant-system --mode=notification-subscriber

# Expected console output for the general subscriber when an order status changes:
# Notification for order ORD_20241216_001: Status changed from 'received' to 'cooking' by chef_mario.
{"timestamp":"2024-12-16T10:32:05.000Z","level":"INFO","service":"notification-subscriber","hostname":"notification-sub-1","request_id":"a1b2c3d4e5f6","action":"notification_received","message":"Received status update for order ORD_20241216_001","details":{"order_number":"ORD_20241216_001","new_status":"cooking"}}
```

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