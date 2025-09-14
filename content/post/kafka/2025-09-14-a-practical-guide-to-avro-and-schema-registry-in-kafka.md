---
title: "A Practical Guide to Avro and Schema Registry in Kafka"
date: "2025-09-14T00:00:00Z"
draft: false
tags:
  - kafka
  - avro
  - schema-registry
categories:
  - Tutorial
author: "phongthien"
---

# A Practical Guide to Avro and Schema Registry in Kafka

## 1. Đặt vấn đề

Trong hệ thống **event streaming** với Kafka, dữ liệu được truyền dưới dạng message giữa producer và consumer. Một vấn đề lớn xuất hiện khi **dữ liệu thay đổi cấu trúc (schema evolution)**:

- Producer thêm/bớt field trong message.
- Consumer cũ chưa được nâng cấp kịp thời.
- Nguy cơ consumer không đọc được dữ liệu, gây lỗi toàn hệ thống.

Ví dụ: Producer ban đầu gửi `{id, name}`, sau đó nâng cấp thêm `email`. Nếu consumer chưa biết field mới thì phải làm sao?


## 2. Cách giải quyết

### 2.1. Avro là gì?

**Apache Avro** là một **serialization framework** mã nguồn mở, tối ưu cho việc lưu trữ và truyền tải dữ liệu:

- **Dựa trên schema**: định nghĩa bằng JSON.
- **Hiệu năng cao**: nhị phân, nhỏ gọn hơn JSON/XML.
- **Đa ngôn ngữ**: sinh code cho Java, Go, Python, C#…
- **Hỗ trợ schema evolution**: thêm/bớt field vẫn đọc được dữ liệu cũ nếu có `default`.

Ví dụ schema Avro:

```json
{
  "type": "record",
  "name": "User",
  "fields": [
    {"name": "id", "type": "int"},
    {"name": "name", "type": "string"},
    {"name": "email", "type": ["null","string"], "default": null}
  ]
}

```

### 2.2. Lợi ích khi sử dụng Avro trong Kafka

- **Hiệu năng & tiết kiệm**: dữ liệu nhị phân gọn nhẹ, giảm chi phí lưu trữ và băng thông.
- **Đảm bảo tính nhất quán**: tất cả dữ liệu đều theo schema đã định nghĩa.
- **Hỗ trợ nâng cấp linh hoạt**: dễ dàng thêm/bớt field mà không phá vỡ hệ thống.
- **Ngôn ngữ trung lập**: dễ dàng tích hợp microservices viết bằng nhiều ngôn ngữ khác nhau.
- **Kết hợp Schema Registry**: chỉ cần gửi schema ID thay vì toàn bộ schema, giúp quản lý tập trung và version hóa rõ ràng.

### 2.3. Schema Registry

Schema Registry là một service trung tâm dùng để:

- Lưu trữ và quản lý **version schema**.
- Đảm bảo **tính tương thích** khi schema thay đổi.
- Giúp producer và consumer giao tiếp an toàn thông qua schema ID.

### 2.4. Các chế độ tương thích (Compatibility Modes)

Khi cập nhật schema, Schema Registry sẽ kiểm tra tính tương thích dựa trên mode đã cấu hình. Có 7 chế độ chính:

| Compatibility Type | Changes allowed | Check against which schemas | Upgrade first |
| --- | --- | --- | --- |
| **BACKWARD** | Delete fields, Add optional fields | Last version | Consumers |
| **BACKWARD_TRANSITIVE** | Delete fields, Add optional fields | All previous versions | Consumers |
| **FORWARD** | Add fields, Delete optional fields | Last version | Producers |
| **FORWARD_TRANSITIVE** | Add fields, Delete optional fields | All previous versions | Producers |
| **FULL** | Add optional fields, Delete optional fields | Last version | Any order |
| **FULL_TRANSITIVE** | Add optional fields, Delete optional fields | All previous versions | Any order |
| **NONE** | Tất cả thay đổi đều được chấp nhận | Không kiểm tra | Tuỳ |

### Ý nghĩa thực tế:

- **BACKWARD**: phổ biến nhất trong streaming (consumer upgrade chậm hơn producer).
- **BACKWARD_TRANSITIVE**: an toàn hơn, đảm bảo schema mới tương thích với toàn bộ lịch sử schema.
- **FORWARD**: phù hợp với batch/ETL (producer upgrade chậm hơn consumer).
- **FULL** và **FULL_TRANSITIVE**: dùng khi hệ thống yêu cầu **tương thích 2 chiều tuyệt đối**, upgrade producer hoặc consumer theo bất kỳ thứ tự nào.
- **NONE**: chỉ nên dùng trong **dev/test**, vì dễ gây crash consumer.



## 3. Thực hiện

### 3.1. Khởi chạy Kafka + Schema Registry

Ví dụ Docker Compose:

```yaml
version: '2'
services:
  kafka:
    image: 'confluentinc/cp-kafka:7.6.0'
    container_name: kafka
    networks:
      dev-net:
        aliases:
          - kafka.remote
    ports:
      - '9092:9092'
    environment:
      KAFKA_PROCESS_ROLES: 'broker,controller'
      KAFKA_NODE_ID: 1
      KAFKA_LISTENERS: 'PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093'
      KAFKA_ADVERTISED_LISTENERS: 'PLAINTEXT://kafka.remote:9092'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: 'CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT'
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CONTROLLER_QUORUM_VOTERS: '1@kafka.remote:9093'
      
      
  schema-registry:
    image: 'confluentinc/cp-schema-registry:latest'
    environment:
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: 'kafka:9092'
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_LISTENERS: 'http://0.0.0.0:8081'

```

### 3.3. Code mẫu Golang

### Producer (gửi Avro message với schema ID)

```go
package main

import (
    "context"
    "fmt"
    "log"

    "github.com/segmentio/kafka-go"
    "github.com/riferrei/srclient" // Schema Registry client
    avro "github.com/hamba/avro/v2"
)

func main() {
    // Tạo client schema registry
    client := srclient.CreateSchemaRegistryClient("http://localhost:8081")

    schema, err := client.GetLatestSchema("user-value")
    if err != nil {
        log.Fatal(err)
    }

    codec, err := avro.Parse(schema.Schema())
    if err != nil {
        log.Fatal(err)
    }

    // Serialize dữ liệu
    user := map[string]interface{}{
        "id":    1,
        "name":  "Alice",
        "email": "alice@example.com",
    }

    value, err := avro.Marshal(codec, user)
    if err != nil {
        log.Fatal(err)
    }

    // Prefix magic byte (0) + schema ID (4 bytes)
    encoded := make([]byte, 5+len(value))
    encoded[0] = 0
    encoded[1] = byte(schema.ID() >> 24)
    encoded[2] = byte(schema.ID() >> 16)
    encoded[3] = byte(schema.ID() >> 8)
    encoded[4] = byte(schema.ID())
    copy(encoded[5:], value)

    // Kafka producer
    w := &kafka.Writer{
        Addr:  kafka.TCP("localhost:9092"),
        Topic: "user-topic",
    }

    err = w.WriteMessages(context.Background(),
        kafka.Message{Value: encoded},
    )
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println("Sent user:", user)
}

```

### Consumer (deserialize theo schema từ Registry)

```go
package main

import (
    "context"
    "fmt"
    "log"

    "github.com/segmentio/kafka-go"
    "github.com/riferrei/srclient"
    avro "github.com/hamba/avro/v2"
)

func main() {
    client := srclient.CreateSchemaRegistryClient("http://localhost:8081")

    r := kafka.NewReader(kafka.ReaderConfig{
        Brokers: []string{"localhost:9092"},
        GroupID: "user-group",
        Topic:   "user-topic",
    })

    for {
        m, err := r.ReadMessage(context.Background())
        if err != nil {
            log.Fatal(err)
        }

        schemaID := int(m.Value[1])<<24 | int(m.Value[2])<<16 | int(m.Value[3])<<8 | int(m.Value[4])
        schema, err := client.GetSchema(schemaID)
        if err != nil {
            log.Fatal(err)
        }

        codec, err := avro.Parse(schema.Schema())
        if err != nil {
            log.Fatal(err)
        }

        payload := m.Value[5:]
        var result map[string]interface{}
        if err := avro.Unmarshal(codec, payload, &result); err != nil {
            log.Fatal(err)
        }

        fmt.Println("Received user:", result)
    }
}

```


## 4. Kết luận

Việc kết hợp **Avro + Schema Registry** trong Kafka mang lại nhiều lợi ích:

- **Avro**: serialization hiệu quả, nhỏ gọn, hỗ trợ schema evolution.
- **Schema Registry**: quản lý schema tập trung, đảm bảo compatibility.
- Giúp hệ thống **nâng cấp dần dần**, không cần update đồng loạt tất cả service.

👉 Lựa chọn **compatibility mode** phù hợp là chìa khóa:

- **BACKWARD / BACKWARD_TRANSITIVE**: event streaming, upgrade consumer chậm.
- **FORWARD / FORWARD_TRANSITIVE**: batch/ETL.
- **FULL / FULL_TRANSITIVE**: yêu cầu ổn định tuyệt đối.
- **NONE**: chỉ nên dùng trong dev/test.
