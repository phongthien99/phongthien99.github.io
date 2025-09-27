---
title: "Umbrella Chart in Kubernetes"
date: "2025-09-27T00:00:00Z"
draft: false
tags:
  - k8s
  - helm
  - devops
categories:
  - Devops
author: "phongthien"
---

# Umbrella Chart in Kubernetes

## **1. Đặt vấn đề**

Trong các ứng dụng Kubernetes hiện đại, một dự án thường không chỉ bao gồm một service đơn lẻ mà là tập hợp nhiều thành phần:

- Web frontend
- Backend API
- Database
- Cache (Redis, Memcached…)
- Message broker (RabbitMQ, Kafka…)

Khi triển khai bằng Helm, nếu tạo **một chart riêng cho từng service**, sẽ gặp một số khó khăn:

- Phải deploy từng chart riêng → mất thời gian, dễ sai sót.
- Quản lý cấu hình chung (password, hostname, phiên bản) khó khăn.
- Không dễ quản lý dependency giữa các service.

**Vấn đề đặt ra:**

> Làm sao gom nhiều Helm Chart con thành một đơn vị triển khai duy nhất, đồng bộ, dễ quản lý cấu hình?
> 


## **2. Giải quyết: Umbrella Chart**

**Umbrella Chart** là **Helm Chart “parent”** dùng để **gom nhiều Helm Chart con** lại thành một gói triển khai duy nhất.

### **Đặc điểm chính:**

1. **Modularity (Tính module hóa)**
    - Mỗi chart con là một module riêng: MariaDB, Redis, Frontend.
    - Umbrella chart gom lại thành một module cấp cao.
2. **Abstraction (Trừu tượng hóa)**
    - Che giấu chi tiết triển khai của chart con.
    - Người dùng chỉ cần deploy umbrella chart và config các giá trị chung.
3. **Encapsulation (Đóng gói)**
    - Chart con có templates và values riêng.
    - Umbrella chart expose một interface duy nhất (`values.yaml`) để cấu hình.
4. **Reusability (Tái sử dụng)**
    - Chart con có thể dùng lại nhiều umbrella chart hoặc dự án khác.
5. **Parameterization (Tham số hóa)**
    - Forward values (`import-values` hoặc `global`) → giống truyền tham số vào hàm.
    - Ví dụ: password, replicas, image tag.

---

## **3. Thực hành: Tạo Umbrella Chart**

### **Bước 1: Tạo Umbrella Chart**

```bash
helm create my-umbrella
rm -rf my-umbrella/templates/*

```

### **Bước 2: Khai báo dependency trong `Chart.yaml`**

```yaml
apiVersion: v2
name: my-umbrella
version: 0.1.0
dependencies:
  - name: mariadb
    version: 18.0.3
    repository: "https://charts.bitnami.com/bitnami"
    import-values:
      - child: ""
        parent: mariadb
  - name: redis
    version: 19.0.2
    repository: "https://charts.bitnami.com/bitnami"
    import-values:
      - child: ""
        parent: redis

```

### **Bước 3: Config `values.yaml`**

```yaml
mariadb:
  auth:
    rootPassword: "rootpass123"
    username: "myuser"
    password: "mypassword"
    database: "mydb"
  primary:
    persistence:
      enabled: false

redis:
  auth:
    password: "redisSecret123"
  replica:
    replicaCount: 2

```

### **Bước 4: Update dependency**

```bash
helm dependency update my-umbrella

```

### **Bước 5: Render template hoặc deploy**

```bash
helm template myapp ./my-umbrella
helm install myapp ./my-umbrella

```

- Khi deploy, MariaDB và Redis sẽ nhận config từ umbrella chart mà không cần chỉnh từng chart con.



## **4. Kết luận**

- **Umbrella Chart** giúp:
    - Quản lý nhiều chart con trong một release.
    - Forward giá trị chung dễ dàng.
    - Triển khai đồng bộ, giảm sai sót.
- **Khi nào nên dùng:**
    - Ứng dụng gồm nhiều service liên quan.
    - Muốn deploy nhiều chart con đồng thời.
    - Muốn maintain cấu hình tập trung.