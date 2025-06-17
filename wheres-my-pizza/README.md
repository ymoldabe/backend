# wheres-my-pizza

## Learning Objectives

- Message Queue Systems
- RabbitMQ Integration
- Concurrent Programming

## Abstract

In this project, you will build a restaurant order management system using RabbitMQ as a message broker. The system simulates a real restaurant workflow where orders go through various processing stages: received -> cooking -> ready -> delivered.

Similar systems are used in real restaurants and food delivery services. For example, when you order food through an app, your order goes through an analogous processing system with task distribution among different staff members.

This project will teach you that before you start writing code, you should think through the system architecture, understand how components will interact, and only then proceed to implementation.

## Context

> you can’t polish your way out of bad architecture
>

The challenge we've chosen may seem unique, but at its core lies the common structure of many other distributed systems: data comes in, gets processed by various components, and is passed along the chain.

Our specific task is to create a reliable order processing system that can scale horizontally. If we simply processed orders sequentially in a single thread, the system would quickly become a bottleneck as load increases.

To achieve more efficient results, we need a distributed architecture with separation of concerns, where each component performs its specific function.

**Message Queue Patterns**

A smart way to solve this type of problem is using message queue patterns. This approach views the system as a set of interacting services, where each service processes a specific type of message and passes the result further down the chain.

**Work Queue Pattern**
- One producer sends tasks to a queue
- Multiple consumers compete to receive tasks
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

## Resources

- [RabbitMQ Documentation](https://www.rabbitmq.com/documentation.html)
- [RabbitMQ Docker Image](https://hub.docker.com/_/rabbitmq)
- [Go AMQP Client](https://github.com/rabbitmq/amqp091-go)

## General Criteria

- Your code MUST be written in accordance with [gofumpt](https://github.com/mvdan/gofumpt). If not, you will automatically receive a score of `0`.
- Your program MUST compile successfully.
- Your program MUST NOT crash unexpectedly (any panics: `nil-pointer dereference`, `index out of range`, etc.). If this happens, you will receive `0` points during defense.
- Only built-in Go packages, `pgx/v5` and the official AMQP client (`github.com/rabbitmq/amqp091-go`) are allowed. If other packages are used, you will receive a score of `0`.
- RabbitMQ server MUST be running and available for connection.
- The project MUST compile with the following command in the project root directory:

```sh
$ go build -o restaurant-system .
```

## Mandatory Part

### Baseline

By default, your program should implement a basic order processing system using RabbitMQ work queue pattern to distribute orders among cooks.

Outcomes:
- Program accepts orders through REST API
- Program uses RabbitMQ to store orders in queue
- Program distributes orders among available cooks
- Program handles basic error scenarios

Notes:
- Orders contain: order ID, list of dishes, customer information, order type
- Cooks compete for orders (competing consumers)
- Default cooking time 10 seconds for demonstration

Constraints:
- On any error, output error message with reason specified
- Orders must be acknowledged only after successful processing
- Maximum 50 concurrent orders in system
- Each order must be processed exactly once

Examples:

```sh
$ docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3-management

# Start order service
$ ./restaurant-system --mode=order-service --port=3000
Order service started on port 3000
Connected to RabbitMQ at localhost:5672
Kitchen queue initialized

# Place order
$ curl -X POST http://localhost:3000/orders \
  -H "Content-Type: application/json" \
  -d '{
        "customer_name": "John Doe",
        "phone": "555-0123",
        "items": [
          {"name": "Margherita Pizza", "quantity": 1},
          {"name": "Caesar Salad", "quantity": 1}
        ]
      }'

Response: {"order_id": "ORD_001", "status": "received", "message": "Order placed successfully"}

# Cook processes orders
$ ./restaurant-system --mode=kitchen-worker --worker-id="chef_mario"
Kitchen worker chef_mario connected to RabbitMQ
Listening for orders on kitchen_queue...
Received order ORD_001: Margherita Pizza (1), Caesar Salad (1)
Customer: John Doe (555-0123)
Processing order... (10 seconds)
Order ORD_001 completed by chef_mario
```

### Multiple Workers

Your program should support multiple cooks working simultaneously with automatic order distribution among them.

Outcomes:
- Program supports multiple cooks
- Orders are distributed evenly among available cooks
- New cooks can connect dynamically
- Cook disconnection doesn't affect processing of other orders

Constraints:
- Use RabbitMQ round-robin distribution
- Maximum 10 cooks simultaneously
- Each cook must have unique ID
- Unacknowledged messages should be redirected to other cooks

```sh
# Start multiple cooks
$ ./restaurant-system --mode=kitchen-worker --worker-id="chef_mario" &
$ ./restaurant-system --mode=kitchen-worker --worker-id="chef_luigi" &
$ ./restaurant-system --mode=kitchen-worker --worker-id="chef_anna" &

Kitchen worker chef_mario connected
Kitchen worker chef_luigi connected  
Kitchen worker chef_anna connected
All workers listening on kitchen_queue

# Monitor order distribution
$ curl -X POST http://localhost:3000/orders -d '{"customer_name":"Alice","items":[{"name":"Pizza","quantity":1}]}'
$ curl -X POST http://localhost:3000/orders -d '{"customer_name":"Bob","items":[{"name":"Pasta","quantity":1}]}'
$ curl -X POST http://localhost:3000/orders -d '{"customer_name":"Carol","items":[{"name":"Salad","quantity":1}]}'

# In cook logs:
# chef_mario: Received order ORD_001 (Alice - Pizza)
# chef_luigi: Received order ORD_002 (Bob - Pasta)  
# chef_anna: Received order ORD_003 (Carol - Salad)
```

### Order Status Tracking

Your program should implement an order status tracking system using publish/subscribe pattern for notifications.

Outcomes:
- Program tracks status of each order
- Order status is updated at each processing stage
- Clients can query status of their orders
- System sends notifications about status changes

Constraints:
- Statuses: received, cooking, ready, delivered
- Use fanout exchange for notifications
- Notifications must contain order_id, status, timestamp
- API endpoint for checking order status

```sh
# Start tracking service
$ ./restaurant-system --mode=tracking-service &

# Subscribe to order notifications
$ ./restaurant-system --mode=notification-subscriber --customer="John Doe"
Subscribed to order notifications
Received: Order ORD_001 status changed to 'cooking' at 2024-12-16 10:30:15
Received: Order ORD_001 status changed to 'ready' at 2024-12-16 10:40:20

# Check order status via API
$ curl http://localhost:3000/orders/ORD_001/status
{"order_id": "ORD_001", "status": "ready", "updated_at": "2024-12-16T10:40:20Z"}
```

### Order Types and Routing

Your program should support different order types (dine-in, takeout, delivery) with routing based on order type.

Outcomes:
- Program supports three order types: dine-in, takeout, delivery
- Orders are routed to appropriate queues based on type
- Different order types have different processing times
- Delivery orders require additional processing

Constraints:
- Use topic exchange with routing keys: `orders.kitchen.{type}`
- Cooking times: dine-in (8 sec), takeout (10 sec), delivery (12 sec)
- Delivery orders must include address
- Order type validation on creation

```sh
# Dine-in order
$ curl -X POST http://localhost:3000/orders \
  -H "Content-Type: application/json" \
  -d '{
        "customer_name": "John Doe",
        "order_type": "dine-in",
        "table_number": 5,
        "items": [{"name": "Steak", "quantity": 1}]
      }'

# Takeout order
$ curl -X POST http://localhost:3000/orders \
  -H "Content-Type: application/json" \
  -d '{
        "customer_name": "Jane Smith", 
        "order_type": "takeout",
        "items": [{"name": "Burger", "quantity": 2}]
      }'

# Delivery order
$ curl -X POST http://localhost:3000/orders \
  -H "Content-Type: application/json" \
  -d '{
        "customer_name": "Bob Wilson",
        "order_type": "delivery", 
        "address": "123 Main St, City",
        "items": [{"name": "Pizza", "quantity": 1}]
      }'

# Cook can listen to specific order types
$ ./restaurant-system --mode=kitchen-worker --worker-id="chef_mario" --order-types="dine-in,takeout"
chef_mario listening for dine-in and takeout orders only
```

### Priority Queue

Your program should support order prioritization using RabbitMQ priority queues.

Outcomes:
- Program supports priority orders
- VIP clients receive high priority automatically
- Large orders (>$50) receive medium priority
- Urgent orders can be marked as priority

Constraints:
- Priorities: 1 (normal), 5 (medium), 10 (high)
- Configure queues with x-max-priority parameter
- VIP status determined by client ID or flag
- Priority calculated when placing order

```sh
# VIP order (high priority)
$ curl -X POST http://localhost:3000/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customer_name": "VIP Customer",
    "customer_type": "VIP",
    "items": [{"name": "Lobster", "quantity": 1, "price": 45.99}],
    "rush": true
  }'

Response: {"order_id": "ORD_002", "priority": 10, "status": "received"}

# Large order (medium priority)  
$ curl -X POST http://localhost:3000/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customer_name": "Regular Customer",
    "items": [
      {"name": "Pizza", "quantity": 3, "price": 15.99},
      {"name": "Salad", "quantity": 2, "price": 8.99}
    ]
  }'

Response: {"order_id": "ORD_003", "priority": 5, "status": "received"}

# Cook receives orders by priority
$ ./restaurant-system --mode=kitchen-worker --worker-id="chef_mario"
Received HIGH PRIORITY order ORD_002 (VIP Customer - Lobster)
Received MEDIUM PRIORITY order ORD_003 (Regular Customer - Pizza x3, Salad x2)
```

### Usage

Your program should output usage information.

Outcomes:
- Program outputs text describing usage of all modes and options

```sh
$ ./restaurant-system --help
Restaurant Order Management System with RabbitMQ

Usage:
  restaurant-system --mode=order-service [--port=N] [--rabbitmq=HOST:PORT]
  restaurant-system --mode=kitchen-worker --worker-id=ID [--order-types=TYPES] [--rabbitmq=HOST:PORT]
  restaurant-system --mode=tracking-service [--rabbitmq=HOST:PORT]
  restaurant-system --mode=notification-subscriber [--customer=NAME] [--rabbitmq=HOST:PORT]
  restaurant-system --setup-queues [--rabbitmq=HOST:PORT]
  restaurant-system --help

Service Modes:
  order-service             REST API for receiving orders
  kitchen-worker           Kitchen staff processing orders  
  tracking-service         Order status tracking service
  notification-subscriber  Customer notification service

Connection Options:
  --rabbitmq              RabbitMQ server address (default: localhost:5672)
  --rabbitmq-user         RabbitMQ username (default: guest)
  --rabbitmq-pass         RabbitMQ password (default: guest)

Order Service Options:
  --port                  HTTP port for REST API (default: 3000)

Kitchen Worker Options:
  --worker-id             Unique worker identifier (required)
  --order-types           Comma-separated order types to process (default: all)

Notification Options:
  --customer              Customer name for notifications

Setup:
  --setup-queues          Initialize RabbitMQ exchanges and queues

Examples:
  # Setup RabbitMQ
  ./restaurant-system --setup-queues

  # Start order service
  ./restaurant-system --mode=order-service --port=3000

  # Start kitchen worker
  ./restaurant-system --mode=kitchen-worker --worker-id=chef1

  # Monitor notifications
  ./restaurant-system --mode=notification-subscriber --customer="John Doe"
```

## Support

If you get stuck, test your code with the example inputs from the project. You should get the same results. If not, re-read the description again. Perhaps you missed something, or your code is incorrect.

Make sure the RabbitMQ server is running and accessible. Check the connection to RabbitMQ and proper configuration of queues and exchanges.

If you're still stuck, ask a friend for help or take a break and come back later.

## Guidelines from Author

Before diving into code, it's crucial to step back and think about your system architecture. This project illustrates a fundamental principle of good software design - your architectural choices often determine the clarity and efficiency of your code.

Start with questions: How will components interact? Which message exchange patterns best fit the task? What delivery guarantees are needed? Only after you've carefully thought through these questions should you proceed to API design and code writing.

This approach may seem like extra work initially, but it pays off. Well-chosen architecture can make your code simpler, more readable, and often more efficient. It's like choosing the right tools before starting work - with the right foundation, the rest of the work becomes much easier.

Remember that in programming, as in many other things, thoughtful preparation is the key to success. Spend time on proper architecture, and you'll find that the coding process becomes smoother and more enjoyable.

Good system architecture is the foundation of clear, efficient code. It often simplifies programming more than clever algorithms can. Invest time in architectural design first. This approach usually leads to more maintainable and understandable programs, regardless of their size or complexity.

## Author

This project has been created by:

Yelnar Moldabekov

Contacts:

- Email: [mranle91@gmail.com](mailto:mranle91@gmail.com)
- [GitHub](https://github.com/ymoldabe/)