---
title: Development environment with Docker and Nginx
author: phongthien99
date: 2024-30-23 23:40:00 +0700
categories: [Improve-Skill]
tags: [nginx]
math: true
media_subpath: '/posts/20180809'
---
#  Development environment with Docker and Nginx

## Đặt Vấn Đề

Việc phát triển cục bộ với Docker đã trở thành một phương pháp phổ biến trong quy trình phát triển phần mềm hiện đại. Việc khởi tạo container và ánh xạ các cổng là bước cơ bản để bắt đầu công việc. Tuy nhiên, khi làm việc với nhiều dự án đồng thời, việc xung đột cổng giữa các container trên cùng một máy chủ Docker có thể gây khó khăn. Điều này làm giảm hiệu quả công việc và khiến bạn khó quản lý các cổng đang sử dụng.

## Giải Pháp

Một giải pháp đơn giản là sử dụng reverse proxy cục bộ để quản lý các dự án. Nginx là một lựa chọn lý tưởng để định tuyến và quản lý các dự án qua các tên miền, giúp giảm thiểu xung đột cổng và tăng tính linh hoạt khi phát triển.

## Cách Thực Hiện

### Bước 1: Tạo Mạng Docker

Trước khi triển khai reverse proxy, bạn cần tạo một mạng Docker để các dịch vụ có thể giao tiếp với nhau.

```bash
docker network create traefik-ingress

```

### Bước 2: Cài Đặt Nginx

Chúng ta sẽ sử dụng Docker Compose để cài đặt Nginx cùng với các dịch vụ thử nghiệm. Dưới đây là cấu hình `docker-compose.yml` để cài đặt Nginx và các dịch vụ mẫu:

```yaml
services:
  nginx:
    image: docker.io/bitnami/nginx:1.23
    ports:
      - '8085:5001'  # Ánh xạ cổng ngoài vào cổng dịch vụ bên trong
    volumes:
      - ./nginx.conf:/opt/bitnami/nginx/conf/server_blocks/my_server_block.conf  # Cấu hình Nginx tùy chỉnh
    networks:
      - traefik-ingress

  origin-01:
    container_name: origin-01
    image: traefik/whoami
    command:
      - --port=8080
      - --name=origin-01
    networks:
      - traefik-ingress

  origin-02:
    container_name: origin-02
    image: traefik/whoami
    command:
      - --port=8080
      - --name=origin-02
    networks:
      - traefik-ingress

networks:
  traefik-ingress:
    external: true

```

### Bước 3: Cấu Hình Nginx

Cấu hình Nginx cần định tuyến lưu lượng truy cập dựa trên tên miền. Dưới đây là cấu hình Nginx giúp chuyển tiếp yêu cầu tới các container tương ứng.

```

server {
    listen 5001;
    listen [::]:5001;

    location / {
        # Sử dụng DNS resolver của Docker
        resolver 127.0.0.11;

        # Định tuyến theo tên miền
        if ($host ~* ^([a-zA-Z0-9\-]+)-([0-9]+)\.lh$) {
            proxy_pass http://$1:$2;  
            break;
        }

        # Trả về lỗi nếu tên miền không hợp lệ
        return 404 "Domain not found";
    }
}
```

Trong cấu hình Nginx này:

- Nginx sẽ lắng nghe trên cổng `5001` và chuyển tiếp yêu cầu tới các dịch vụ như `origin-01` hoặc `origin-02` dựa trên tên miền phụ (ví dụ: `origin-01.lh`).
- DNS resolver `127.0.0.11` là DNS nội bộ của Docker, giúp Nginx có thể nhận diện và định tuyến các yêu cầu tới các container.

### Bước 4: Khởi Chạy Dịch Vụ

Sau khi đã cấu hình `docker-compose.yml` và `nginx.conf`, bạn có thể khởi chạy dịch vụ bằng lệnh:

```bash

docker-compose up -d
```

Lệnh này sẽ khởi động Nginx như một reverse proxy, cùng với các dịch vụ `origin-01` và `origin-02`, mỗi dịch vụ chạy trên cổng riêng nhưng đều được truy cập qua Nginx.

### Bước 5: Kiểm Tra Thực Tế

Để kiểm tra, bạn cần thêm tên miền giả vào tệp `/etc/hosts`:

```bash

127.0.0.1 origin-01-8080.lh
```

Sau đó, bạn có thể thử truy cập dịch vụ bằng lệnh `curl`:

```bash

curl http://origin-01-8080.lh:8085/test
```

### Kết Luận

Việc sử dụng Nginx làm reverse proxy cục bộ giúp bạn dễ dàng quản lý các container Docker mà không gặp phải xung đột cổng. Với phương pháp định tuyến theo tên miền, bạn có thể truy cập nhiều dịch vụ khác nhau thông qua một cổng  duy nhất. Giải pháp này đơn giản hóa môi trường phát triển, đặc biệt khi làm việc với nhiều dự án cần sự tách biệt nhưng lại chia sẻ cùng một máy chủ Docker.