---
title: Setting Up Fake Wildcard Domain with Coredns on Local Ubuntu
author: phongthien99
date: 2024-08-17 01:17:00 +0800
categories: [Improve-Skill]
tags: [dns]
math: true
media_subpath: '/posts/20180809'
---
# Setting Up Fake Wildcard Domain with Coredns on Ubuntu

## Đặt vấn đề
  Trong quá trình phát triển phần mềm, việc sử dụng nhiều tên miền giả dạng `*.lh` là khá phổ biến. Thông thường, để xử lý các tên miền này, chúng ta phải thủ công sửa file `/etc/hosts` mỗi khi cần tạo một tên miền mới. Điều này không chỉ gây mất thời gian mà đôi khi còn dẫn tới các lỗi không mong muốn nếu quên cập nhật file.

## Giải pháp

   CoreDNS cung cấp khả năng định tuyến DNS mạnh mẽ và có thể dễ dàng cấu hình để xử lý các wildcard domain.

## Thực hiện

  Triển khai thông qua Docker-compose
### Bước 1: Tạo file Docker Compose

Tạo một thư mục cho cấu hình Docker Compose:

```bash

mkdir coredns-wildcard
cd coredns-wildcard

```

Tạo một file `docker-compose.yml` với nội dung sau:

```yaml

version: '3'
services:
  coredns:
    image: coredns/coredns:latest
    container_name: coredns
    command: "-conf /Corefile"
    ports:
      - "53:53/udp"
    volumes:
      - ./Corefile:/etc/coredns/Corefile
    restart: unless-stopped

```

### Bước 2: Cấu hình CoreDNS

Tạo một file cấu hình tên là `Corefile` trong thư mục `coredns-wildcard` với nội dung sau:

```

.:53 {
    template IN ANY lh {
        match "\w*\.(lh\.)$"
        answer "{{ .Name }} 3600 IN A 127.0.0.1" 
        fallthrough
    }
    forward . 8.8.8.8
    log
}


```

Cấu hình này sẽ chuyển hướng tất cả các tên miền dạng `*.lh` tới địa chỉ `127.0.0.1` và sử dụng Google DNS cho các yêu cầu khác.

### Bước 3: Chạy Docker Compose

Trong thư mục `coredns-wildcard`, chạy lệnh sau để khởi động CoreDNS:

```bash

docker-compose up -d

```

### Bước 4: Cấu hình DNS trên máy tính cá nhân

Để sử dụng CoreDNS, chỉnh sửa file `/etc/resolv.conf` để sử dụng CoreDNS làm DNS cục bộ:

```bash

sudo nano /etc/resolv.conf

```

**Thêm dòng sau vào đầu file:**

```

nameserver 127.0.0.1

```

### Bước 6: Kiểm tra cấu hình

Kiểm tra xem cấu hình đã hoạt động hay chưa bằng cách sử dụng lệnh `nslookup`:

```bash

nslookup test.lt

```

Nếu cấu hình đúng, bạn sẽ nhận được phản hồi với địa chỉ IP `127.0.0.1`.

## Kết luận

Với các bước trên, việc thiết lập wildcard domain với CoreDNS trên Ubuntu thông qua Docker Compose trở nên đơn giản và hiệu quả. Đây là một giải pháp tốt để quản lý các tên miền giả phục vụ cho quá trình phát triển phần mềm một cách linh hoạt.