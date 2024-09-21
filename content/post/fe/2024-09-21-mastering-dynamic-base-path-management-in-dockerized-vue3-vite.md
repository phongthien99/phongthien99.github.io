---
title: Mastering Dynamic Base Path Management in Dockerized Vue 3 + Vite
author: phongthien99
date: 2024-09-21 12:40:00 +0800
categories: [Fe]
tags: [nginx,fe,vue3,vite]
math: true
media_subpath: '/posts/20240503'
---
# Mastering Dynamic Base Path Management in Dockerized Vue 3 + Vite

## Đặt vấn đề

Trong phát triển ứng dụng web hiện đại, việc thiết lập đường dẫn build động (dynamic build path) là một yếu tố quan trọng để đảm bảo tính linh hoạt và khả năng mở rộng của ứng dụng. Đối với các ứng dụng Vue 3 sử dụng Vite, việc quản lý đường dẫn build không chỉ ảnh hưởng đến việc truy cập tài nguyên mà còn quyết định cách mà ứng dụng tương tác với các dịch vụ khác, như API.

Một trong những thách thức lớn là việc thay đổi base path của ứng dụng trong các môi trường khác nhau mà không cần phải rebuild lại toàn bộ. Khi ứng dụng được triển khai trong các môi trường như development, staging hoặc production, yêu cầu về base path có thể khác nhau, và việc không có cơ chế để điều chỉnh chúng một cách linh hoạt có thể dẫn đến:

- **Khó khăn trong triển khai**: Cần phải duy trì nhiều cấu hình hoặc bản build riêng cho từng môi trường, gây tốn thời gian và công sức.

### Giải Pháp

Để giải quyết vấn đề dynamic base path trong Vue 3 + Vite, chúng ta kết hợp ba kỹ thuật chính:

### 1. **Dynamic Environment Variables**

- Sử dụng file như `/env.js` để chứa các biến môi trường (ví dụ: `BASE_PATH`, `API_ENDPOINT`).
- Tải các biến này vào ứng dụng tại runtime thông qua đoạn script trong `index.html`.

### 2. **Nginx Configuration Generation**

- Sử dụng script hoặc template engine để tự động sinh file cấu hình nginx dựa trên các biến môi trường hiện tại.
- Áp dụng các thay đổi base path động trong cấu hình nginx để tương thích với các môi trường khác nhau.

### 3. **Tạo Template Cho `index.html`**

- Sinh `index.html` cuối cùng dựa trên template và các biến môi trường tại runtime.

Việc kết hợp các kỹ thuật này giúp thay đổi base path linh hoạt mà không cần build lại ứng dụng, đảm bảo khả năng thích ứng và tối ưu hóa quá trình triển khai.

## Thực hiện

Thực hiện trên project [github.com/wei-zone/vue3-quick-start.git](http://github.com/wei-zone/vue3-quick-start.git)

### Bước 1: Chỉnh sửa file `index.html`

- Thêm đoạn script để tải `env.js` từ `BASE_PATH` vào trong thẻ `<head>` của file `index.html`.

```html

<!DOCTYPE html>
<html lang="en">
  <head>
    <!-- Thêm đoạn này vào -->
    <script src="${BASE_PATH}/env.js"></script>
  </head>
  <body>
    <div id="app"></div>
  </body>
</html>

```

### Bước 2: Tạo file `env.template.js` trong thư mục `public`

- Tạo file `env.template.js` để định nghĩa biến `BASE_PATH` và các biến môi trường khác nếu cần.

```jsx

(function(window) {
    window.env = {
        BASE_PATH: '${BASE_PATH}' // Biến môi trường sẽ được thay thế khi build
    };
})(this);

```

### Bước 3: Cài đặt và cấu hình `vite-plugin-dynamic-base`

- Cài đặt thư viện `vite-plugin-dynamic-base` vào `devDependencies`.

```bash

pnpm add vite-plugin-dynamic-base --save-dev

```

- Cấu hình plugin `vite-plugin-dynamic-base` trong file `vite.config.ts`:

```tsx

import { defineConfig } from 'vite';
import vue from '@vitejs/plugin-vue';
import { dynamicBase } from 'vite-plugin-dynamic-base';

export default defineConfig({
  plugins: [
    vue(),
    dynamicBase({
      publicPath: 'window.env.BASE_PATH', // Sử dụng biến BASE_PATH từ env.js
      transformIndexHtml: true, // Biến đổi file index.html để sử dụng BASE_PATH
    }),
  ],
});

```

- Thực hiện build dự án:

```bash
bash
Copy code
pnpm build

```

### Bước 4: Tạo file `nginx.template.conf`

- Tạo file `nginx.template.conf` để cấu hình Nginx với `BASE_PATH` động.

```
nginx
Copy code
user  nginx;
worker_processes  1;
error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events {
  worker_connections  1024;
}

http {
  include       /etc/nginx/mime.types;
  default_type  application/octet-stream;
  log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';
  access_log  /var/log/nginx/access.log  main;
  sendfile        on;
  keepalive_timeout  65;

  server {
    listen       80;
    server_name  localhost;

    location ${BASE_PATH} {  # Sử dụng BASE_PATH được thay thế khi chạy script
      alias   /app/;  # Thư mục chứa file build của ứng dụng Vue
      index  index.html;
      try_files $uri $uri/ /index.html;  # Hỗ trợ cho SPA
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
      root   /usr/share/nginx/html;
    }
  }
}

```

### Bước 5: Tạo script `gen-template.sh` để sinh file từ các template

- Tạo script `gen-template.sh` để thay thế các biến môi trường trong file template.

```bash

#!/bin/bash

# Thay thế biến trong env.template.js và tạo file env.js
envsubst < /app/env.template.js > /app/env.js

# Thay thế biến trong index.html và ghi đè lại file
envsubst < /app/index.html | tee /app/index.html > /dev/null

# Thay thế biến trong nginx.template.conf và tạo file nginx.conf
envsubst < /nginx.template.conf > /etc/nginx/nginx.conf

```

### Bước 6: Cấu hình `docker-compose.yml` để chạy thử

- Cấu hình file `docker-compose.yml` để chạy Nginx với `BASE_PATH` đã được thay thế.

```yaml

version: '3.8'

services:
  nginx:
    image: nginx:latest
    ports:
      - "8086:80" # Mở cổng 8086 để truy cập Nginx từ bên ngoài
    command: ["/bin/sh", "-c", "/init.sh && nginx-debug -g 'daemon off;'"]
    environment:
      - BASE_PATH=/test-app # Biến môi trường sẽ được sử dụng trong script gen-template.sh
    volumes:
      - ./nginx.template.conf:/nginx.template.conf # Ánh xạ file nginx.template.conf vào container
      - ./dist:/app # Thư mục chứa file build của ứng dụng Vue
      - ./gen-template.sh:/init.sh # Ánh xạ file script init.sh vào container

```

## Kết luận

Thiết lập dynamic build path cho Vue 3 với Vite giúp điều chỉnh cấu hình ứng dụng linh hoạt mà không cần rebuild. Giải pháp này tối ưu hóa triển khai, giảm thiểu sai sót và dễ dàng thích ứng với thay đổi môi trường.

## Tài liệu tham khảo

[Read Dynamic Environment Variables in Angular](https://medium.com/@sushil.singh56/read-dynamic-environment-variables-in-angular-621e3ba38eb4)