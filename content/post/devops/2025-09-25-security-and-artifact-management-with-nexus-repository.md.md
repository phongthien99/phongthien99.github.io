---
title: "Security and Artifact Management with Nexus Repository"
date: "2025-09-25T00:21:26Z"
draft: false
tags:
  - Nexus
categories:
  - Devops
author: "phongthien"
---

# Security and Artifact Management with Nexus Repository

## 1. Đặt vấn đề

Trong phát triển phần mềm hiện đại, hầu hết dự án đều phụ thuộc vào rất nhiều **thư viện bên thứ ba**. Điều này kéo theo nhiều rủi ro:

- **Bảo mật**
    - Lấy dependency trực tiếp từ internet (Maven Central, npmjs.org, PyPI, …) dễ gặp package chứa mã độc.
    - Không có kiểm soát: bất kỳ ai cũng có thể thêm/sử dụng artifact mà không bị giám sát.
- **Quản lý artifact**
    - Các gói nội bộ (internal library, SDK) cần một nơi tập trung để chia sẻ trong team.
    - Artifact version nhiều, dễ trùng lặp hoặc thất lạc.
    - CI/CD cần nguồn tin cậy để lấy/gửi artifact, tránh “mất gói” hoặc “dùng nhầm version”.

👉 Vì vậy cần một **trung tâm artifact tập trung, an toàn, dễ quản lý**.

---

## 2. Giải pháp: Nexus Repository

**Nexus Repository Manager (Nexus 3)** là giải pháp phổ biến:

- **Bảo mật**
    - Hỗ trợ xác thực người dùng, phân quyền chi tiết.
    - Proxy external repository (Maven Central, npmjs.org, Docker Hub, …) và cache lại → ngăn gói độc hại, giảm phụ thuộc internet.
- **Quản lý artifact tập trung**
    - Lưu trữ artifact nội bộ (hosted).
    - Proxy repo bên ngoài (proxy).
    - Gom nhiều repo thành một entry duy nhất cho developer dùng (group).
    - Chính sách cleanup tự động xoá version cũ.
- **Tích hợp CI/CD**
    - Dễ push/pull artifact trong Jenkins, GitLab CI, GitHub Actions.

---

## 3. Triển khai Nexus bằng Docker

### 3.1. Cấu hình `docker-compose.yml`

```yaml
services:
  nexus:
    image: sonatype/nexus3
    container_name: nexus
    restart: always
    volumes:
      - "nexus-data:/sonatype-work"
    networks:
      default:
        aliases:
          - nexus-8081.remote
          - nexus-8085.remote

volumes:
  nexus-data: {}

networks:
  default:
    external:
      name: traefik-ingress

```

👉 Giải thích:

- **Volumes** `nexus-data` lưu toàn bộ cấu hình, artifact.
- **Network alias** giúp client gọi được Nexus qua `nexus-8081.remote`.
- Nếu dùng Traefik, có thể expose qua domain `https://nexus.company.com`.

Chạy dịch vụ:

```bash
docker compose up -d

```

Đăng nhập lần đầu: `cat nexus-data/admin.password`.

---

## 4. Kết nối Maven với Nexus (qua Proxy)

Để mọi dependency đều đi qua Nexus thay vì internet, cần chỉnh file `~/.m2/settings.xml`.

### 4.1. File cấu hình rút gọn

```xml
<settings>
  <mirrors>
    <mirror>
      <id>nexus</id>
      <mirrorOf>*</mirrorOf>
      <url>http://nexus-8081.remote:8080/repository/maven-public/</url>
    </mirror>
  </mirrors>
</settings>

```

👉 Ý nghĩa:

- `mirrorOf="*"`: tất cả repository đều được redirect qua Nexus.
- `maven-public`: là **Group repo** trong Nexus (gom Maven Central + repo nội bộ).
- Developer chỉ cần 1 URL duy nhất, không quan tâm repo ngoài.

---

### 4.2. Example thử nghiệm pull artifact

Tạo một project Maven mới từ archetype:

```bash
mvn archetype:generate \
  -DgroupId=com.demo \
  -DartifactId=nexus-test \
  -DarchetypeArtifactId=maven-archetype-quickstart \
  -DinteractiveMode=false

```

👉 Maven sẽ:

1. Gửi request dependency đến `http://nexus-8081.remote:8080/repository/maven-public/`.
2. Nexus kiểm tra cache:
    - Nếu có sẵn → trả về ngay.
    - Nếu chưa có → proxy ra Maven Central, tải artifact, cache lại, rồi trả về.

Lần sau build lại, artifact sẽ lấy trực tiếp từ cache trong Nexus, không ra internet nữa.

---

## 5. Kết luận

- Việc **bảo mật và quản lý artifact** là cực kỳ quan trọng khi hệ thống phụ thuộc nhiều package bên ngoài.
- **Nexus Repository** cung cấp giải pháp tập trung:
    - Proxy & cache external repo → an toàn, nhanh hơn.
    - Lưu trữ artifact nội bộ, dễ quản lý version.
    - Hỗ trợ phân quyền & tích hợp tốt với CI/CD.

👉 Với cấu hình `settings.xml` rút gọn, team có thể ngay lập tức chuyển toàn bộ Maven build sang dùng Nexus mà không cần thay đổi nhiều.