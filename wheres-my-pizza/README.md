# wheres-my-pizza

## Learning Objectives

- Message Queue Systems
- RabbitMQ Integration
- Concurrent Programming
- Microservices Architecture

## Abstract

In this project, you will build a restaurant order management system using RabbitMQ as a message broker and PostgreSQL for persistent storage. The system simulates a real restaurant workflow where orders go through various processing stages: received -> cooking -> ready -> delivered.

Similar systems are used in real restaurants and food delivery services. For example, when you order food through an app, your order goes through an analogous processing system with task distribution among different staff members.

This project will teach you that before you start writing code, you should think through the system architecture, understand how components will interact, and only then proceed to implementation.

## Context

> You can't polish your way out of bad architecture.
>

The challenge we've chosen may seem unique, but at its core lies the common structure of many other distributed systems: data comes in, gets processed by various components, and is passed along the chain.

Our specific task is to create a reliable order processing system that can scale horizontally. If we simply processed orders sequentially in a single thread, the system would quickly become a bottleneck as load increases.

To achieve more efficient results, we need a distributed architecture with separation of concerns, where each component performs its specific function.

**Message Queue Patterns**

A smart way to solve this type of problem is using message queue patterns. This approach views the system as a set of interacting services, where each service processes a specific type of message and passes the result further down the chain.

**Work Queue Pattern**
- One producer sends tasks to a queue
- Multiple consumers wait for the task to arrive, but only one receives it.
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

Order processing algorithm using queues:

```
1. Client places order through API
2. Order is placed in kitchen_queue
3. Available cook takes order from queue
4. After cooking, order is directed to quality_queue
5. Quality controller checks the order
6. Ready order is placed in packaging_queue
7. Packager prepares order for delivery
8. Order is directed to courier through delivery_queue
9. All interested parties receive status notifications
```

**Data Structure and Architecture**

Let's consider our program requirements. We aim to process hundreds of orders simultaneously, support multiple workers, and ensure response times in seconds, not minutes. At this scale, we cannot afford inefficient algorithms or architecture.

The order processing workflow requires coordination between multiple components. This means we need a smart way to organize interaction between services. We're looking for a structure that can efficiently represent order state and its transitions between stages.

Our program will work in microservice mode: each component (cook, quality controller, packager, courier) works independently but is coordinated through RabbitMQ. Each service must quickly receive appropriate tasks and send results further down the chain.

Let's call the combination of an order and its current state a "processing stage" - this is a common term in workflow systems. We'll need to track transitions between stages and ensure reliable message delivery.

This approach gives us a clear path forward. By focusing on these key requirements, we can design a solution that is both elegant and efficient.

## System Architecture

### Service Responsibilities

**Order Service**
- Accept HTTP requests for new orders
- Validate order data and calculate totals
- Store orders in PostgreSQL
- Publish orders to RabbitMQ kitchen queue
- Provide order status API endpoints

**Kitchen Workers**
- Consume orders from kitchen queue
- Simulate cooking process
- Update order status in database
- Publish status updates to notification exchange

**Tracking Service**
- Monitor order status changes
- Handle status update requests
- Provide centralized status tracking
- Log all status transitions

**Notification Subscriber**
- Listen for status update notifications
- Filter notifications by customer
- Display real-time status updates

## Database Schema

### Orders Table

```sql
create table orders (
    "id"                uuid          primary key default gen_random_uuid(),
    "created_at"        timestamptz   not null    default now(),
    "updated_at"        timestamptz   not null    default now(),
    "number"            varchar(20)   unique not null,
    "customer_name"     varchar(100)  not null,
    "customer_type"     varchar(10)   default 'regular',
    "type"        varchar(10)   not null check (order_type in ('dine-in', 'takeout', 'delivery')),
    "table_number"      integer,
    "delivery_address"  text,
    "total_amount"      decimal(10,2) not null,
    "priority"          integer       default 1,
    "status"            varchar(20)   default 'received',
    "processed_by"      varchar(50),
    "completed_at"      timestamptz
);
```

### Order Items Table

```sql
CREATE TABLE order_items (
    "id"          uuid          primary key default gen_random_uuid(),
    "created_at"  timestamptz   not null    default now(),
    "order_id"    varchar(20)   references orders(id),
    "name"        varchar(100)  not null,
    "quantity"    integer       not null,
    "price"       decimal(8,2)  not null
);
```

### Order Status Log Table

```sql
create table order_status_log (
    "id"          uuid          primary key default gen_random_uuid(),
    "created_at"  timestamptz   not null    default now(),
    order_id      varchar(20)   references orders(id),
    "status"      varchar(20),
    "changed_by"  varchar(50),
    "changed_at"  timestamp     default current_timestamp,
    "notes"       text
);
```

### Workers Table

```sql
create table workers (
    "id"                uuid        primary key default gen_random_uuid(),
    "created_at"        timestamptz not null    default now(),
    "name"              varchar(50) unique not null,
    "type"              varchar(20) not null,
    "status"            varchar(10) default 'online',
    "last_seen"         timestamp   default current_timestamp,
    "orders_processed"  integer     default 0
);
```

## RabbitMQ Configuration

### Exchanges and Queues Setup

```
Exchanges:
├── orders_direct (type: direct, durable: true)
│   └── Routing Keys:
│       ├── kitchen.dine-in
│       ├── kitchen.takeout
│       └── kitchen.delivery
│
└── notifications_fanout (type: fanout, durable: true)
    └── Broadcasts to all subscribers

Queues:
├── kitchen_queue (durable: true, x-max-priority: 10)
├── kitchen_dine_in_queue (durable: true, x-max-priority: 10)
├── kitchen_takeout_queue (durable: true, x-max-priority: 10)
├── kitchen_delivery_queue (durable: true, x-max-priority: 10)
└── notifications_queue (durable: true, auto-delete: false)
```

### Message Formats

**Order Message**
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

**Status Update Message**
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

All services must implement structured logging in JSON format:

```json
{
  "timestamp": "2024-12-16T10:30:15.123Z",
  "level": "INFO",
  "service": "order-service",
  "worker_name": "chef_mario",
  "order_number": "ORD_20241216_001",
  "action": "order_received",
  "message": "Order received and queued for processing",
  "duration_ms": 45,
  "details": {
    "customer_name": "John Doe",
    "order_type": "delivery",
    "priority": 10,
    "total_amount": 15.99
  }
}
```

**Log Levels:**
- ERROR: System errors, connection failures, invalid data
- WARN: Retry attempts, temporary failures, configuration issues  
- INFO: Order lifecycle events, worker status changes
- DEBUG: Detailed processing information, performance metrics

**Required Log Events:**
- Order received/processed/completed
- Worker connected/disconnected
- Status changes
- Error conditions and recoveries
- Performance metrics (processing times)

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

#### Database Setup:

Create all required tables as specified in the Database Schema section. Each service must connect to PostgreSQL and handle connection errors gracefully.

#### RabbitMQ Setup:

- Queue name: `kitchen_queue`
- Exchange: direct exchange named `orders_direct`
- Routing key: `kitchen.order`
- Queue should be durable and survive server restarts
- Messages should be persistent
- Connection must handle reconnection automatically

#### Order Processing Flow:

1. Order Service receives HTTP POST request
2. Validates request data and calculates totals
3. Stores order in PostgreSQL orders table
4. Publishes order message to RabbitMQ
5. Kitchen Worker consumes order from queue
6. Updates order status to 'cooking' in database
7. Simulates cooking process (configurable duration)
8. Updates order status to 'ready' in database
9. Logs completion and acknowledges message

Outcomes:
- Program accepts orders through REST API
- Program uses PostgreSQL to persist order data
- Program uses RabbitMQ to distribute orders among cooks
- Program handles database and RabbitMQ connection failures
- All status changes are logged to database

Notes:
- Orders contain: order number, customer info, items list, pricing, timestamps
- Cooks compete for orders (competing consumers pattern)
- Default cooking time: 10 seconds for demo purposes
- Order IDs follow format: `ORD_YYYYMMDD_NNN`

Constraints:
- On any error, output structured JSON log with error details
- Orders must be acknowledged only after successful database update
- Maximum 50 concurrent orders in system
- Each order must be processed exactly once
- Database transactions must be used for order updates

Examples:

```sh
$ docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3-management
$ docker run -d --name postgres -p 5432:5432 -e POSTGRES_PASSWORD=password postgres:13

# Initialize database and queues
$ ./restaurant-system --setup-db --db="postgres://user:password@localhost/restaurant"
$ ./restaurant-system --setup-queues --rabbitmq="localhost:5672"

# Start order service
$ ./restaurant-system --mode=order-service --port=3000 --db="postgres://user:password@localhost/restaurant"
{"timestamp":"2024-12-16T10:30:00Z","level":"INFO","service":"order-service","message":"Service started on port 3000"}

# Start kitchen worker
$ ./restaurant-system --mode=kitchen-worker --worker-name="chef_mario" --db="postgres://user:password@localhost/restaurant"
{"timestamp":"2024-12-16T10:30:05Z","level":"INFO","service":"kitchen-worker","worker_name":"chef_mario","message":"Worker connected and ready"}

# Place order
$ curl -X POST http://localhost:3000/orders \
  -H "Content-Type: application/json" \
  -d '{
        "customer_name": "John Doe",
        "order_type": "takeout",
        "items": [
          {"name": "Margherita Pizza", "quantity": 1, "price": 15.99},
          {"name": "Caesar Salad", "quantity": 1, "price": 8.99}
        ]
      }'

Response: {"order_number": "ORD_20241216_001", "status": "received", "total_amount": 24.98}
```

### Multiple Workers

Your program should support multiple cooks working simultaneously with automatic order distribution among them.

#### Worker Management:

- Register workers in PostgreSQL workers table
- Track worker status and last seen timestamp
- Handle worker disconnections gracefully
- Redistribute unacknowledged orders to available workers

#### Load Balancing:

- Use RabbitMQ round-robin distribution by default
- Track worker performance metrics in database
- Support worker-specific order type filtering
- Implement worker health monitoring

Outcomes:
- Program supports multiple cooks simultaneously
- Orders are distributed evenly among available cooks
- New cooks can connect dynamically without service restart
- Cook disconnection doesn't affect processing of other orders
- Worker status is tracked in database

Constraints:
- Use RabbitMQ round-robin distribution mechanism
- Maximum 10 active workers simultaneously
- Each cook must have unique worker name
- Unacknowledged messages are redelivered to other workers
- Worker heartbeat every 30 seconds

```sh
# Start multiple workers with different specializations
$ ./restaurant-system --mode=kitchen-worker --worker-name="chef_mario" --order-types="pizza,pasta" &
$ ./restaurant-system --mode=kitchen-worker --worker-name="chef_luigi" --order-types="salad,soup" &
$ ./restaurant-system --mode=kitchen-worker --worker-name="chef_anna" &

# Worker logs show specialization
{"timestamp":"2024-12-16T10:30:00Z","level":"INFO","service":"kitchen-worker","worker_name":"chef_mario","message":"Worker registered for order types: pizza,pasta"}

# Monitor worker status
$ curl http://localhost:3000/workers/status
[
  {"worker_name": "chef_mario", "status": "online", "orders_processed": 5, "last_seen": "2024-12-16T10:35:00Z"},
  {"worker_name": "chef_luigi", "status": "online", "orders_processed": 3, "last_seen": "2024-12-16T10:35:01Z"}
]
```

### Order Status Tracking

Your program should implement a comprehensive order status tracking system using publish/subscribe pattern for real-time notifications.

#### Status Management:

- Track all status transitions in `order_status_log` table
- Implement status validation (prevent invalid transitions)
- Provide real-time status updates via RabbitMQ
- Support status queries via REST API

#### Notification System:

- Use fanout exchange for broadcasting status updates
- Support customer-specific notification filtering
- Log all notification deliveries
- Handle notification delivery failures

Outcomes:
- Program tracks complete status history for each order
- Order status is updated at each processing stage
- Clients can query current and historical status
- System sends real-time notifications about status changes
- Status transitions are validated and logged

Constraints:
- Valid statuses: received, cooking, ready, out-for-delivery, delivered, cancelled
- Use fanout exchange named `notifications_fanout`
- All status changes must be atomic (database + message)
- Notifications include timestamp, order details, estimated completion
- API endpoints for status queries and history

```sh
# Start tracking service
$ ./restaurant-system --mode=tracking-service --db="postgres://user:password@localhost/restaurant" &
{"timestamp":"2024-12-16T10:30:00Z","level":"INFO","service":"tracking-service","message":"Tracking service started"}

# Start notification subscriber
$ ./restaurant-system --mode=notification-subscriber --customer="John Doe" &
{"timestamp":"2024-12-16T10:30:05Z","level":"INFO","service":"notification-subscriber","message":"Subscribed to notifications for customer: John Doe"}

# Check order status and history
$ curl http://localhost:3000/orders/ORD_20241216_001/status
{
  "order_number": "ORD_20241216_001",
  "current_status": "cooking",
  "updated_at": "2024-12-16T10:32:00Z",
  "estimated_completion": "2024-12-16T10:42:00Z",
  "processed_by": "chef_mario"
}

$ curl http://localhost:3000/orders/ORD_20241216_001/history
[
  {"status": "received", "timestamp": "2024-12-16T10:30:00Z", "changed_by": "order-service"},
  {"status": "cooking", "timestamp": "2024-12-16T10:32:00Z", "changed_by": "chef_mario"}
]
```

### Order Types and Routing

Your program should support different order types with specialized routing and processing requirements.

#### Order Type Processing:

- **Dine-in**: Requires table number, shorter cooking time (8 sec)
- **Takeout**: Standard processing, medium cooking time (10 sec)  
- **Delivery**: Requires address validation, longer cooking time (12 sec)

#### Routing Configuration:

- Use topic exchange with routing patterns
- Route orders to specialized worker queues
- Support worker subscription to specific order types
- Implement priority routing for rush orders

Outcomes:
- Program supports three distinct order types with different requirements
- Orders are routed to appropriate processing queues
- Different order types have different processing times and validation
- Delivery orders include address validation and special handling
- Workers can specialize in specific order types

Constraints:
- Use topic exchange named `orders_topic` with routing keys `kitchen.{type}.{priority}`
- Order type validation on creation with specific field requirements
- Cooking time varies by type: dine-in (8s), takeout (10s), delivery (12s)
- Delivery orders require valid address format
- Table numbers required for dine-in orders

```sh
# Place different order types
$ curl -X POST http://localhost:3000/orders \ 
  -d '{"customer_name": "John", "order_type": "dine-in", "table_number": 5, 
       "items": [{"name": "Steak", "quantity": 1, "price": 25.99}]}'

$ curl -X POST http://localhost:3000/orders \
  -d '{"customer_name": "Jane", "order_type": "delivery", 
       "delivery_address": "123 Main St, City, State 12345",
       "items": [{"name": "Pizza", "quantity": 1, "price": 18.99}]}'

# Worker specializing in delivery orders
$ ./restaurant-system --mode=kitchen-worker --worker-name="delivery_chef" --order-types="delivery"
{"timestamp":"2024-12-16T10:30:00Z","level":"INFO","service":"kitchen-worker","worker_name":"delivery_chef","message":"Listening for delivery orders only"}
```

### Priority Queue

Your program should support order prioritization using RabbitMQ priority queues with automatic priority assignment.

#### Priority Assignment Rules:

- **Priority 10 (High)**: VIP customers, rush orders, orders > $100
- **Priority 5 (Medium)**: Large orders ($50-$100), repeat customers  
- **Priority 1 (Normal)**: Standard orders

#### Priority Processing:

- Configure all queues with x-max-priority: 10
- Workers process high-priority orders first
- Track priority metrics in database
- Support manual priority override for special cases

Outcomes:
- Program automatically assigns priorities based on order characteristics
- VIP customers and large orders receive expedited processing
- Workers process orders in priority order
- Priority assignment is logged and auditable
- System supports manual priority overrides

Constraints:
- All kitchen queues configured with x-max-priority parameter set to 10
- Priority calculated automatically based on customer type and order value
- VIP status stored in customer database or determined by previous orders
- Priority changes logged in order_status_log table
- High-priority orders bypass normal queue position

```sh
# VIP customer order (automatically high priority)
$ curl -X POST http://localhost:3000/orders \
  -d '{
    "customer_name": "VIP Customer",
    "customer_type": "vip",
    "order_type": "delivery",
    "delivery_address": "456 Executive Blvd",
    "items": [{"name": "Premium Steak", "quantity": 1, "price": 45.99}],
    "rush": true
  }'

Response: {"order_number": "ORD_20241216_002", "priority": 10, "status": "received", "estimated_completion": "2024-12-16T10:45:00Z"}

# Large order (automatically medium priority)
$ curl -X POST http://localhost:3000/orders \
  -d '{
    "customer_name": "Office Catering",
    "items": [
      {"name": "Pizza", "quantity": 5, "price": 15.99},
      {"name": "Salad", "quantity": 3, "price": 8.99}
    ]
  }'

Response: {"order_number": "ORD_20241216_003", "priority": 5, "total_amount": 106.92}

# Worker processes by priority
{"timestamp":"2024-12-16T10:32:00Z","level":"INFO","service":"kitchen-worker","worker_name":"chef_mario","order_number":"ORD_20241216_002","message":"Processing HIGH PRIORITY order","priority":10}
```

### Usage

Your program should output comprehensive usage information for all operational modes.

Outcomes:
- Program outputs detailed usage information for all modes and options
- Includes examples for common usage scenarios
- Documents all configuration parameters
- Provides troubleshooting guidance

```sh
$ ./restaurant-system --help
Restaurant Order Management System with RabbitMQ and PostgreSQL

Usage:
  restaurant-system --mode=order-service [OPTIONS]
  restaurant-system --mode=kitchen-worker --worker-name=worker_name [OPTIONS]
  restaurant-system --mode=tracking-service [OPTIONS]
  restaurant-system --mode=notification-subscriber [OPTIONS]
  restaurant-system --setup-db [OPTIONS]
  restaurant-system --setup-queues [OPTIONS]
  restaurant-system --help

Service Modes:
  order-service             REST API for receiving and managing orders
  kitchen-worker           Kitchen staff processing orders from queue
  tracking-service         Centralized order status tracking service
  notification-subscriber  Real-time customer notification service

Connection Options:
  --db                    PostgreSQL connection string (required for all modes)
                         Format: postgres://user:pass@host:port/dbname
  --rabbitmq             RabbitMQ server address (default: localhost:5672)
  --rabbitmq-user        RabbitMQ username (default: guest)
  --rabbitmq-pass        RabbitMQ password (default: guest)

Order Service Options:
  --port                 HTTP port for REST API (default: 3000)
  --max-concurrent       Maximum concurrent orders (default: 50)

Kitchen Worker Options:
  --worker-name            Unique worker name (required)
  --order-types          Comma-separated order types to process (default: all)
                        Options: dine-in,takeout,delivery
  --cooking-time         Base cooking time in seconds (default: 10)

Notification Options:
  --customer            Customer name filter for notifications
  --notification-types  Types of notifications to receive (default: all)

Setup Commands:
  --setup-db            Initialize PostgreSQL database schema
  --setup-queues        Initialize RabbitMQ exchanges and queues
  --migrate-db          Run database migrations

Logging Options:
  --log-level           Log level: DEBUG, INFO, WARN, ERROR (default: INFO)
  --log-file            Log file path (default: stdout)

Examples:
  # Setup environment
  ./restaurant-system --setup-db --db="postgres://user:pass@localhost/restaurant"
  ./restaurant-system --setup-queues --rabbitmq="localhost:5672"

  # Start order service
  ./restaurant-system --mode=order-service --port=3000 --db="postgres://user:pass@localhost/restaurant"

  # Start specialized kitchen worker
  ./restaurant-system --mode=kitchen-worker --worker-name=pizza_chef --order-types=dine-in,takeout --db="postgres://user:pass@localhost/restaurant"

  # Monitor specific customer notifications
  ./restaurant-system --mode=notification-subscriber --customer="John Doe"

Environment Variables:
  DB_URL      PostgreSQL connection string
  RABBITMQ    RabbitMQ server address
  LOG_LEVEL   Default log level

Troubleshooting:
  - Ensure PostgreSQL and RabbitMQ services are running
  - Check connection strings and credentials
  - Verify database schema is initialized
  - Check RabbitMQ management interface at http://localhost:15672
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