---
title: "Hybrid Helm Chart: Combining Dependency Charts and Embedded Subcharts"
date: "2025-10-01T00:00:00Z"
draft: false
tags:
  - k8s
  - helm
  - devops
categories:
  - Devops
author: "phongthien"
---


# Hybrid Helm Chart: Combining Dependency Charts and Embedded Subcharts

## 1. Đặt vấn đề

Helm cho phép tổ chức chart thành nhiều thành phần nhỏ, gọi là **subchart**. Có hai cách phổ biến để sử dụng subchart:

1. **Dependency Chart** (theo tài liệu Helm):
    - Khai báo trong `Chart.yaml` → `dependencies`.
    - Được tải về từ một **Helm repository** hoặc **OCI registry**.
    - Ví dụ: Redis, PostgreSQL từ Bitnami.
2. **Embedded Subchart** (subchart nhúng / nội bộ):
    - Chart con được viết thủ công và đặt trực tiếp trong thư mục `charts/` của chart chính.
    - Dùng để quản lý dịch vụ nội bộ hoặc microservice riêng của ứng dụng.

Trong thực tế, nhiều dự án vừa cần tận dụng **Dependency Chart** (tái sử dụng chart cộng đồng chuẩn), vừa cần viết **Embedded Subchart** (dịch vụ nội bộ đặc thù).

👉 Vấn đề đặt ra: **làm sao kết hợp cả Dependency Chart và Embedded Subchart trong một release duy nhất để triển khai đồng bộ?**

---

## 2. Giải pháp

Sử dụng mô hình **Hybrid Helm Chart**:

- Chart chính đóng vai trò **Umbrella chart**.
- Kết hợp cả:
    - **Dependency Chart**: khai báo trong `dependencies`, fetch từ repo ngoài.
    - **Embedded Subchart**: viết thủ công và đặt trong `charts/`.

Với mô hình này:

- Tận dụng được chart ngoài (hạ tầng như DB, Cache, MQ).
- Đồng thời tổ chức dịch vụ nội bộ thành chart con, quản lý dễ dàng.
- Tất cả được triển khai tập trung trong **một release Helm**.

---

## 3. Thực hiện

### 3.1. Cấu trúc thư mục ví dụ

```
my-hybrid-app/
├── Chart.yaml
├── values.yaml
├── charts/
│   ├── redis-17.11.3.tgz      # Dependency Chart (fetch từ Bitnami repo)
│   └── api/                   # Embedded Subchart (microservice nội bộ)
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── deployment.yaml
│           └── service.yaml

```

- `redis` → **Dependency Chart**, fetch từ repo ngoài.
- `api` → **Embedded Subchart**, nằm inline trong `charts/`.

---

### 3.2. Chart.yaml (chart chính)

```yaml
apiVersion: v2
name: my-hybrid-app
version: 0.1.0

dependencies:
  - name: redis
    version: 17.11.3
    repository: "https://charts.bitnami.com/bitnami"

```

---

### 3.3. values.yaml

```yaml
# Config cho Dependency Chart (redis)
redis:
  architecture: standalone
  auth:
    enabled: false

# Config cho Embedded Subchart (api)
api:
  replicaCount: 2
  image: myregistry/my-api:1.0.0
  service:
    type: ClusterIP
    port: 8080

```

> Theo Helm docs: khi deploy, Helm sẽ merge values.yaml của chart chính với subchart theo key (redis:, api:).
> 

---

### 3.4. Quy trình triển khai

### Bước 1: Cập nhật dependency

Tải Dependency Chart từ repo ngoài:

```bash
helm dependency update ./my-hybrid-app

```

---

### Bước 2: Đóng gói chart

Đóng gói chart chính thành `.tgz`:

```bash
helm package ./my-hybrid-app

```

Kết quả tạo file:

```
my-hybrid-app-0.1.0.tgz

```

- Trong package sẽ có cả **Dependency Chart** (`redis-17.11.3.tgz`) và **Embedded Subchart** (`charts/api/`).
- Lưu ý: phải chạy `helm dependency update` trước, nếu không Dependency Chart sẽ thiếu trong package.

---

### Bước 3: Cài đặt hoặc nâng cấp release

Triển khai trực tiếp từ source:

```bash
helm upgrade --install my-hybrid ./my-hybrid-app -f values.yaml

```

Hoặc từ package `.tgz`:

```bash
helm upgrade --install my-hybrid my-hybrid-app-0.1.0.tgz -f values.yaml

```

👉 Kubernetes sẽ triển khai cả Redis (Dependency Chart) và API (Embedded Subchart) chỉ trong một release.

---

## 4. Kết luận

**Hybrid Helm Chart** là mô hình kết hợp cả **Dependency Chart** (chart ngoài từ repo/OCI) và **Embedded Subchart** (chart nội bộ inline).

- **Giải quyết vấn đề**: tái sử dụng chart chuẩn + triển khai dịch vụ custom.
- **Ưu điểm**: quản lý release tập trung, dễ mở rộng, tận dụng chart cộng đồng mà vẫn linh hoạt.
- **Ứng dụng**: phù hợp cho hệ thống microservices có cả hạ tầng chuẩn (DB, cache, message broker) và nhiều service riêng.