## Project Setup and Compilation
### Does the program compile successfully with `go build -o restaurant-system .`?
- [ ] Yes
- [ ] No

### Does the code follow gofumpt formatting standards?
- [ ] Yes
- [ ] No

### Does the program exit with a non-zero status code and clear error message when invalid arguments are provided?
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

### Does the system follow proper separation of concerns between services?
- [ ] Yes
- [ ] No

## Database Setup and Schema
### Does the program correctly create all required database tables (orders, order_items, order_status_log, workers)?
- [ ] Yes
- [ ] No

### Does the orders table contain all required fields with proper constraints?
- [ ] Yes
- [ ] No

### Does the order_items table properly reference orders with foreign key?
- [ ] Yes
- [ ] No

### Does the order_status_log table track all status changes with timestamps?
- [ ] Yes
- [ ] No

### Does the workers table manage worker registration and monitoring?
- [ ] Yes
- [ ] No

## RabbitMQ Configuration
### Does the program correctly set up required exchanges (orders_direct, notifications_fanout)?
- [ ] Yes
- [ ] No

### Does the program create all required queues with proper durability settings?
- [ ] Yes
- [ ] No

### Does the program configure priority queues with x-max-priority parameter?
- [ ] Yes
- [ ] No

### Does the program implement proper routing keys for different order types?
- [ ] Yes
- [ ] No

### Does the program handle RabbitMQ connection failures and reconnection?
- [ ] Yes
- [ ] No

## Order Processing
### Does the Order Service accept HTTP POST requests for new orders?
- [ ] Yes
- [ ] No

### Does the program validate order data according to specified rules?
- [ ] Yes
- [ ] No

### Does the program calculate total amounts and assign priorities correctly?
- [ ] Yes
- [ ] No

### Does the program generate proper order numbers in ORD_YYYYMMDD_NNN format?
- [ ] Yes
- [ ] No

### Does the program store orders in PostgreSQL within transactions?
- [ ] Yes
- [ ] No

### Does the program publish order messages to RabbitMQ kitchen queue?
- [ ] Yes
- [ ] No

## Kitchen Worker Implementation
### Do Kitchen Workers consume orders from the correct queues?
- [ ] Yes
- [ ] No

### Do Kitchen Workers update order status to 'cooking' when processing starts?
- [ ] Yes
- [ ] No

### Do Kitchen Workers simulate cooking process with configurable duration?
- [ ] Yes
- [ ] No

### Do Kitchen Workers update order status to 'ready' upon completion?
- [ ] Yes
- [ ] No

### Do Kitchen Workers acknowledge messages only after successful database updates?
- [ ] Yes
- [ ] No

## Multiple Workers Support
### Does the program support multiple kitchen workers simultaneously?
- [ ] Yes
- [ ] No

### Does the program distribute orders evenly among available workers using round-robin?
- [ ] Yes
- [ ] No

### Does the program register workers in the PostgreSQL workers table?
- [ ] Yes
- [ ] No

### Does the program track worker status and performance metrics?
- [ ] Yes
- [ ] No

### Does the program handle worker disconnections gracefully?
- [ ] Yes
- [ ] No

## Order Status Tracking
### Does the program track all status transitions in order_status_log table?
- [ ] Yes
- [ ] No

### Does the program validate status transitions to prevent invalid changes?
- [ ] Yes
- [ ] No

### Does the program provide REST API endpoints for status queries?
- [ ] Yes
- [ ] No

### Does the program provide order history through API endpoints?
- [ ] Yes
- [ ] No

### Does the program implement atomic status updates (database + message)?
- [ ] Yes
- [ ] No

## Notification System
### Does the program use fanout exchange for broadcasting status updates?
- [ ] Yes
- [ ] No

### Does the program send real-time notifications about status changes?
- [ ] Yes
- [ ] No

### Does the program support customer-specific notification filtering?
- [ ] Yes
- [ ] No

### Does the program log all notification deliveries?
- [ ] Yes
- [ ] No

### Does the program handle notification delivery failures gracefully?
- [ ] Yes
- [ ] No

## Order Types and Routing
### Does the program support three distinct order types (dine-in, takeout, delivery)?
- [ ] Yes
- [ ] No

### Does the program enforce different validation rules for each order type?
- [ ] Yes
- [ ] No

### Does the program implement different cooking times for different order types?
- [ ] Yes
- [ ] No

### Does the program route orders to specialized worker queues based on type?
- [ ] Yes
- [ ] No

### Does the program validate table numbers for dine-in and addresses for delivery orders?
- [ ] Yes
- [ ] No

## Priority Queue Implementation
### Does the program automatically assign priorities based on order characteristics?
- [ ] Yes
- [ ] No

### Does the program give VIP customers priority 10 (high priority)?
- [ ] Yes
- [ ] No

### Does the program assign priority 5 for large orders ($50-$100)?
- [ ] Yes
- [ ] No

### Do workers process orders in priority order (high priority first)?
- [ ] Yes
- [ ] No

### Does the program log priority assignments in order_status_log?
- [ ] Yes
- [ ] No

## Configuration and Logging
### Does the program properly read configuration from files and environment variables?
- [ ] Yes
- [ ] No

### Does the configuration include PostgreSQL connection details?
- [ ] Yes
- [ ] No

### Does the configuration include RabbitMQ connection details?
- [ ] Yes
- [ ] No

### Does the program use structured JSON logging throughout the application?
- [ ] Yes
- [ ] No

### Do logs include contextual information like timestamps, service names, and order numbers?
- [ ] Yes
- [ ] No

### Does the program log all required events (order lifecycle, worker status, errors)?
- [ ] Yes
- [ ] No

## System Operation
### Does the program implement graceful shutdown handling for all services?
- [ ] Yes
- [ ] No

### Does the program display comprehensive usage information with `--help` flag?
- [ ] Yes
- [ ] No

### Does the program handle database setup with `--setup-db` command?
- [ ] Yes
- [ ] No

### Does the program handle RabbitMQ setup with `--setup-queues` command?
- [ ] Yes
- [ ] No

### Does the program support all required operational modes?
- [ ] Yes
- [ ] No

## API Implementation
### Does the program implement order creation endpoint (POST /orders)?
- [ ] Yes
- [ ] No

### Does the program implement order status query endpoint (GET /orders/{id}/status)?
- [ ] Yes
- [ ] No

### Does the program implement order history endpoint (GET /orders/{id}/history)?
- [ ] Yes
- [ ] No

### Does the program implement worker status monitoring endpoint (GET /workers/status)?
- [ ] Yes
- [ ] No

### Do all API endpoints return proper HTTP status codes and error messages?
- [ ] Yes
- [ ] No

## Message Handling
### Does the program properly format order messages according to specification?
- [ ] Yes
- [ ] No

### Does the program properly format status update messages according to specification?
- [ ] Yes
- [ ] No

### Does the program ensure message persistence and durability?
- [ ] Yes
- [ ] No

### Does the program handle message acknowledgments correctly?
- [ ] Yes
- [ ] No

### Does the program implement proper error handling for failed message deliveries?
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