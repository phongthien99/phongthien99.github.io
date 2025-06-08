---
title: "Migration MongoDB With Flyway"
date: 2025-06-08T00:00:00Z
draft: false
tags:
  - MongoDB
  - Flyway
  - Database Migration
categories:
  - Database
  - DevOps
author: phongthien
---

## Đặt Vấn Đề

Khi làm việc với các hệ thống lớn sử dụng MongoDB, việc quản lý migration dữ liệu (tạo collection, thêm field, thay đổi schema logic...) thường không được quan tâm đúng mức. Điều này dẫn đến:

- Dữ liệu thiếu đồng nhất giữa các môi trường.
- Khó rollback hoặc kiểm soát phiên bản dữ liệu.
- Không theo kịp tốc độ phát triển CI/CD khi cần thay đổi schema thường xuyên.

## Cách Giải Quyết

Flyway gần đây đã hỗ trợ **MongoDB** thông qua các **JavaScript-based migration**. Dù MongoDB là NoSQL, ta vẫn có thể định nghĩa các thao tác migration dưới dạng `.js` file (giống shell script của MongoDB) và quản lý bằng Flyway như:

- Ghi log migration vào collection.
- Chạy tuần tự từng script.
- Cho phép rollback bằng cách viết script ngược lại.

Trong bài viết này, ta sẽ triển khai Flyway sử dụng Docker để migrate MongoDB.

## Thực Hiện

### 1. Cấu Trúc Thư Mục

```bash

├── docker-compose.yml
├── config.cfg             # Cấu hình Flyway
└── migrations/
    ├── V1__init_collection.js
    └── V2__add_index.js

```

### 2. Nội Dung `docker-compose.yml`

```yaml
version: '3.8'
services:
  flyway:
    image: redgate/flyway
    entrypoint: [""]
    command: >
      sh -c "flyway migrate && flyway info"
    volumes:
      - ./migrations:/migrations
      - ./config.cfg:/flyway/conf/flyway.conf
    # environment:
    #   - FLYWAY_NATIVE_CONNECTORS=true
    networks:
      - traefik-ingress
volumes:
  flyway_data:
networks:
  traefik-ingress:
    external: true  # Sử dụng mạng traefik đã được tạo sẵn
```

> Lưu ý: entrypoint: [""] để bỏ entrypoint mặc định trong image redgate/flyway.
> 

### 3. Cấu Hình `config.cfg`

```

flyway.url=xxx
flyway.user=xxx
flyway.password=xxx
flyway.schemas=xxx
flyway.baselineOnMigrate=true
flyway.sqlMigrationSuffixes=.js
flyway.locations=filesystem:/migrations

```

> Flyway sẽ tìm các file .js trong thư mục /migrations và chạy theo thứ tự V1__..., V2__...,...
> 

### 4. Ví Dụ Migration Script

`migrations/V1__init_collection.js`

```jsx
db.createCollection("videos");
db.videos.insertOne({
  title: "Flyway Introduction",
  uploaded_at: new Date()
});

```

`migrations/V2__add_index.js`

```jsx
db.videos.createIndex({ title: 1 }, { unique: true });

```

### 5. Chạy Migration

```bash
docker compose up

```

Flyway sẽ:

- Kết nối tới MongoDB .
- Tìm và chạy các file `.js` chưa được áp dụng.
- Log thông tin migration (tên file, thời gian, trạng thái).

## Kết Luận

Việc sử dụng **Flyway với MongoDB** là một bước tiến giúp chuẩn hóa việc migration dữ liệu NoSQL, vốn trước đây chủ yếu là thủ công. Với Flyway, bạn có thể:

✅ Kiểm soát được lịch sử migration

✅ Dễ dàng tích hợp vào CI/CD pipelines

✅ Giảm thiểu rủi ro khi deploy schema/data changes

Việc chạy Flyway qua **Docker** còn giúp môi trường dev/test dễ dàng tiếp cận và cấu hình nhất quán. Với những hệ thống cần tính ổn định và có quy trình release chặt chẽ, đây là lựa chọn nên cân nhắc.