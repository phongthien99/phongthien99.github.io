---
title: Guide to Installing and Deploying Helm Chart from a Git Repository
author: phongthien99
date: 2025-04-30 10:40:00 +0800
categories: [Devops]
tags: [ansible]
math: true
media_subpath: '/posts/20240503'
---
# Guide to Installing and Deploying Helm Chart from a Git Repository

Trong bài viết này, chúng ta sẽ tìm hiểu cách sử dụng plugin [`helm-git`](https://github.com/aslafy-z/helm-git) để cài đặt và triển khai Helm chart trực tiếp từ Git repository. Cách làm này giúp bạn linh hoạt hơn trong việc quản lý các chart, đặc biệt khi cần triển khai từ nguồn Git riêng thay vì Helm repository thông thường. Bài viết sẽ hướng dẫn từ bước cài đặt plugin, thêm Git repo, chỉ định phiên bản chart với `ref`, cho đến cách sử dụng đường dẫn `@charts/...`.

---

## 🔧 Bước 1: Cài Đặt Plugin `helm-git`

Plugin `helm-git` giúp Helm xử lý Git repository như một nguồn Helm chart. Để cài đặt, chạy lệnh sau:

```bash

helm plugin install https://github.com/aslafy-z/helm-git --version 1.3.0

```

> 💡 Nếu plugin đã được cài trước đó, bạn có thể thêm --force để cài lại:
> 

```bash

helm plugin install https://github.com/aslafy-z/helm-git --version 1.3.0 --force

```

---

## 🗂️ Bước 2: Thêm Helm Repository Từ Git

Trước khi thêm repo mới, bạn nên xóa repo cũ nếu cần để tránh xung đột:

```bash

helm repo remove cert-manager || true

```

Thêm repository Helm từ Git bằng cú pháp:

```bash

helm repo add cert-manager git+https://github.com/jetstack/cert-manager@deploy/charts?ref=v0.6.2

```

### ✅ Giải Thích Tham Số:

- `@deploy/charts`: Chỉ định **đường dẫn đến thư mục chart con** trong repo Git.
- `ref=v0.6.2`: Chỉ định phiên bản cụ thể — có thể là tag, branch hoặc commit hash.

---

## 🚀 Bước 3: Cài Đặt hoặc Cập Nhật Helm Chart

Giờ bạn có thể triển khai chart từ repo Git bằng lệnh:

```bash

helm upgrade --install cert-manager cert-manager/cert-manager \
  -f prod-values.yaml \
  -n cert-manager \
  --create-namespace

```

### 🔍 Giải Thích Tham Số:

- `upgrade --install`: Cài đặt nếu chưa có, hoặc nâng cấp nếu đã tồn tại.
- `f prod-values.yaml`: Sử dụng file cấu hình cho môi trường production.
- `n cert-manager`: Chạy trong namespace `cert-manager`.
- `-create-namespace`: Tạo namespace nếu chưa tồn tại.

---

## ✅ Kết Luận

Plugin `helm-git` mở rộng khả năng triển khai Helm chart từ Git, giúp bạn dễ dàng kiểm soát phiên bản, đặc biệt khi cần thử nghiệm các phiên bản cụ thể hoặc chưa được publish lên Helm repo. Việc sử dụng tham số `ref` và chỉ định đường dẫn chart con (`@...`) giúp bạn triển khai chính xác phiên bản mong muốn một cách an toàn và linh hoạt.