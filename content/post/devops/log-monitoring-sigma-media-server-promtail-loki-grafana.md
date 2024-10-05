---
title: Log Monitoring for Sigma Media Server Using Promtail, Loki, and Grafana
author: phongthien99
date: 2024-10-05 12:40:00 +0800
categories: [Fe]
tags: [monitor]
math: true
media_subpath: '/posts/20240503'
---
# Log Monitoring for Sigma Media Server Using Promtail, Loki, and Grafana

## Đặt vấn đề

     Trong quá trình vận hành **Sigma Media Server**, việc kiểm tra log  là một nhiệm vụ quan trọng để đảm bảo hệ thống hoạt động ổn định. Tuy nhiên, các log này thường được lưu trữ tại thư mục `/var/log/sigma-machine`, và việc phải thường xuyên truy cập vào thư mục, sử dụng lệnh `cat` hoặc `tail` để theo dõi log không chỉ tốn thời gian mà còn gây khó khăn khi cần giám sát liên tục và xử lý log từ nhiều dịch vụ khác nhau. Điều này đặt ra nhu cầu cho một giải pháp giám sát log tập trung, giúp theo dõi và phân tích log nhanh chóng, hiệu quả hơn.

## Giải pháp

Để giải quyết vấn đề giám sát log của Sigma Media Server một cách hiệu quả, chúng ta sẽ triển khai một **log monitoring stack** gồm ba thành phần chính: **Promtail**, **Loki**, và **Grafana**.

1. **Promtail**: Đây là agent dùng để thu thập log từ Sigma Media Server. Promtail sẽ đọc các file log tại đường dẫn `/var/log/sigma-machine/`, sau đó chuyển chúng đến Loki để lưu trữ và xử lý. Việc này giúp tự động hóa quá trình lấy log, không cần truy cập thủ công vào thư mục log nữa.
2. **Loki**: Loki là hệ thống lưu trữ log được thiết kế tối ưu cho việc lưu trữ và truy vấn dữ liệu log. Với kiến trúc nhẹ và tích hợp tốt với Promtail, Loki cho phép lưu trữ log một cách có tổ chức và hiệu quả. Dữ liệu log từ nhiều dịch vụ khác nhau trong Sigma Media Server sẽ được lưu tại một nơi, dễ dàng quản lý và truy vấn.
3. **Grafana**: Để trực quan hóa log, Grafana sẽ kết nối với Loki như một nguồn dữ liệu. Từ đây, bạn có thể tạo các dashboard tùy chỉnh để theo dõi log theo thời gian thực, thiết lập cảnh báo, và phân tích log một cách nhanh chóng. Grafana cung cấp giao diện trực quan giúp dễ dàng tìm kiếm, lọc và giám sát log mà không cần lệnh dòng lệnh phức tạp

## Thực hiện

    Triển khai thực nghiệm qua docker-compose

### Bước 1: Tạo file Docker Compose

File Docker Compose cấu hình các dịch vụ chính gồm Sigma Media Server, Promtail, Loki và Grafana:

```yaml
version: "3.8"
services:
  server-01:
      image: registry.gviet.vn:5000/sigma-livestream/sigma-media-server:4.1.35
      privileged: true
      user: root 
      volumes:
       - var-log:/var/log/sigma-machine:rw
      networks:
        - traefik-ingress
  promtail:
    image: grafana/promtail:2.8.2
    volumes:
      - ./promtail-config.yml:/etc/promtail/promtail-config.yml
      - var-log:/var/log/sigma-media-server-01:ro
      - var-log-02:/var/log/sigma-media-server-02:ro
    command: -config.file=/etc/promtail/promtail-config.yml
    networks:
        - traefik-ingress
    depends_on:
      - loki 
  loki:
    image: grafana/loki:2.8.2
    volumes:
      - ./loki-config.yml:/etc/loki/local-config.yaml
    command: -config.file=/etc/loki/local-config.yaml
    networks:
        - traefik-ingress

  grafana:
    image: grafana/grafana
    ports:
      - 3000:3000
    networks:
        - traefik-ingress
 
  
volumes:
  var-log:

networks:
  traefik-ingress:
    external: true

```

### Bước 2: Cấu hình Promtail

Promtail được cấu hình để thu thập log từ các file cụ thể của Sigma Media Server:

```
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: log-service-01
    static_configs:
      - targets:
          - localhost
        labels:
          container_name: sigma-media-server-01
          __path__: /var/log/sigma-media-server-01/now.*
      - targets:
          - localhost
        labels:
          container_name: sigma-media-server-02
          __path__: /var/log/sigma-media-server-02/now.*

```

### Bước 3: Cấu hình Loki

Loki sẽ lưu trữ và quản lý log, được cấu hình như sau:

```bash
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

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

schema_config:
  configs:
    - from: 2024-10-01
      store: tsdb
      object_store: filesystem
      schema: v12
      index:
        prefix: index_
        period: 24h

```

### Bước 5: **Cấu Hình Grafana**

1. Truy cập **Grafana** tại `http://localhost:3000` trên trình duyệt.
2. Đăng nhập với thông tin mặc định:
    - **Username**: `admin`
    - **Password**: `admin`
3. Sau khi đăng nhập, thêm **Loki** làm **data source**:
    - Vào **Configuration** > **Data Sources** > **Add data source**.
    - Chọn **Loki**.
    - Trong phần URL, nhập `http://loki:3100`.
    - Nhấn **Save & Test** để xác minh kết nối.

Bạn có thể sử dụng json model này để sử dụng:

```yaml
{
  "annotations": {
    "list": [
      {
        "builtIn": 1,
        "datasource": {
          "type": "datasource",
          "uid": "grafana"
        },
        "enable": true,
        "hide": true,
        "iconColor": "rgba(0, 211, 255, 1)",
        "name": "Annotations & Alerts",
        "type": "dashboard"
      }
    ]
  },
  "description": "Universal and flexible dashboard for logging",
  "editable": true,
  "fiscalYearStartMonth": 0,
  "gnetId": 18042,
  "graphTooltip": 0,
  "id": 3,
  "links": [],
  "panels": [
    {
      "datasource": {
        "type": "loki",
        "uid": "adzuilqytpp1cc"
      },
      "description": "Live logs is a like 'tail -f' in a real time",
      "gridPos": {
        "h": 9,
        "w": 24,
        "x": 0,
        "y": 0
      },
      "id": 6,
      "options": {
        "dedupStrategy": "none",
        "enableLogDetails": true,
        "prettifyLogMessage": false,
        "showCommonLabels": false,
        "showLabels": true,
        "showTime": false,
        "sortOrder": "Descending",
        "wrapLogMessage": false
      },
      "targets": [
        {
          "datasource": {
            "type": "loki",
            "uid": "adzuilqytpp1cc"
          },
          "editorMode": "code",
          "expr": "{filename=\"/var/log/${container_name}/now.debug\"}",
          "hide": false,
          "queryType": "range",
          "refId": "A"
        }
      ],
      "title": "Debug log",
      "type": "logs"
    },
    {
      "datasource": {
        "type": "loki",
        "uid": "adzuilqytpp1cc"
      },
      "description": "Live logs is a like 'tail -f' in a real time",
      "gridPos": {
        "h": 9,
        "w": 24,
        "x": 0,
        "y": 9
      },
      "id": 4,
      "options": {
        "dedupStrategy": "none",
        "enableLogDetails": true,
        "prettifyLogMessage": false,
        "showCommonLabels": false,
        "showLabels": true,
        "showTime": false,
        "sortOrder": "Descending",
        "wrapLogMessage": false
      },
      "targets": [
        {
          "datasource": {
            "type": "loki",
            "uid": "adzuilqytpp1cc"
          },
          "editorMode": "code",
          "expr": "{filename=\"/var/log/${container_name}/now.nginx\"}",
          "hide": false,
          "queryType": "range",
          "refId": "A"
        }
      ],
      "title": "Nginx log",
      "type": "logs"
    },
    {
      "datasource": {
        "type": "loki",
        "uid": "adzuilqytpp1cc"
      },
      "description": "Live logs is a like 'tail -f' in a real time",
      "gridPos": {
        "h": 9,
        "w": 24,
        "x": 0,
        "y": 18
      },
      "id": 3,
      "options": {
        "dedupStrategy": "none",
        "enableLogDetails": true,
        "prettifyLogMessage": false,
        "showCommonLabels": false,
        "showLabels": true,
        "showTime": false,
        "sortOrder": "Descending",
        "wrapLogMessage": false
      },
      "targets": [
        {
          "datasource": {
            "type": "loki",
            "uid": "adzuilqytpp1cc"
          },
          "editorMode": "code",
          "expr": "{filename=~\"/var/log/${container_name}/now\\\\.origin\"}",
          "hide": false,
          "queryType": "range",
          "refId": "A"
        }
      ],
      "title": "Origin log",
      "type": "logs"
    },
    {
      "datasource": {
        "type": "loki",
        "uid": "adzuilqytpp1cc"
      },
      "description": "Live logs is a like 'tail -f' in a real time",
      "gridPos": {
        "h": 9,
        "w": 24,
        "x": 0,
        "y": 27
      },
      "id": 5,
      "options": {
        "dedupStrategy": "none",
        "enableLogDetails": true,
        "prettifyLogMessage": false,
        "showCommonLabels": false,
        "showLabels": true,
        "showTime": false,
        "sortOrder": "Descending",
        "wrapLogMessage": false
      },
      "targets": [
        {
          "datasource": {
            "type": "loki",
            "uid": "adzuilqytpp1cc"
          },
          "editorMode": "code",
          "expr": "{filename=\"/var/log/${container_name}/now.srs\"}",
          "hide": false,
          "queryType": "range",
          "refId": "A"
        }
      ],
      "title": "Srs log",
      "type": "logs"
    },
    {
      "datasource": {
        "type": "loki",
        "uid": "adzuilqytpp1cc"
      },
      "description": "Live logs is a like 'tail -f' in a real time",
      "gridPos": {
        "h": 9,
        "w": 24,
        "x": 0,
        "y": 36
      },
      "id": 2,
      "options": {
        "dedupStrategy": "none",
        "enableLogDetails": true,
        "prettifyLogMessage": false,
        "showCommonLabels": false,
        "showLabels": true,
        "showTime": false,
        "sortOrder": "Descending",
        "wrapLogMessage": false
      },
      "targets": [
        {
          "datasource": {
            "type": "loki",
            "uid": "adzuilqytpp1cc"
          },
          "editorMode": "code",
          "expr": "{filename=~\"/var/log/${container_name}/now\\\\.portal-local\"}",
          "hide": false,
          "queryType": "range",
          "refId": "A"
        }
      ],
      "title": "Portal log",
      "type": "logs"
    }
  ],
  "refresh": "10s",
  "schemaVersion": 39,
  "tags": [],
  "templating": {
    "list": [
      {
        "current": {
          "isNone": true,
          "selected": false,
          "text": "None",
          "value": ""
        },
        "datasource": {
          "type": "loki",
          "uid": "adzuilqytpp1cc"
        },
        "definition": "",
        "hide": 0,
        "includeAll": false,
        "label": "Container",
        "multi": false,
        "name": "container_name",
        "options": [],
        "query": {
          "label": "container_name",
          "refId": "LokiVariableQueryEditor-VariableQuery",
          "stream": "{container_name=~\".+\"}",
          "type": 1
        },
        "refresh": 1,
        "regex": "",
        "skipUrlSync": false,
        "sort": 0,
        "tagValuesQuery": "",
        "tagsQuery": "",
        "type": "query",
        "useTags": false
      },
      {
        "current": {
          "selected": false,
          "text": "",
          "value": ""
        },
        "hide": 0,
        "name": "search",
        "options": [
          {
            "selected": true,
            "text": "",
            "value": ""
          }
        ],
        "query": "",
        "skipUrlSync": false,
        "type": "textbox"
      }
    ]
  },
  "time": {
    "from": "now-15m",
    "to": "now"
  },
  "timepicker": {
    "refresh_intervals": [
      "10s",
      "30s",
      "1m",
      "5m",
      "15m",
      "30m",
      "1h",
      "2h",
      "1d"
    ]
  },
  "timezone": "",
  "title": "Monitor sigma server",
  "uid": "cdzv1t37ajzeoc",
  "version": 15,
  "weekStart": ""
}
```

Thay đổi các  uid của datasource tương ứng

```yaml
 "datasource": {
        "type": "loki",
        "uid": "adzuilqytpp1cc"
      },
```

## Kết luận

Việc sử dụng stack **Promtail-Loki-Grafana** giúp giám sát log của Sigma Media Server một cách tập trung, hiệu quả và trực quan. Nó giải quyết được bài toán theo dõi log từ nhiều dịch vụ khác nhau mà không cần thao tác thủ công phức tạp