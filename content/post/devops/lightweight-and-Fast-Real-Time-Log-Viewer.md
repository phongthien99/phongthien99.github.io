---
title: Lightweight and Fast Real-Time Log Viewer
author: phongthien99
date: 2025-04-27 12:40:00 +0800
categories: [Devops]
tags: [monitor]
math: true
media_subpath: '/posts/20240503'
---
# Lightweight and Fast Real-Time Log Viewer

## Đặt Vấn Đề

Trong quá trình vận hành hệ thống, việc nhanh chóng **tra cứu log** để xác định lỗi hoặc xác minh các sự kiện là cực kỳ quan trọng

Thông thường, chúng ta hay sử dụng các lệnh như:

```bash
tail -f /var/log/server.log | grep -E "ERROR|WARN"
grep -i "timeout" /var/log/server.log
grep -E "ERROR|WARN" /var/log/server.log

```

Tuy nhiên, cách làm này tồn tại nhiều hạn chế:

- Phải SSH vào server.
- Phải thao tác thủ công, mất thời gian.
- Khó theo dõi realtime với nhiều server hoặc nhiều file log.
- Hạn chế khả năng lọc, tìm kiếm nâng cao hoặc lưu trữ kết quả.

## Giải Pháp

Xây dựng một hệ thống **xem log trực tiếp từ trình duyệt**, hỗ trợ:

- **Tìm kiếm từ khóa** nhanh chóng.
- **Lọc log nâng cao**.
- **Xem realtime** trên nhiều file log.

## Công Nghệ Sử Dụng

- **Promtail**: Thu thập log từ file hệ thống.
- **VictoriaLogs**: Lưu trữ và phục vụ truy vấn log với hiệu suất cao.

---

# Hướng Dẫn Triển Khai Chi Tiết

Hệ thống triển khai trên môi trường Ubuntu để theo dõi log của Sigma Media Server, với các file log có pattern: `/var/log/sigma-machine/now.*`

## 1. Cài Đặt và Cấu Hình VictoriaLogs

### Bước 1: Tải và Cài Đặt VictoriaLogs

```bash
wget https://github.com/VictoriaMetrics/VictoriaMetrics/releases/download/v1.21.0-victorialogs/victoria-logs-linux-amd64-v1.21.0-victorialogs.tar.gz
tar -xzf victoria-logs-linux-amd64-v1.21.0-victorialogs.tar.gz
chmod +x victoria-logs-prod
sudo mv victoria-logs-prod /usr/local/bin/victoria-logs
sudo mkdir /var/lib/victorialogs-data

```

### Bước 2: Tạo service cho VictoriaLogs

Tạo file `/etc/systemd/system/victorialogs.service` với nội dung:

```
[Unit]
Description=VictoriaLogs Service
After=network.target

[Service]
User=root
Group=root
ExecStart=/usr/local/bin/victoria-logs --storageDataPath=/var/lib/victorialogs-data
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target

```

### Bước 3: Khởi Động VictoriaLogs

```bash
sudo systemctl daemon-reload
sudo systemctl enable victorialogs
sudo systemctl start victorialogs

```

> VictoriaLogs sẽ lắng nghe trên port 9428, lưu dữ liệu tại **/var/lib/victorialogs-data.**
> 

---

## 2. Cài Đặt và Cấu Hình Promtail

### Bước 1: Tải và Cài Đặt Promtail

```bash
wget https://github.com/grafana/loki/releases/latest/download/promtail-linux-amd64.zip
unzip promtail-linux-amd64.zip
chmod +x promtail-linux-amd64
sudo mv promtail-linux-amd64 /usr/local/bin/promtail

```

### Bước 2: Tạo File Cấu Hình Promtail

Tạo thư mục và file cấu hình:

```bash
mkdir -p /etc/promtail
touch /etc/promtail/promtal-config.yaml

```

Nội dung `/etc/promtail/promtail-config.yaml`:

```yaml

server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://localhost:9428/insert/loki/api/v1/push

scrape_configs:
  - job_name: sigma_media_server_logs
    static_configs:
      - targets:
          - localhost
        labels:
          job: sigma
          __path__: /var/log/sigma-machine/now.*

```

### Bước 3: Tạo service cho Promtail

Tạo file `/etc/systemd/system/promtail.service` với nội dung:

```

[Unit]
Description=Promtail Service
After=network.target

[Service]
User=root
Group=root
ExecStart=/usr/local/bin/promtail -config.file=/etc/promtail/promtail-config.yaml
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target

```

### Bước 4: Khởi Động Promtail

```bash

sudo systemctl daemon-reload
sudo systemctl enable promtail
sudo systemctl start promtail

```

---

# Xem và Truy Vấn Log Qua Trình Duyệt

Giờ đây, bạn có thể mở trình duyệt và truy cập địa chỉ:

```

http://localhost:9428
```

Một số ví dụ truy vấn log:

- **Tìm kiếm log với từ khóa:**

```

{filename="/var/log/sigma-machine/now.sys"} |~ "(?i)dataDir"
```

---

**Giải thích:**

- `|~` dùng để **lọc theo regex**.
- `(?i)` trong regex là **bật chế độ ignore case** (không phân biệt hoa thường).
- `"dataDir"` là từ khóa bạn tìm.

# Kết Luận

Với giải pháp này, bạn có thể:

- **Truy cập log mọi lúc, mọi nơi** mà không cần SSH vào server.
- **Tìm kiếm và lọc log** nhanh chóng.
- **Theo dõi realtime** nhiều file log từ  server.
- **Dễ dàng mở rộng** và tích hợp với các hệ thống phân tích log hoặc cảnh báo chuyên sâu trong tương lai.