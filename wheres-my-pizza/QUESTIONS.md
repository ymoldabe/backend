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

### Is the program free of external packages except for the official AMQP client (github.com/rabbitmq/amqp091-go)?
- [ ] Yes
- [ ] No

## Basic Functionality (Baseline)
### Does the program accept orders through REST API?
- [ ] Yes
- [ ] No

### Does the program correctly connect to RabbitMQ server?
- [ ] Yes
- [ ] No

### Does the program use RabbitMQ work queue pattern to distribute orders among cooks?
- [ ] Yes
- [ ] No

### Does the program handle order data correctly (order ID, dishes list, customer information, order type)?
- [ ] Yes
- [ ] No

### Does the program acknowledge orders only after successful processing?
- [ ] Yes
- [ ] No

### Does the program enforce maximum 50 concurrent orders constraint?
- [ ] Yes
- [ ] No

### Does each order get processed exactly once?
- [ ] Yes
- [ ] No

## Multiple Workers Implementation
### Does the program support multiple cooks working simultaneously?
- [ ] Yes
- [ ] No

### Does the program distribute orders evenly among available cooks using round-robin?
- [ ] Yes
- [ ] No

### Can new cooks connect dynamically without affecting the system?
- [ ] Yes
- [ ] No

### Does the program enforce maximum 10 cooks simultaneously constraint?
- [ ] Yes
- [ ] No

### Does each cook have a unique ID?
- [ ] Yes
- [ ] No

### Are unacknowledged messages redirected to other cooks when a cook disconnects?
- [ ] Yes
- [ ] No

## Order Status Tracking
### Does the program implement order status tracking system?
- [ ] Yes
- [ ] No

### Does the program use publish/subscribe pattern (fanout exchange) for notifications?
- [ ] Yes
- [ ] No

### Does the program track all required statuses (received, cooking, ready, delivered)?
- [ ] Yes
- [ ] No

### Do notifications contain order_id, status, and timestamp?
- [ ] Yes
- [ ] No

### Does the program provide API endpoint for checking order status?
- [ ] Yes
- [ ] No

### Can clients query the status of their orders?
- [ ] Yes
- [ ] No

## Order Types and Routing
### Does the program support all three order types (dine-in, takeout, delivery)?
- [ ] Yes
- [ ] No

### Does the program use topic exchange with correct routing keys (orders.kitchen.{type})?
- [ ] Yes
- [ ] No

### Does the program implement different cooking times for different order types?
- [ ] Yes
- [ ] No

### Does the program validate order type on creation?
- [ ] Yes
- [ ] No

### Do delivery orders require and validate address information?
- [ ] Yes
- [ ] No

### Can cooks listen to specific order types only?
- [ ] Yes
- [ ] No

## Priority Queue Implementation
### Does the program support order prioritization using RabbitMQ priority queues?
- [ ] Yes
- [ ] No

### Are queues configured with x-max-priority parameter?
- [ ] Yes
- [ ] No

### Does the program implement correct priority levels (1-normal, 5-medium, 10-high)?
- [ ] Yes
- [ ] No

### Are VIP clients automatically assigned high priority?
- [ ] Yes
- [ ] No

### Are large orders (>$50) automatically assigned medium priority?
- [ ] Yes
- [ ] No

### Can urgent orders be manually marked as priority?
- [ ] Yes
- [ ] No

### Do cooks receive orders in priority order?
- [ ] Yes
- [ ] No

## Service Modes Implementation
### Does the program implement order-service mode with REST API?
- [ ] Yes
- [ ] No

### Does the program implement kitchen-worker mode?
- [ ] Yes
- [ ] No

### Does the program implement tracking-service mode?
- [ ] Yes
- [ ] No

### Does the program implement notification-subscriber mode?
- [ ] Yes
- [ ] No

### Does the program support --setup-queues command for RabbitMQ initialization?
- [ ] Yes
- [ ] No

## Configuration and Command Line Arguments
### Does the program support --help flag with comprehensive usage information?
- [ ] Yes
- [ ] No

### Does the program support --rabbitmq connection parameter?
- [ ] Yes
- [ ] No

### Does the program support --port parameter for order service?
- [ ] Yes
- [ ] No

### Does the program support --worker-id parameter for kitchen workers?
- [ ] Yes
- [ ] No

### Does the program support --order-types parameter for selective processing?
- [ ] Yes
- [ ] No

### Does the program support --customer parameter for notification subscription?
- [ ] Yes
- [ ] No

## Error Handling and Reliability
### Does the program handle RabbitMQ connection failures gracefully?
- [ ] Yes
- [ ] No

### Does the program implement proper error messages with clear reasons?
- [ ] Yes
- [ ] No

### Does the program handle invalid JSON in API requests?
- [ ] Yes
- [ ] No

### Does the program handle missing required fields in orders?
- [ ] Yes
- [ ] No

### Does the program handle network disconnections properly?
- [ ] Yes
- [ ] No

### Does the program implement proper cleanup on shutdown?
- [ ] Yes
- [ ] No

## API Endpoints
### Does the program implement POST /orders endpoint for order creation?
- [ ] Yes
- [ ] No

### Does the program implement GET /orders/{id}/status endpoint?
- [ ] Yes
- [ ] No

### Do API responses follow the specified JSON format?
- [ ] Yes
- [ ] No

### Does the program handle HTTP request validation properly?
- [ ] Yes
- [ ] No

### Does the program return appropriate HTTP status codes?
- [ ] Yes
- [ ] No

## RabbitMQ Integration
### Does the program properly declare and configure exchanges?
- [ ] Yes
- [ ] No

### Does the program properly declare and configure queues?
- [ ] Yes
- [ ] No

### Does the program implement proper message acknowledgment?
- [ ] Yes
- [ ] No

### Does the program handle RabbitMQ channel closures?
- [ ] Yes
- [ ] No

### Does the program implement proper connection recovery?
- [ ] Yes
- [ ] No

## Concurrency and Performance
### Does the program handle concurrent order processing efficiently?
- [ ] Yes
- [ ] No

### Does the program prevent race conditions in shared resources?
- [ ] Yes
- [ ] No

### Does the program implement proper resource cleanup?
- [ ] Yes
- [ ] No

### Does the program maintain performance under load?
- [ ] Yes
- [ ] No

## Project Defense

### Can the team explain their RabbitMQ exchange and queue design decisions?
- [ ] Yes
- [ ] No

### Can the team explain how they implemented message routing patterns?
- [ ] Yes
- [ ] No

### Can the team demonstrate understanding of work queue vs pub/sub patterns?
- [ ] Yes
- [ ] No

### Can the team explain their error handling and reliability approach?
- [ ] Yes
- [ ] No

### Can the team demonstrate the system working with multiple workers?
- [ ] Yes
- [ ] No

### Can the team explain how they implemented priority queues?
- [ ] Yes
- [ ] No

## Detailed Feedback

### What was great? What you liked the most about the program and the team performance?

### What could be better? How those improvements could positively impact the outcome?