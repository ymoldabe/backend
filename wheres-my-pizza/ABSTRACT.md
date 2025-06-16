# wheres-my-pizza

## Learning Objectives

- Event-driven architecture
- RabbitMQ: work-queue, fan-out, dead-letter / retry queues, manual **ACK/NACK + QoS**
- Exchange types & routing keys
- Transactional Outbox pattern (**events** table) + PostgreSQL
- Idempotent message processing (Inbox tables)
- Docker-Compose orchestration
- Unit, contract & end-to-end testing

---

## Abstract

Build **wheres-my-pizza**, a three-service Go application that tracks a pizza order from placement to delivery.
All inter-service communication happens through RabbitMQ events—**no synchronous HTTP**.
A single transactional outbox keeps the DB and broker consistent; consumers stay idempotent via inbox tables.
The public REST API is **read-only** and lets clients query order status and full event history.
