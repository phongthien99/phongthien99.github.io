---
title: "Log Observability with Fluent Bit, Loki, and Grafana"
date: "2025-11-21T00:00:00Z"
draft: false
tags:
  - k8s
  - helm
  - devops
categories:
  - Devops
author: "phongthien"
---
# Log Observability with Fluent Bit, Loki, and Grafana

## **1. Đặt vấn đề**

Trong kỷ nguyên của hệ thống phân tán và microservices, log đã trở thành “dòng máu” nuôi dưỡng khả năng quan sát của toàn bộ nền tảng. Thế nhưng, chính bởi sự phân tán của ứng dụng — mỗi container, mỗi node, mỗi dịch vụ lại sinh log ở một nơi khác nhau — việc tập trung và phân tích log trở thành một thách thức không hề nhỏ.

Nhiều tổ chức vẫn phụ thuộc vào phương pháp xem log truyền thống: truy cập từng container, đọc từng file, hoặc sử dụng những giải pháp nặng nề như ELK. Dù mạnh mẽ, ELK lại tiêu tốn lượng tài nguyên khổng lồ, khó mở rộng, và chi phí vận hành không dễ chịu.

Trong bối cảnh đó, nhu cầu về một hệ thống gọn nhẹ hơn, dễ triển khai hơn và chi phí thấp hơn trở nên cấp bách. Một giải pháp có thể thu gom log từ nhiều nguồn, lưu trữ thông minh mà không “ngốn” tài nguyên như Elasticsearch, đồng thời vẫn đem lại trải nghiệm quan sát mạnh mẽ qua giao diện trực quan. Câu trả lời cho bài toán ấy chính là sự kết hợp giữa Fluent Bit, Loki và Grafana.

## 2. **Giải pháp: Fluent Bit → Loki → Grafana**

### **2.1 Kiến trúc tổng quan**

```
[Application Log]
        ↓
   Fluent Bit
 (collector)
        ↓
       Loki
 (storage/index)
        ↓
     Grafana
 (visualization)

```

### **2.2 Vai trò từng hệ thống**

- **Fluent Bit**: agent nhẹ, lấy log từ file/container và gửi về Loki qua HTTP.
- **Loki**: lưu log theo mô hình index tối giản → giảm chi phí.
- **Grafana**: dashboard trực quan, query bằng LogQL.

### **2.3 Ưu điểm của giải pháp**

| Tiêu chí | Fluent Bit + Loki + Grafana |
| --- | --- |
| Tài nguyên | Rất thấp |
| Triển khai | Dễ, chỉ cần Docker/K8s |
| Chi phí | Tối ưu hơn ELK |
| Tìm kiếm log | Mạnh với LogQL |
| Tương thích | Tuyệt vời với Kubernetes |

Bộ ba này đặc biệt phù hợp với doanh nghiệp muốn **tối ưu tài nguyên**, **giảm chi phí**, và **tăng khả năng observability**.

## 3. **Thực nghiệm triển khai bằng Docker Compose**

### **3.1 Mục tiêu thực nghiệm**

- Sinh log thủ công từ một container demo.
- Thu gom log bằng Fluent Bit.
- Lưu log tại Loki.
- Hiển thị log trên Grafana.

### **3.2 Cấu hình Docker Compose**

```yaml
version: "3.3"

networks:
  loki:
volumes:
  logs:

services:
  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml
    networks:
      - loki

  grafana:
    environment:
      - GF_PATHS_PROVISIONING=/etc/grafana/provisioning
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
      - GF_FEATURE_TOGGLES_ENABLE=alertingSimplifiedRouting,alertingQueryAndExpressionsStepMode
   
    entrypoint:
      - sh
      - -euc
      - |
        mkdir -p /etc/grafana/provisioning/datasources
        cat <<EOF > /etc/grafana/provisioning/datasources/ds.yaml
        apiVersion: 1
        datasources:
        - name: Loki
          type: loki
          access: proxy 
          orgId: 1
          url: http://loki:3100
          basicAuth: false
          isDefault: true
          version: 1
          editable: false
        EOF
        /run.sh
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    networks:
      - loki
  # App demo sinh log
  app:
    image: busybox
    command: sh -c "while true; do echo 'Hello World from demo app' >> /logs/app.log; sleep 2; done"
    volumes:
      - logs:/logs

  # Fluent Bit
  fluent-bit:
    image: fluent/fluent-bit:2.2
    volumes:
      - ./fluent-bit-config:/fluent-bit/etc
      - logs:/logs:ro
    depends_on:
      - app
      - loki
    networks:
      - loki
```

---

### **3.3 Cấu hình Fluent Bit**

`fluent-bit.conf`:

```
[SERVICE]
    Flush        2
    Log_Level    info
    Parsers_File parsers.conf

[INPUT]
    Name        tail
    Path        /logs/*.log
    Tag         app.*
    DB          /fluent-bit/tail.db
    Parser      docker

[FILTER]
    Name        record_modifier
    Match       *
    Record      stream /logs/app.log
    Record      source app

[OUTPUT]
    Name        loki
    Match       *
    host        loki  
    port        3100
    labels                 app=demo-app

```

**Ý nghĩa:**

Fluent Bit tail file log, gắn label `app=demo-app`, gửi về Loki.

---

### **3.4 Cấu hình Loki**

`loki.yaml`:

```yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096
  log_level: debug
  grpc_server_max_concurrent_streams: 1000

common:
  instance_addr: 127.0.0.1
  path_prefix: /tmp/loki
  storage:
    filesystem:
      chunks_directory: /tmp/loki/chunks
      rules_directory: /tmp/loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

query_range:
  results_cache:
    cache:
      embedded_cache:
        enabled: true
        max_size_mb: 100

limits_config:
  metric_aggregation_enabled: true

schema_config:
  configs:
    - from: 2020-10-24
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

pattern_ingester:
  enabled: true
  metric_aggregation:
    loki_address: localhost:3100

ruler:
  alertmanager_url: http://localhost:9093

frontend:
  encoding: protobuf

# By default, Loki will send anonymous, but uniquely-identifiable usage and configuration
# analytics to Grafana Labs. These statistics are sent to https://stats.grafana.org/
#
# Statistics help us better understand how Loki is used, and they show us performance
# levels for most users. This helps us prioritize features and documentation.
# For more information on what's sent, look at
# https://github.com/grafana/loki/blob/main/pkg/analytics/stats.go
# Refer to the buildReport method to see what goes into a report.
#
# If you would like to disable reporting, uncomment the following lines:
#analytics:
#  reporting_enabled: false
```

---

### **3.5 Quan sát log trong Grafana**

1. Truy cập Grafana tại `http://localhost:3000`
2. Chọn Explore
3. Chon Loki
4. Query thử:

```
Expr: {app="demo-app"} |= ``

```

## 4. Kết luận

Trong bối cảnh chi phí hạ tầng luôn là mối quan tâm hàng đầu, giải pháp này mang lại sự cân bằng hiếm có giữa hiệu suất, chi phí và khả năng mở rộng. Nó không chỉ giúp đội ngũ vận hành thấy rõ hơn hoạt động của hệ thống, mà còn tạo nền tảng vững vàng cho các chiến lược quan sát sâu rộng hơn sau này, như alerting hoặc kết hợp metric–log–trace.

Nếu mục tiêu của bạn là xây dựng một hệ thống quan sát log tinh gọn, dễ quản trị và hiệu quả, bộ Fluent Bit – Loki – Grafana xứng đáng nằm trong danh sách ưu tiên hàng đầu.