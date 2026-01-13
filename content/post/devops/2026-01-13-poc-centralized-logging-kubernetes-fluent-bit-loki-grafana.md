---
title: "POC: Centralized Logging Implementation on Kubernetes with Fluent Bit, Grafana Loki, and Grafana"
date: "2026-01-13T00:00:00Z"
draft: false
tags:
  - k8s
  - helm
  - devops
categories:
  - Devops
author: "phongthien"
---
# POC: Centralized Logging Implementation on Kubernetes with Fluent Bit, Grafana Loki, and Grafana

## 1. Problem Statement

Hệ thống hiện tại bao gồm nhiều service chạy phân tán (microservices / container / nhiều node), mỗi service ghi log cục bộ hoặc theo cơ chế riêng lẻ. Log được lưu trữ rải rác trên nhiều máy chủ, pod hoặc file system khác nhau, **không có một điểm tập trung để quan sát và truy vấn**.

Khi xảy ra sự cố (lỗi application, timeout, performance degradation), việc truy vết nguyên nhân gặp nhiều khó khăn do:

- Log bị phân tán theo service, node và thời gian
- Không thể truy vấn log xuyên suốt nhiều service trong cùng một request
- Phải truy cập thủ công từng máy/pod để kiểm tra log
- Không có khả năng filter, search và correlation theo label (service, env, request_id…)

Điều này dẫn đến:

- Thời gian xác định nguyên nhân sự cố (MTTR) kéo dài
- Khó debug các lỗi liên quan đến luồng request liên service
- Gia tăng rủi ro vận hành và ảnh hưởng đến độ ổn định hệ thống
- Đội vận hành phụ thuộc nhiều vào thao tác thủ công, thiếu khả năng quan sát tập trung

Do đó, cần một giải pháp **centralized logging** có khả năng:

- Thu thập log từ nhiều nguồn phân tán
- Chuẩn hoá, gắn label và lưu trữ tập trung
- Cho phép truy vấn nhanh, chính xác theo thời gian và ngữ cảnh
    
    nhằm cải thiện khả năng quan sát và rút ngắn thời gian xử lý sự cố.
    



## 2. Objectives

POC này nhằm đánh giá tính khả thi của giải pháp centralized logging trong môi trường Kubernetes, với các mục tiêu cụ thể như sau:

- **Xử lý log đầu vào ~5GB/ngày**
    
    Hệ thống có khả năng thu thập, ingest và lưu trữ log từ nhiều service chạy trên Kubernetes với tổng dung lượng khoảng 5GB mỗi ngày mà không xảy ra mất log hoặc suy giảm hiệu năng đáng kể.
    
- **Đáp ứng chính sách lưu trữ (Retention) 7 ngày**
    
    Log được lưu trữ tập trung trong vòng 7 ngày, đảm bảo khả năng truy vấn, tra cứu phục vụ mục đích vận hành, giám sát và điều tra sự cố.
    
- **Chạy ổn định trên nền tảng Kubernetes**
    
    Giải pháp được triển khai và vận hành ổn định trong Kubernetes cluster, đảm bảo các pod hoạt động bình thường, tự phục hồi khi restart và không ảnh hưởng đến workload hiện có.
    
- **Khả năng mở rộng lưu trữ thông qua Object Storage (S3-compatible)**
    
    Dữ liệu log được lưu trữ trên object storage (S3 hoặc S3-compatible như MinIO), cho phép mở rộng dung lượng linh hoạt, giảm phụ thuộc vào storage cục bộ của node và tối ưu chi phí khi mở rộng hệ thống.
    


## 3. Proposed Solution

Giải pháp được đề xuất sử dụng mô hình **centralized logging** dựa trên bộ công cụ **Fluent Bit – Grafana Loki – Grafana**, triển khai trên nền tảng Kubernetes nhằm giải quyết vấn đề log phân tán và khó truy vết.


### Kiến trúc tổng thể

- **Fluent Bit**
    - Triển khai dưới dạng **DaemonSet** trên Kubernetes
    - Thu thập log từ stdout/stderr của container
    - Gắn metadata Kubernetes (namespace, pod, container, label)
    - Gửi log tới Loki thông qua HTTP API
- **Grafana Loki (Microservices Mode)**
    - Loki được triển khai theo **mô hình microservices (Distributed mode)**, trong đó mỗi thành phần đảm nhiệm một vai trò riêng biệt:
        - **Distributor**: nhận log từ Fluent Bit và phân phối tới các Ingester
        - **Ingester**: buffer và ghi log xuống storage
        - **Querier**: xử lý truy vấn log từ Grafana
        - **Query Frontend**: tối ưu và phân phối truy vấn
        - **Compactor**: quản lý retention và compact dữ liệu
        - **Index Gateway / Ruler (tuỳ chọn trong POC)**
    - Dữ liệu log được lưu trữ trên **object storage S3-compatible**, cho phép mở rộng dung lượng linh hoạt và giảm phụ thuộc vào storage cục bộ
- **Grafana**
    - Cung cấp giao diện truy vấn, phân tích và trực quan hoá log
    - Hỗ trợ filter theo label, time range và service
    - Giúp truy vết log xuyên suốt nhiều service
        
        ```mermaid
        flowchart TD
            %% Ngoài Kubernetes Cluster
            ObjectStorage[(S3 / MinIO<br/>Object Storage)]
        
            subgraph K8S ["KUBERNETES CLUSTER"]
                subgraph CLIENTS ["Clients"]
                    FluentBit(["Fluent Bit / Agents"])
                    Grafana["Grafana UI"]
                end
        
                subgraph GATEWAY ["Gateway"]
                    Gateway("Loki Gateway<br/>nginx-unprivileged:1.29-alpine")
                end
        
                subgraph WRITE ["Write Path"]
                    Distributor("Distributor<br/>loki:3.6.3")
                    Ingester("Ingester<br/>loki:3.6.3")
                end
        
                subgraph READ ["Read Path"]
                    QueryFrontend("Query Frontend<br/>loki:3.6.3")
                    QueryScheduler("Query Scheduler<br/>loki:3.6.3")
                    Querier("Querier<br/>loki:3.6.3")
                end
        
                subgraph STORAGE ["Index & Compactor"]
                    IndexGateway("Index Gateway<br/>loki:3.6.3")
                    Compactor("Compactor<br/>loki:3.6.3")
                end
        
                subgraph CACHE ["Cache"]
                    ChunksCache{Chunks Cache<br/>memcached:1.6.39}
                    ResultsCache{Results Cache<br/>memcached:1.6.39}
                end
        
                subgraph CANARY ["Canary"]
                    Canary("Loki Canary<br/>loki-canary:3.6.3")
                end
            end
        
            %% Luồng ghi (write)
            FluentBit --> Gateway
            Gateway --> Distributor
            Distributor --> Ingester
            Ingester -- Push chunks & index --> ObjectStorage
        
            %% Luồng đọc (read)
            Grafana --> Gateway
            Gateway --> QueryFrontend
            QueryFrontend --> QueryScheduler
            QueryScheduler --> Querier
            Querier -.-> ObjectStorage
            Querier --> IndexGateway
        
            %% Index & compactor
            Ingester --> IndexGateway
            IndexGateway --> ObjectStorage
            Compactor --> ObjectStorage
        
            %% Cache usage
            Querier --> ResultsCache
            Ingester --> ChunksCache
        
            classDef canary fill:#fffbe7,stroke:#fbe292,stroke-width:2px;
            class Canary canary;
        
        ```
        


## 4. Tech Stack

| Nhóm | Công cụ / Thành phần | Image container | Mô tả |
| --- | --- | --- | --- |
| Log Aggregation | **Grafana Loki (Core services)** | `docker.io/grafana/loki:3.6.3` | Distributor, Ingester, Querier, Query-Frontend, Query-Scheduler, Index-Gateway, Compactor |
| Health Check | **Loki Canary** | `docker.io/grafana/loki-canary:3.6.3` | Ghi & query log mẫu để kiểm tra end-to-end |
| Log Collector | **Fluent Bit** | `cr.fluentbit.io/fluent/fluent-bit:latest` | Thu thập log container & gửi về Loki |
| Reverse Proxy | **Loki Gateway (Nginx)** | `docker.io/nginxinc/nginx-unprivileged:1.29-alpine` | Entry point cho ingestion & query |
| Cache (Chunks) | **Memcached** | `memcached:1.6.39-alpine` | Cache log chunks, giảm truy cập object storage |
| Cache (Results) | **Memcached** | `memcached:1.6.39-alpine` | Cache kết quả truy vấn LogQL |
| Metrics Exporter | **Memcached Exporter** | `prom/memcached-exporter:v0.15.4` | Expose metrics cho Prometheus |
| Visualization | **Grafana** | `docker.io/grafana/grafana:13.2.1` | Truy vấn & trực quan hóa log |
| Object Storage | **S3 / MinIO** | `minio/minio:latest` *(nếu self-host)* | Lưu trữ log dài hạn |
| Sidecar Dashboard | **Grafana Sidecar** | `quay.io/kiwigrid/k8s-sidecar:2.2.1`  | Tự động load dashboard ConfigMap vào Grafana |

## 5. Môi trường triển khai POC

- Môi trường triển khai: **k3d (Kubernetes in Docker)**
- Số cluster: **01**
- Kiến trúc cluster:
    - **01 Master node**
    - **02 Worker nodes**

### 5.1 Namespace

| Namespace | Thành phần | Helm chart sử dụng | Ghi chú |
| --- | --- | --- | --- |
| `logging` | Grafana Loki (microservices  mode) | `grafana/loki` | Hệ thống log tập trung |
| `logging-agent` | Fluent Bit | `grafana/fluent-bit` | Thu thập log từ node/pod |
| `monitoring` | Grafana | `grafana/grafana` | Truy vấn & hiển thị log |

## 6. Results

- Hệ thống **đạt mục tiêu chính**: thu thập, lưu trữ và **truy vấn log tập trung** trên Kubernetes.
- Log từ các ứng dụng được ingest và hiển thị đầy đủ trên Grafana thông qua Loki.


## 7. Conclusion 

POC chứng minh hệ thống Loki đáp ứng tốt nhu cầu xem và truy vấn log tập trung trong phạm vi thử nghiệm, là nền tảng phù hợp để mở rộng sang môi trường production khi cần.