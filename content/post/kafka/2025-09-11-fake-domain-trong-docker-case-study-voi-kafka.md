---
title: "Fake Domain trong Docker: Case Study với Kafka"
date: 2025-09-11 10:17:00 +0800
draft: false
tags: ["kafka"]
categories: [ "Programming"]
author: "phongthien99"
---

# Fake Domain trong Docker: Case Study với Kafka

## 1. Đặt vấn đề

Trong môi trường **development**, chúng ta thường chạy nhiều service bằng Docker: database, message broker, API… để mô phỏng gần giống production. Các service này cần kết nối với nhau, đồng thời developer cũng muốn dùng tool từ **máy host** (máy thật) để test trực tiếp.

Vấn đề thường gặp:

- Nếu container A gọi container B bằng `localhost`, thì `localhost` chỉ trỏ tới chính container A, không phải container B.
- Nếu dùng **tên container** (ví dụ `kafka`), thì trong Docker network hoạt động tốt, nhưng từ máy host thì hostname này không resolve được.
- Nếu dùng **IP container**, thì mỗi lần restart container, IP có thể thay đổi → config dễ bị hỏng.
- Kết quả là developer thường phải duy trì **hai cấu hình khác nhau**: một cho client chạy trong Docker network, một cho client chạy trên host.

👉 Giải pháp là sử dụng **fake domain alias**:

- Docker cho phép gán alias cho container trong network.
- Các container cùng network gọi service qua alias này.
- Trên host, ta ánh xạ alias đó về `127.0.0.1` trong file `/etc/hosts`.

Nhờ vậy:

- Container và host cùng dùng một domain để kết nối.
- Config được thống nhất, giảm lỗi.
- Developer dev/test thuận tiện hơn, không phải chỉnh sửa config nhiều lần.

---

## 2. Case Study: Kafka trong Docker

### 2.1. Cấu hình Kafka với fake domain alias

Ví dụ trong `docker-compose.yml`:

```yaml
kafka:
  image: confluentinc/cp-kafka:7.6.0
  container_name: kafka
  networks:
    dev-net:
      aliases:
        - kafka.remote   # fake domain
  ports:
    - 9092:9092
  environment:
    KAFKA_PROCESS_ROLES: broker,controller
    KAFKA_NODE_ID: 1
    KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
    KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka.remote:9092
    KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
    KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
    KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
    KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka.remote:9093

```

Ở đây:

- Kafka **quảng bá** (advertise) địa chỉ `kafka.remote:9092`.
- Trong network `dev-net`, các container khác có thể gọi `kafka.remote:9092` mà không cần IP.

---

### 2.2. Kết nối từ máy host

Máy host không biết `kafka.remote` là gì, nên nếu chạy client Kafka từ host sẽ bị lỗi.

Giải pháp: thêm alias vào `/etc/hosts`:

```
127.0.0.1 kafka.remote

```

Khi đó:

- Container trong network: gọi Kafka bằng `kafka.remote:9092`.
- Host thật: cũng gọi Kafka bằng `kafka.remote:9092`.

Ví dụ:

```bash
# Từ container khác
kafka-console-producer.sh --bootstrap-server kafka.remote:9092 --topic test

# Từ host
kafka-console-consumer.sh --bootstrap-server kafka.remote:9092 --topic test --from-beginning

```

👉 Cả hai trường hợp đều dùng chung config `kafka.remote:9092`.

---

## 3. Kết luận

Trong môi trường dev, vấn đề hostname giữa container và host thường gây rắc rối khi kết nối đến các service như Kafka. Sử dụng **fake domain alias** là một giải pháp đơn giản nhưng hiệu quả:

- Container và host dùng **cùng một domain**.
- Config thống nhất, giảm lỗi khi chuyển đổi môi trường.
- Dev/test thuận tiện, gần giống với production (nơi thường dùng domain thay vì IP).

Đây là một thủ thuật nhỏ nhưng cực kỳ hữu ích để giữ cho môi trường development gọn gàng, nhất quán và thân thiện với developer.
