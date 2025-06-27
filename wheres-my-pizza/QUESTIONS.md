## Project Setup and Compilation
### Does the program compile successfully with `go build -o restaurant-system .`?
- [ ] Yes
- [ ] No

### Does the code follow gofumpt formatting standards?
- [ ] Yes
- [ ] No

### Does the program handle runtime errors gracefully without crashing?
- [ ] Yes
- [ ] No

### Is the program free of external packages except for pgx/v5 and official AMQP client?
- [ ] Yes
- [ ] No

## Architecture
### Is the application structured according to microservices architecture principles?
- [ ] Yes
- [ ] No

### Does the application have clearly separated service responsibilities (Order Service, Kitchen Workers, Tracking Service, Notification Subscriber)?
- [ ] Yes
- [ ] No

### Does the application implement proper message queue patterns (Work Queue, Publish/Subscribe, Routing)?
- [ ] Yes
- [ ] No

### Are components properly decoupled through RabbitMQ message broker?
- [ ] Yes
- [ ] No

## Database Schema
### Does the program correctly create all required database tables (orders, order_items, order_status_log, workers)?
- [ ] Yes
- [ ] No

### Does the orders table contain all required fields with proper constraints?
- [ ] Yes
- [ ] No

### Does the order_items table properly reference orders with a foreign key?
- [ ] Yes
- [ ] No

### Does the order_status_log table track all status changes with timestamps?
- [ ] Yes
- [ ] No

### Does the workers table manage worker registration and monitoring?
- [ ] Yes
- [ ] No

## RabbitMQ Configuration
### Does the program correctly set up required exchanges (orders_topic, notifications_fanout)?
- [ ] Yes
- [ ] No

### Does the program create all required queues with proper durability settings?
- [ ] Yes
- [ ] No

### Does the program configure priority queues with the x-max-priority parameter?
- [ ] Yes
- [ ] No

### Does the program handle RabbitMQ connection failures and reconnection?
- [ ] Yes
- [ ] No

## Order Service
### Does the Order Service accept HTTP POST requests for new orders?
- [ ] Yes
- [ ] No

### Does the program validate order data according to specified rules?
- [ ] Yes
- [ ] No

### Does the program calculate total amounts and assign priorities correctly?
- [ ] Yes
- [ ] No

### Does the program generate proper order numbers in ORD_YYYYMMDD_NNN format using UTC time?
- [ ] Yes
- [ ] No

### Does the program store orders in PostgreSQL within transactions?
- [ ] Yes
- [ ] No

### Does the program publish order messages to the RabbitMQ kitchen queue?
- [ ] Yes
- [ ] No

## Kitchen Worker
### Do Kitchen Workers consume orders from the correct queues?
- [ ] Yes
- [ ] No

### Do Kitchen Workers update order status to 'cooking' when processing starts?
- [ ] Yes
- [ ] No

### Do Kitchen Workers simulate the cooking process with a configurable duration?
- [ ] Yes
- [ ] No

### Do Kitchen Workers update order status to 'ready' upon completion?
- [ ] Yes
- [ ] No

### Do Kitchen Workers acknowledge messages only after successful database updates?
- [ ] Yes
- [ ] No

### Do specialized workers requeue messages they cannot process?
- [ ] Yes
- [ ] No

## Multiple Workers & Load Balancing
### Does the program support multiple kitchen workers simultaneously?
- [ ] Yes
- [ ] No

### Does the program distribute orders among available workers?
- [ ] Yes
- [ ] No

### Does the program register workers in the PostgreSQL workers table?
- [ ] Yes
- [ ] No

### Does the program track worker status (online, offline, processing) and performance metrics?
- [ ] Yes
- [ ] No

### Does the program handle worker disconnections gracefully (graceful shutdown)?
- [ ] Yes
- [ ] No

## Tracking Service
### Does the Tracking Service provide a REST API endpoint to get the current status of a specific order?
- [ ] Yes
- [ ] No

### Does the Tracking Service provide a REST API endpoint to get the full status history of an order?
- [ ] Yes
- [ ] No

### Does the Tracking Service provide a REST API endpoint to get the status of all registered kitchen workers?
- [ ] Yes
- [ ] No

## Notification Service
### Does the Notification Service use a fanout exchange for broadcasting status updates?
- [ ] Yes
- [ ] No

### Does the Notification Service consume messages from the notifications_queue?
- [ ] Yes
- [ ] No

### Does the Notification Service display notifications in a human-readable format?
- [ ] Yes
- [ ] No

## Logging and Configuration
### Does the program properly read configuration from files?
- [ ] Yes
- [ ] No

### Does the program use structured JSON logging throughout the application?
- [ ] Yes
- [ ] No

### Do logs include contextual information like timestamps, service names, and order numbers?
- [ ] Yes
- [ ] No

## Project Defense
### Can the team explain their microservices architecture decisions?
- [ ] Yes
- [ ] No

### Can the team explain how they implemented message queue patterns?
- [ ] Yes
- [ ] No

### Can the team demonstrate understanding of RabbitMQ features used?
- [ ] Yes
- [ ] No

### Can the team explain their database transaction handling approach?
- [ ] Yes
- [ ] No

### Can the team explain their priority queue implementation?
- [ ] Yes
- [ ] No

### Can the team demonstrate the system working with multiple workers?
- [ ] Yes
- [ ] No

## Detailed Feedback

### What was great? What you liked the most about the program and the team performance?

### What could be better? How those improvements could positively impact the outcome?