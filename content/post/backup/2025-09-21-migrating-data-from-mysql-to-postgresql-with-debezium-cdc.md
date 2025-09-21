---
title: "Migrating Data from MySQL to PostgreSQL with Debezium CDC"
date: "2025-09-21T00:00:00Z"
draft: false
tags:
  - Debezium
  - Database Migration
categories:
  - Database
  - DevOps
author: "phongthien"
---

# Migrating Data from MySQL to PostgreSQL with Debezium CDC

## 1. Đặt vấn đề

Nhiều doanh nghiệp bắt đầu với **MySQL** nhờ dễ triển khai và phổ biến. Khi dữ liệu lớn hơn, PostgreSQL trở thành lựa chọn ưu việt nhờ khả năng mở rộng, chuẩn SQL mạnh mẽ và tính năng phong phú.

**Thách thức**: Làm sao di chuyển dữ liệu từ MySQL sang PostgreSQL mà **không ảnh hưởng hệ thống đang hoạt động**?

- Dump & restore không khả thi với cơ sở dữ liệu lớn.
- Cần đảm bảo **toàn vẹn dữ liệu**, **đồng bộ liên tục**, **kiểm thử trước cutover**.

CDC (Change Data Capture) là giải pháp, cho phép **snapshot + đồng bộ realtime**.

---

## 2. Giải pháp

Sử dụng **Debezium + Kafka + JDBC Sink Connector**:

1. **Snapshot ban đầu**: Debezium chụp toàn bộ dữ liệu MySQL và push vào Kafka topic → PostgreSQL.
2. **CDC streaming**: Debezium theo dõi binlog MySQL, push các thay đổi realtime sang PostgreSQL.
3. **Cutover**: Chỉ cần switch connection ứng dụng sang PostgreSQL, downtime gần như bằng 0.

**Lưu ý**:

- Snapshot dữ liệu lớn cần cấu hình hợp lý (`snapshot.mode=initial`).
- Chuẩn hóa schema giữa MySQL và PostgreSQL (`TINYINT(1)` → `BOOLEAN`, `AUTO_INCREMENT` → `SEQUENCE`).

---

## 3. Thử nghiệm

### 3.1. Docker Compose

### 3.1.1 Kafka + Schema Registry + AKHQ 

```yaml
version: '3.8'
services:
  kafka:
    image: confluentinc/cp-kafka:7.6.0
    container_name: kafka
    hostname: kafka
    networks:
      traefik-ingress:
        aliases:
          - kafka.remote
    ports:
      - 9093:9093
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
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      CLUSTER_ID: yZmtfI8tQ1mNvY2dH0Ttng
      KAFKA_NUM_PARTITIONS: 3
      KAFKA_DEFAULT_REPLICATION_FACTOR: 1

  schema-registry:
    image: confluentinc/cp-schema-registry:7.0.1
    container_name: schema-registry
    hostname: schema-registry
    networks:
      traefik-ingress:
        aliases:
          - schema-registry.remote
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry.remote
      SCHEMA_REGISTRY_LISTENERS: http://0.0.0.0:8081
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: PLAINTEXT://kafka.remote:9092
    ports:
      - "8081:8081"

  akhq:
    image: tchiotludo/akhq:0.26.0
    container_name: akhq
    hostname: akhq
    networks:
      traefik-ingress:
        aliases:
          - akhq.remote
    environment:
      AKHQ_CONFIGURATION: |
        akhq:
          server:
            access-log: true
          connections:
            kafka-cluster:
              properties:
                bootstrap.servers: "kafka.remote:9092"
              schema-registry:
                url: "http://schema-registry.remote:8081"
    ports:
      - "9082:8080"

networks:
  traefik-ingress:
    external: true

```

### 3.1.2 Kafka Connect

```yaml
version: '3.7'
services:
  kafka-connect:
    image: debezium/connect:2.7.3.Final
    container_name: kafka-connect
    environment:
      BOOTSTRAP_SERVERS: "kafka.remote:9092"
      HOST_NAME: kafka-connect
      ADVERTISED_HOST_NAME: kafka-connect
      GROUP_ID: debezium-connect-cluster
      OFFSET_STORAGE_TOPIC: "debezium-connect-offsets"
      CONFIG_STORAGE_TOPIC: "debezium-connect-configs"
      STATUS_STORAGE_TOPIC: "debezium-connect-status"
      CONFIG_STORAGE_REPLICATION_FACTOR: 1
      OFFSET_STORAGE_REPLICATION_FACTOR: 1
      STATUS_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_PLUGIN_PATH: /kafka/connect
    networks:
      - traefik-ingress

networks:
  traefik-ingress:
    external: true

```

### 3.1.3 MySQL & PostgreSQL 

```yaml
version: '3.8'
services:
  mysql:
    image: mysql:8.0
    command: --default-authentication-plugin=mysql_native_password --server-id=1 --log-bin=mysql-bin --binlog-format=ROW --binlog-row-image=FULL
    container_name: source-mysql
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: appdb
      MYSQL_USER: appuser
      MYSQL_PASSWORD: apppass
    ports:
      - "3306:3306"
    restart: unless-stopped
    volumes:
      - mysql_data:/var/lib/mysql

  postgres:
    image: postgres:15
    container_name: target-postgres
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: appdb
    restart: unless-stopped
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  mysql_data:
  postgres_data:

networks:
  default:
    external:
      name: traefik-ingress

```

---

### 3.2. Tạo Connector và Sink

Debezium / Kafka Connect cung cấp REST API để tạo connector.

### 3.2.1 Tạo Source Connector (MySQL → Kafka)

```bash
curl -X POST http://localhost:8083/connectors \
-H "Content-Type: application/json" \
-d '{
    "name": "mysql-source-json",
    "config": {
        "connector.class": "io.debezium.connector.mysql.MySqlConnector",
        "tasks.max": "1",
        "database.hostname": "source-mysql",
        "database.port": "3306",
        "database.user": "appuser",
        "database.password": "apppass",
        "database.server.id": "1",
        "topic.prefix": "mysql-appdb",
        "database.include.list": "appdb",
        "table.include.list": "appdb.customers",
        "snapshot.mode": "initial",
        "schema.history.internal.kafka.bootstrap.servers": "kafka.remote:9092",
        "schema.history.internal.kafka.topic": "schema-changes.appdb",
        "key.converter": "org.apache.kafka.connect.json.JsonConverter",
        "key.converter.schemas.enable": "true",
        "value.converter": "org.apache.kafka.connect.json.JsonConverter",
        "value.converter.schemas.enable": "true"
    }
}'

```

### 3.2.2 Tạo Sink Connector (Kafka → PostgreSQL)

```bash
curl -X POST http://localhost:8083/connectors \
-H "Content-Type: application/json" \
-d '{
  "name": "postgres-sink-json",
  "config": {
    "connector.class": "io.debezium.connector.jdbc.JdbcSinkConnector",
    "tasks.max": "1",
    "topics": "mysql-appdb.appdb.customers",
    "connection.url": "jdbc:postgresql://postgres:5432/appdb",
    "connection.username": "postgres",
    "connection.password": "postgres",
    "insert.mode": "upsert",
    "delete.enabled": "true",
    "primary.key.mode": "record_key",
    "primary.key.fields": "id",
    "schema.evolution": "basic",
    "table.name.format": "customers_sink",
    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "key.converter.schemas.enable": "true",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "true",
    "transforms": "unwrap",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.unwrap.drop.tombstones": "false",
    "transforms.unwrap.delete.handling.mode": "rewrite"
  }
}'

```

> Sau khi gọi API, Debezium sẽ bắt đầu snapshot dữ liệu và CDC streaming, đồng bộ liên tục từ MySQL sang PostgreSQL.
> 

---

## 1. Thêm dữ liệu mới (INSERT)

```sql
INSERT INTO customers (name, email) VALUES
('Charlie', 'charlie@example.com'),
('David', 'david@example.com');

```

- Sau vài giây, kiểm tra PostgreSQL:

```sql
SELECT * FROM customers_sink;

```

- Bạn sẽ thấy 2 record mới được thêm vào, đúng với dữ liệu MySQL.

---

## 2. Cập nhật dữ liệu (UPDATE)

```sql
UPDATE customers
SET email = 'alice.smith@example.com'
WHERE name = 'Alice';

```

- Kiểm tra PostgreSQL:

```sql
SELECT * FROM customers_sink WHERE name = 'Alice';

```

- Email trong PostgreSQL cũng sẽ được cập nhật tương ứng.
- `unwrap` SMT đảm bảo rằng chỉ **after state** được ghi, không cần xử lý thủ công.

---

## 3. Xóa dữ liệu (DELETE)

```sql
DELETE FROM customers WHERE name = 'Bob';

```

- Kiểm tra PostgreSQL:

```sql
SELECT * FROM customers_sink;

```

## 4. Kết luận

- **Downtime gần như bằng 0** nhờ snapshot + CDC.
- Toàn vẹn dữ liệu được đảm bảo trong suốt quá trình di chuyển.
- Quá trình cutover chỉ là **switch connection**, không ảnh hưởng người dùng.
