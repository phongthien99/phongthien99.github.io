---
title: Bypass login Portainer
author: phongthien99
date: 2024-08-12 01:17:00 +0800
categories: [Improve-Skill]
tags: [portainer]
math: true
media_subpath: '/posts/20180809'
---
# Bypass login Portainer

## Vấn đề gặp phải

Khi sử dụng Portainer phiên bản 2.19.4, hệ thống yêu cầu người dùng phải đăng nhập bằng tài khoản và mật khẩu. Mặc dù đây là một tính năng quan trọng nhằm tăng cường bảo mật, nhưng trong môi trường làm việc local, việc này có thể trở nên không cần thiết và gây ra sự phiền toái cho người dùng.

## Giải pháp

Để đơn giản hóa việc truy cập vào Potainer trong môi trường local mà không cần phải đăng nhập, bạn có thể:

1. **Đặt một proxy server trước Portainer và sử dụng access token để truy cập Portainer API.**
2. **Tạo một JWT token giả dài hạn để tránh việc token hết hạn quá sớm khi client kiểm tra.**

## Thực hiện

### 1. Lấy Access Token

Theo hướng dẫn của Portainer: [Portainer API Access](https://docs.portainer.io/api/access).

### 2. Cấu hình Traefik làm Proxy Server

Cấu hình Traefik để làm proxy server với file cấu hình YAML sau:

```yaml

http:
  serversTransports:
    portainer-transport:
      insecureSkipVerify: true
  routers:
    portainer-local-router:
      rule: "Host(`portainer.lh`)"   # Thay thế bằng domain hoặc quy tắc routing của bạn
      service: portainer-local
      middlewares:
        - portainerApiKeyHeaders
  middlewares:
    portainerApiKeyHeaders:
      headers:
        customRequestHeaders:
          X-API-Key: <<access token>>  # Thay thế bằng access token của bạn

  services:
    portainer-local:
      loadBalancer:
        servers:
          - url: <<address portainer>>  # Thay thế bằng địa chỉ IP của Portainer
        serversTransport: portainer-transport

```

Sau đó, ánh xạ domain giả `portainer.lh` về địa chỉ IP local.

### 3. Tạo JWT Token dài hạn

Ví dụ về một JWT token dài hạn:

```yaml

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MSwidXNlcm5hbWUiOiJhZG1pbiIsInJvbGUiOjEsInNjb3BlIjoiZGVmYXVsdCIsImZvcmNlQ2hhbmdlUGFzc3dvcmQiOmZhbHNlLCJleHAiOjM4MDYxNTA3OTIsImlhdCI6MTcyMzM5MjA3N30.49qEbdGLK39PgplVPUwsiDud_vgP1wyP19lAYic19Gs

```

Set key `portainer.JWT` và giá trị trên vào `localStorage` của trang `portainer.lh:8089`, trong đó:

- `8089` là cổng HTTP entry của Traefik.

Có thể sử dụng script sau để thiết lập token:

```jsx

// Define your JWT token
const jwtToken = '"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MSwidXNlcm5hbWUiOiJhZG1pbiIsInJvbGUiOjEsInNjb3BlIjoiZGVmYXVsdCIsImZvcmNlQ2hhbmdlUGFzc3dvcmQiOmZhbHNlLCJleHAiOjM4MDYxNTA3OTIsImlhdCI6MTcyMzM5MjA3N30.49qEbdGLK39PgplVPUwsiDud_vgP1wyP19lAYic19Gs"';

// Store the token in localStorage
localStorage.setItem('portainer.JWT', jwtToken);

// Verify by retrieving it
const storedToken = localStorage.getItem('portainer.JWT');
console.log(storedToken);  // Should output the JWT token

```

## Kết luận

Việc cấu hình Traefik làm proxy server và sử dụng JWT token dài hạn giúp đơn giản hóa việc truy cập vào Portainer trong môi trường local mà không cần phải đăng nhập liên tục. Hãy đảm bảo rằng bạn đã cấu hình chính xác và kiểm tra token để đảm bảo sự hoạt động trơn tru.

## Tài liệu

- [Traefik Routing Documentation](https://doc.traefik.io/traefik/routing/routers/)