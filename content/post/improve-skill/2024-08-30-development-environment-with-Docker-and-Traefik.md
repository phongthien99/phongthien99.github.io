---
title:  Development environment with Docker and Traefik
author: phongthien99
date: 2024-08-30 01:17:00 +0800
categories: [Improve-Skill]
tags: [traefik]
math: true
media_subpath: '/posts/20180809'
---
# Development environment with Docker and Traefik

## Đặt vấn đề

Phát triển cục bộ với Docker đã trở nên phổ biến trong các quy trình phát triển phần mềm. Thông thường, bạn chỉ cần chạy một container và ánh xạ các cổng cần thiết là có thể bắt đầu công việc. Tuy nhiên, khi làm việc với nhiều dự án đồng thời, việc xung đột cổng giữa các dự án trên cùng một máy chủ Docker có thể gây ra khó khăn trong việc quản lý. Điều này không chỉ làm giảm hiệu quả làm việc mà còn gây phiền toái khi bạn phải nhớ chính xác dự án nào sử dụng cổng nào.

## Giải pháp

Sử dụng một reverse proxy cục bộ để quản lý các dự án sẽ giúp bạn giải quyết vấn đề xung đột cổng. Traefik là một lựa chọn phù hợp để định tuyến và quản lý các dự án cục bộ thông qua việc ánh xạ miền và base path tương ứng.

## Thực hiện

### Bước 1: Khởi tạo mạng lưới (network)

    Trước khi cài đặt Traefik, bạn cần tạo một mạng lưới Docker để đảm bảo các dịch vụ của bạn có thể giao tiếp với nhau.

```bash
bashCopy code
docker network create traefik-ingress

```

### Bước 2: Cài đặt Traefik

Chúng ta sẽ cài đặt Traefik thông qua Docker Compose. File `docker-compose.yml` dưới đây sẽ cài đặt Traefik và thiết lập các cổng cần thiết.

```yaml
yamlCopy code
version: '3.7'
services:
  traefik:
    image: "traefik:v3.1"
    container_name: "traefik"
    command:
      - "--log.level=DEBUG"
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entryPoints.web.address=:80"
    ports:
      - "8080:80"    # Định tuyến web (HTTP)
      - "8443:80"    # Định tuyến cho HTTPS
      - "18080:8080" # Dashboard của Traefik
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
    networks:
      - traefik-ingress
    restart: unless-stopped
networks:
  traefik-ingress:
     external: true

```

### Bước 3: Thiết lập một dự án mới

    Sau khi Traefik đã được cài đặt, bạn có thể bắt đầu thêm các dự án của mình và cấu hình để chúng hoạt động qua Traefik.

```yaml
yamlCopy code
version: '3.8'

services:
  mam.svc:
    build:
      context: .
      dockerfile: Dockerfile.dev
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.mam-api-svc.rule=PathPrefix(`/api/library`)"
      - "traefik.http.routers.mam-api-svc.service=mam-api-svc"
      - "traefik.http.services.mam-api-svc.loadbalancer.server.port=8080"

      - "traefik.http.routers.mam-grpc-svc.rule=Host(`mam.lh`)"
      - "traefik.http.routers.mam-grpc-svc.service=mam-grpc-svc"
      - "traefik.http.services.mam-grpc-svc.loadbalancer.server.port=50051"
      - "traefik.http.services.mam-grpc-svc.loadbalancer.server.scheme=h2c"
    environment:
      - NODE_ENV=development
    volumes:
      - .:/src
    command: ["pnpm", "run", "start:dev"]
    networks:
      - traefik-ingress

networks:
  traefik-ingress:
    external: true

```

### Giải thích cấu hình:

- **traefik.enable=true**: Kích hoạt Traefik cho dịch vụ này.
- **traefik.http.routers.mam-api-svc.rule=PathPrefix(`/api/library`)**: Định tuyến các yêu cầu HTTP có đường dẫn bắt đầu bằng `/api/library` tới dịch vụ `mam-api-svc`.
- **traefik.http.routers.mam-api-svc.service=mam-api-svc**: Đặt tên dịch vụ HTTP là `mam-api-svc`.
- **traefik.http.services.mam-api-svc.loadbalancer.server.port=8080**: Định tuyến lưu lượng HTTP tới cổng 8080 của dịch vụ này.
- **traefik.http.routers.mam-grpc-svc.rule=Host(`mam.lh`)**: Định tuyến các yêu cầu GRPC tới dịch vụ `mam-grpc-svc` khi host là `mam.lh`.
- **traefik.http.services.mam-grpc-svc.loadbalancer.server.port=50051**: Định tuyến lưu lượng GRPC tới cổng 50051.
- **traefik.http.services.mam-grpc-svc.loadbalancer.server.scheme=h2c**: Sử dụng HTTP/2 không bảo mật (h2c) cho GRPC.

### Kết luận

Triển khai Traefik làm reverse proxy cục bộ giúp bạn quản lý các dự án một cách dễ dàng và tránh xung đột cổng. Với khả năng định tuyến theo miền và base path, bạn có thể dễ dàng phân biệt các dịch vụ mà không cần nhớ cổng của từng ứng dụng. Hơn nữa, Traefik hỗ trợ HTTP/2 và GRPC, mang lại sự linh hoạt và mạnh mẽ cho môi trường phát triển cục bộ, từ đó tăng hiệu suất làm việc.

## Tài liệu tham khảo

- [Traefik Documentation](https://traefik.io/traefik)
- [Development Environment with Docker and Traefik](https://dev.to/flemming/development-enviroment-with-docker-and-traefik-1lg6)