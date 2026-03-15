---
title: "Transactional Outbox Pattern"
author: phongthien99
date: 2026-03-15 00:00:00 +0700
categories: ["Microservice pattern"]
tags: [Design-pattern]
math: true

---
# Transactional Outbox Pattern — Giải pháp đảm bảo tính nhất quán dữ liệu trong Microservices

---

## 1. Đặt vấn đề — Dual Write Problem

### Bối cảnh

Trong kiến trúc **Microservices**, một nghiệp vụ thường yêu cầu đồng thời hai thao tác:

- **Ghi dữ liệu vào database** (ví dụ: tạo đơn hàng).
- **Gửi event/message đến Message Broker** (Kafka, RabbitMQ…) để các service khác xử lý tiếp (ví dụ: gửi email xác nhận, trừ tồn kho…).

Vấn đề nằm ở chỗ: hai thao tác này **không nằm trong cùng một transaction**. Database có transaction riêng, Message Broker có transaction riêng. Không có cơ chế atomic nào đảm bảo cả hai cùng thành công hoặc cùng thất bại.

Đây chính là **Dual Write Problem** — kẻ thù thầm lặng trong mọi hệ thống phân tán.

### Minh họa vấn đề

```mermaid
sequenceDiagram
    participant S as 🖥 Order Service
    participant DB as 🗄 Database
    participant MQ as 📨 Message Broker

    S->>DB: 1. INSERT order
    Note over DB: ✅ Commit thành công

    S->>MQ: 2. Publish "OrderCreated" event
    Note over MQ: ❌ Broker down!<br/>Message bị mất!

    rect rgba(220, 38, 38, 0.1)
        Note over S,MQ: ⚠️ KHÔNG NHẤT QUÁN<br/>DB có order nhưng event không được gửi<br/>→ Các service khác không biết có order mới
    end
```

### Các kịch bản thất bại

| Kịch bản | Mô tả | Hậu quả |
| --- | --- | --- |
| **Broker lỗi** | DB ghi thành công, Broker không nhận được message | Downstream services mất event, dữ liệu bất đồng bộ |
| **Service crash** | DB ghi thành công, service crash trước khi publish | Event mãi mãi bị mất, không có cách retry |
| **Network partition** | DB ghi thành công, mạng đến Broker bị đứt | Message timeout, không chắc đã gửi hay chưa |
| **Publish trước DB** | Message được gửi nhưng DB rollback | Service khác nhận event "ảo" — dữ liệu phantom |

> **💡 Lưu ý:** Ngay cả khi bạn đặt publish message *trước* commit database, vấn đề vẫn tồn tại — chỉ là theo chiều ngược lại. Không có thứ tự nào giải quyết được vấn đề này nếu không thay đổi cách tiếp cận.
> 

---

## 2. Giải pháp — Outbox Pattern là gì?

### Ý tưởng cốt lõi

Outbox Pattern biến **hai bước riêng biệt** thành **một bước atomic duy nhất** bằng cách:

> **Thay vì publish message trực tiếp đến Broker, ta ghi message vào một bảng `outbox` trong cùng database, trong cùng một transaction với dữ liệu nghiệp vụ.** Sau đó, một tiến trình riêng biệt sẽ đọc bảng `outbox` và publish message đến Broker.
> 

Vì cả dữ liệu nghiệp vụ và message đều được ghi vào **cùng một database**, ta có thể tận dụng **ACID transaction** của database để đảm bảo tính atomic.

### So sánh: Trước vs Sau khi áp dụng

```mermaid
graph LR
    subgraph "❌ KHÔNG CÓ Outbox Pattern"
        A1[Order Service] -->|1. Write| B1[(Database)]
        A1 -->|2. Publish| C1[Message Broker]
        style C1 stroke:#dc2626,stroke-dasharray: 5 5
    end

    subgraph "✅ CÓ Outbox Pattern"
        A2[Order Service] -->|"1. Write order + outbox<br/>(single transaction)"| B2[(Database)]
        D2[Relay/CDC] -->|2. Read outbox| B2
        D2 -->|3. Publish| C2[Message Broker]
    end
```

### Tại sao Outbox Pattern hoạt động?

Outbox Pattern hoạt động dựa trên một nguyên tắc đơn giản nhưng mạnh mẽ:

- **Atomic write:** Dữ liệu nghiệp vụ và outbox message được ghi trong cùng một DB transaction. Nếu transaction rollback → cả hai đều rollback. Nếu commit → cả hai đều tồn tại.
- **Guaranteed delivery:** Một tiến trình riêng (Polling Publisher hoặc CDC) liên tục quét bảng outbox và publish message. Nếu Broker down, message vẫn nằm trong outbox, chờ được gửi lại.
- **At-least-once semantics:** Message sẽ được gửi *ít nhất một lần*. Consumer phía nhận cần xử lý idempotent.

---

## 3. Thiết kế hệ thống (Design)

### 3.1 Kiến trúc tổng quan

```mermaid
flowchart TB
    subgraph Service["🖥 Order Service"]
        API[API Handler]
        BL[Business Logic]
    end

    subgraph DB["🗄 Database"]
        OT[orders table]
        OB[outbox table]
    end

    subgraph Relay["⚙️ Message Relay"]
        POLL[Polling Publisher]
        CDC[CDC Connector<br/>Debezium]
    end

    subgraph Broker["📨 Message Broker"]
        KAFKA[Kafka / RabbitMQ]
    end

    subgraph Consumers["📥 Downstream Services"]
        INV[Inventory Service]
        NOTIF[Notification Service]
        PAY[Payment Service]
    end

    API --> BL
    BL -->|"Single TX"| OT
    BL -->|"Single TX"| OB

    POLL -.->|"Option A: Polling"| OB
    CDC -.->|"Option B: CDC"| OB

    POLL --> KAFKA
    CDC --> KAFKA

    KAFKA --> INV
    KAFKA --> NOTIF
    KAFKA --> PAY

    style OB fill:#fef3c7,stroke:#f59e0b,stroke-width:2px
    style KAFKA fill:#dbeafe,stroke:#3b82f6,stroke-width:2px
```

### 3.3 Luồng xử lý chi tiết

```mermaid
sequenceDiagram
    participant C as 👤 Client
    participant S as 🖥 Order Service
    participant DB as 🗄 Database
    participant R as ⚙️ Message Relay
    participant K as 📨 Kafka
    participant INV as 📦 Inventory Service

    C->>S: POST /orders (tạo đơn hàng)

    rect rgba(34, 197, 94, 0.08)
        Note over S,DB: 🔒 BEGIN TRANSACTION
        S->>DB: INSERT INTO orders (...)
        S->>DB: INSERT INTO outbox (event_type='OrderCreated', status='PENDING')
        Note over S,DB: ✅ COMMIT
    end

    S-->>C: 201 Created

    loop Polling mỗi 5 giây
        R->>DB: SELECT * FROM outbox WHERE status = 'PENDING'
        DB-->>R: Trả về danh sách events
    end

    R->>K: Publish "OrderCreated" event
    K-->>R: ACK

    R->>DB: UPDATE outbox SET status='SENT', processed_at=now()

    K->>INV: Deliver "OrderCreated"
    INV->>INV: Trừ tồn kho (idempotent check)
```

### 3.4 Hai chiến lược đọc Outbox

```mermaid
flowchart LR
    subgraph A["Option A: Polling Publisher"]
        direction TB
        T1[⏰ Scheduler<br/>chạy mỗi N giây]
        T2[📖 Query outbox<br/>WHERE status = PENDING]
        T3[📨 Publish to Broker]
        T4[✏️ Update status = SENT]
        T1 --> T2 --> T3 --> T4
    end

    subgraph B["Option B: CDC — Change Data Capture"]
        direction TB
        S1[📝 DB ghi outbox row]
        S2[📋 Transaction Log<br/>WAL / Binlog]
        S3[🔌 Debezium Connector]
        S4[📨 Publish to Kafka]
        S1 --> S2 --> S3 --> S4
    end
```

| Tiêu chí | Polling Publisher | CDC (Debezium) |
| --- | --- | --- |
| **Độ trễ** | Cao (phụ thuộc polling interval) | Gần real-time |
| **Tải lên DB** | Cao (query liên tục) | Thấp (đọc từ WAL/binlog) |
| **Độ phức tạp** | Thấp — dễ implement | Cao — cần setup Debezium, Kafka Connect |
| **Phù hợp** | Hệ thống nhỏ, traffic thấp | Hệ thống lớn, yêu cầu latency thấp |
| **Rủi ro** | Duplicate nếu crash giữa publish và update | Cần quản lý offset Kafka Connect |

---

## 5. Kết luận

Outbox Pattern là một giải pháp **đơn giản nhưng hiệu quả** để giải quyết Dual Write Problem — một trong những thách thức phổ biến nhất khi xây dựng hệ thống Microservices.

Thay vì cố gắng thực hiện hai thao tác phân tán một cách atomic (điều bất khả thi nếu không có distributed transaction), Outbox Pattern **tận dụng ACID transaction** của database sẵn có để đảm bảo tính nhất quán, sau đó dùng một relay process riêng biệt để đưa message đến Broker.

Outbox Pattern không loại bỏ hoàn toàn failure, nhưng nó biến failure thành thứ có thể **phát hiện, retry và phục hồi** — và đó chính là điều quan trọng nhất khi xây dựng hệ thống đáng tin cậy.