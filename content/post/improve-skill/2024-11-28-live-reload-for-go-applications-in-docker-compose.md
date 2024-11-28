---
title: Live Reload for Go Applications in Docker Compose
author: phongthien99
date: 2024-11-28 01:17:00 +0800
categories: [Improve-Skill]
tags: [portainer]
math: true
media_subpath: '/posts/20180809'
---
# Live Reload for Go Applications in Docker Compose

### **1. Vấn Đề**

Khi phát triển ứng dụng Go, việc phải dừng ứng dụng, biên dịch lại, và khởi động lại mỗi khi thay đổi mã nguồn là một quy trình tốn thời gian và làm gián đoạn luồng làm việc. Vậy làm thế nào để giảm thiểu công đoạn này và cải thiện hiệu quả phát triển? Giải pháp chính là **live reload**, cho phép ứng dụng tự động cập nhật mà không cần thao tác thủ công.

---

### **2. Giải Pháp**

Live reload giúp tự động phát hiện thay đổi trong mã nguồn, biên dịch và khởi động lại ứng dụng. Trong Go, một công cụ phổ biến hỗ trợ tính năng này là **Air**. Air đơn giản, mạnh mẽ, và dễ tích hợp, giúp lập trình viên tiết kiệm thời gian.

---

### **3. Cách Thực Hiện**

### **Bước 1: Tạo ứng dụng mẫu**

Tạo một ứng dụng Go đơn giản để thử nghiệm:

```bash

mkdir air-example && cd ./ go mod init example/air
touch main.go

```

Thêm đoạn mã sau vào file `main.go`:

```go
package main

import (
	"fmt"
	"net/http"
)

func helloHandler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello, World!")
}

func main() {
	http.HandleFunc("/", helloHandler)
	fmt.Println("Starting server at port 8080")
	if err := http.ListenAndServe(":8080", nil); err != nil {
		fmt.Println(err)
	}
}

```

---

### **Bước 2: Tích hợp Air vào Docker**

Cấu hình [Dockerfile.de](http://Dockerfile.dev)v với Air để hỗ trợ live reload:

```

# Stage 1: Base image
FROM golang:1.22.3 as base

# Stage 2: Development environment
FROM base as dev

# Cài đặt Air
RUN curl -sSfL https://raw.githubusercontent.com/cosmtrek/air/master/install.sh | sh -s -- -b $(go env GOPATH)/bin

# Lệnh mặc định để chạy Air
CMD ["air"]

```

---

### **Bước 3: Cấu hình Docker-Compose**

Kết hợp Air với Docker Compose để tự động tải lại ứng dụng:

```yaml

version: "3.9"
services:
  demo-air:
    container_name: air-demo
    build:
      context: .
      target: dev
    working_dir: /opt/app
    command: air --build.cmd "go build -o build/app main.go" --build.bin "build/app"
    volumes:
      - ./:/opt/app
    networks:
      - app-network
networks:
  app-network:
    external: true

```

**Giải thích:**

- **`command`**: Chạy Air với tham số tùy chỉnh để build và chạy ứng dụng.
- **`volumes`**: Mount mã nguồn từ máy vào container để theo dõi thay đổi.
- **`networks`**: Tạo hoặc sử dụng mạng Docker để kết nối các container.

---

### **Bước 4: Chạy Ứng Dụng**

Khởi động ứng dụng với lệnh:

```bash
docker-compose up

```

Mỗi khi thay đổi mã nguồn trong thư mục dự án, Air sẽ tự động phát hiện và tải lại ứng dụng.

---

### **4. Kết Luận**

Sử dụng Air cùng Docker-Compose giúp tối ưu hóa quá trình phát triển ứng dụng Go. Quy trình live reload giúp giảm thiểu thao tác thủ công, tăng tốc độ phát triển và mang lại trải nghiệm làm việc liền mạch.