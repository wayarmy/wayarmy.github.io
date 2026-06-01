---
layout: post
title: "Message Queues & Streaming - Hàng đợi tin nhắn và streaming"
date: 2026-06-01
categories: [database]
tags: [sqs, sns, kafka, kinesis, pub-sub, event-driven]
---

## Mục lục
1. [Góc nhìn tổng quan - Bưu điện số](#overview)
2. [SQS Standard vs FIFO](#sqs)
3. [SNS - Pub/Sub notifications](#sns)
4. [Apache Kafka Architecture](#kafka)
5. [AWS Kinesis Data Streams](#kinesis)
6. [Pub/Sub Patterns](#pubsub)
7. [Dead Letter Queues](#dlq)
8. [Exactly-once Semantics](#exactly-once)
9. [Event-Driven Architecture](#event-driven)
10. [Tổng kết và comparison](#tong-ket)

---

## 1. Góc nhìn tổng quan {#overview}

### Ví dụ đời thường

Message queues giống **hệ thống bưu điện**:

- **Queue (SQS)** = hộp thư: người gửi bỏ thư vào, người nhận lấy ra khi có thể. Thư được xử lý rồi xóa.
- **Topic (SNS)** = bảng thông báo: dán 1 lần, TẤT CẢ người đăng ký đều thấy.
- **Kafka** = sổ nhật ký công chứng: mọi message được ghi lại vĩnh viễn, ai muốn đọc thì đọc từ vị trí mình muốn.
- **Dead Letter Queue** = phòng thư bị trả lại: thư không gửi được → chuyển vào đây để kiểm tra.

---

## 2. SQS Standard vs FIFO {#sqs}

```
SQS Standard:
- Throughput: unlimited
- Ordering: best-effort (not guaranteed)
- Delivery: at-least-once (có thể duplicate)
- Use: High throughput, order không quan trọng

SQS FIFO:
- Throughput: 300 msg/s (3000 with batching)
- Ordering: strict FIFO (per message group)
- Delivery: exactly-once (deduplication)
- Use: Order processing, banking

Message lifecycle:
1. Producer sends message → queue
2. Consumer polls → receives message (invisible to others)
3. Consumer processes → deletes message
4. If not deleted within Visibility Timeout → message reappears!

Key settings:
- Visibility Timeout: 30s-12h (time to process)
- Message Retention: 1min-14days
- Delay Queue: 0-15min (delay before visible)
- Long Polling: WaitTimeSeconds=20 (reduce empty responses)
```

---

## 3-4. SNS & Kafka {#sns}

### SNS (Simple Notification Service)
```
Fan-out pattern:
  Publisher → SNS Topic → Multiple subscribers
                       ├─→ SQS Queue 1 (order processing)
                       ├─→ SQS Queue 2 (analytics)
                       ├─→ Lambda (email notification)
                       └─→ HTTP endpoint (webhook)
```

### Apache Kafka
```
Kafka architecture:
┌─────────────────────────────────────────────┐
│              Kafka Cluster                   │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐   │
│  │ Broker 1 │ │ Broker 2 │ │ Broker 3 │   │
│  └──────────┘ └──────────┘ └──────────┘   │
│                                             │
│  Topic "orders":                            │
│  Partition 0: [msg1][msg2][msg3]→          │
│  Partition 1: [msg4][msg5][msg6]→          │
│  Partition 2: [msg7][msg8]→                │
└─────────────────────────────────────────────┘

Producers → Topics (partitioned) → Consumer Groups

Key concepts:
- Topics: Named feeds of messages
- Partitions: Parallelism unit (ordered within partition)
- Consumer Groups: Each message consumed by 1 member per group
- Offsets: Position tracking (consumer decides pace)
- Retention: Time-based (7 days default) or size-based
- Compaction: Keep only latest per key
```

---

## 5-10. Kinesis, Patterns, DLQ, Event-Driven {#kinesis}

### Dead Letter Queue
```
DLQ receives messages that fail processing after N retries:
- SQS: maxReceiveCount → DLQ
- Kafka: Error topic
- Purpose: Isolate failures, prevent blocking, debug later

Main Queue → Process → Success ✓
                    └→ Failure (retry N times)
                         └→ DLQ (investigate manually)
```

### Exactly-once Challenge
```
At-most-once: Fire and forget (may lose messages)
At-least-once: Retry until ACK (may duplicate)
Exactly-once: Deliver exactly 1 time (hardest!)

Achieving exactly-once:
1. Idempotent consumers (same message → same result)
2. Deduplication (SQS FIFO deduplication ID)
3. Transactional outbox pattern
4. Kafka: enable.idempotence=true + transactions
```

### Event-Driven Architecture
```
Request-Response:   A calls B, waits for response
Event-Driven:       A emits event, B/C/D react independently

Benefits:
- Loose coupling (services don't know about each other)
- Easy to add new consumers
- Temporal decoupling (process when ready)
- Natural audit trail

Patterns:
1. Event Notification (lightweight, just notify)
2. Event-Carried State Transfer (include all data in event)
3. Event Sourcing (events ARE the source of truth)
4. CQRS + Events (sync read models via events)
```

| System | Best For | Ordering | Throughput | Retention |
|--------|---------|----------|-----------|-----------|
| SQS | Simple queue, serverless | No/FIFO | High | 14 days |
| SNS | Fan-out notifications | No | High | None |
| Kafka | Streaming, replay, high-throughput | Per-partition | Very High | Configurable |
| Kinesis | AWS streaming, real-time analytics | Per-shard | High | 7 days |

---
