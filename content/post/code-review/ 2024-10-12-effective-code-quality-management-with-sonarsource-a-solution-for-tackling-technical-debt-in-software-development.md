---
title: "Effective Code Quality Management with SonarSource: A Solution for Tackling Technical Debt in Software Development"
author: phongthien99
date: 2024-10-12 01:17:00 +0800
categories: [review-code]
tags: [SonarSource]
math: true
media_subpath: '/posts/20180809'
---
# Effective Code Quality Management with SonarSource: A Solution for Tackling Technical Debt in Software Development

## Đặt vấn đề

Trong quá trình phát triển phần mềm, có những giai đoạn dự án chạy với tốc độ "như ăn cướp", khi các nhà phát triển buộc phải tập trung giải quyết các vấn đề trước mắt mà không có thời gian tuân thủ các tiêu chuẩn chất lượng. Điều này dẫn đến mã nguồn có thể trở nên lộn xộn, thiếu cấu trúc và khó bảo trì – tạo thành một "đống rác" chứa các đoạn mã lỗi, không nhất quán, và tiềm ẩn nhiều rủi ro bảo mật.

Những giải pháp tạm thời này có thể giúp dự án tiến triển trong ngắn hạn, nhưng về lâu dài sẽ làm tăng nợ kỹ thuật, khiến mã trở nên cồng kềnh và khó bảo trì. Chính vì vậy, cần một công cụ để phân tích và quản lý chất lượng mã, giúp phát hiện và xử lý các vấn đề này kịp thời, nhằm duy trì tính ổn định và bền vững cho dự án.

## Giải pháp

SonarSource là một công cụ phân tích mã nguồn mạnh mẽ, giúp đội ngũ phát triển kiểm soát chất lượng mã một cách toàn diện. Nó không chỉ phát hiện "đống rác" ẩn trong mã nguồn mà còn cung cấp các giải pháp cụ thể để dọn dẹp một cách hệ thống. Sử dụng SonarSource giúp giảm thiểu nợ kỹ thuật, duy trì hiệu suất mã ổn định, và giảm thiểu rủi ro bảo mật, ngay cả khi dự án phải phát triển nhanh chóng trong những thời điểm gấp gáp.

## Thực hiện

Dưới đây là các bước để triển khai SonarSource thực nghiệm thông qua Docker Compose:

### Bước 1: Tạo file Docker Compose

File Docker Compose cấu hình các dịch vụ chính như sau:

```yaml
yaml
Copy code
services:
  sonarqube:
    image: sonarqube:community
    read_only: true
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_logs:/opt/sonarqube/logs
      - sonarqube_temp:/opt/sonarqube/temp
    ports:
      - "9000:9000"
volumes:
  sonarqube_data:
  sonarqube_extensions:
  sonarqube_logs:
  sonarqube_temp:

```

### Bước 2: Đăng nhập vào SonarQube

- Truy cập giao diện SonarQube qua `http://localhost:9000`.
- Đăng nhập bằng tài khoản admin hoặc tài khoản cá nhân của bạn.

### Bước 3: Tạo Project Mới

- Chọn **Projects** và nhấp vào **Create Project**.
- Bạn có thể chọn tạo dự án từ một repository (GitHub, GitLab, Bitbucket) hoặc tạo thủ công.

### Bước 4: Cấu hình Project

- Nhập tên và **key** duy nhất cho project.
- Nếu sử dụng SonarCloud, bạn có thể chọn hoặc tạo tổ chức để quản lý.

### Bước 5: Tạo Token Xác Thực

- Tạo token để thực hiện quá trình quét mã.
- Lưu token này để dùng trong các bước sau.

### Bước 6: Cấu hình công cụ phân tích mã

Tạo file `sonar-project.properties` trong project của bạn:

```bash

sonar.projectKey=<<key đã tạo ở project>>

```

Sau đó, chạy lệnh quét mã bằng cách sử dụng Docker:

```bash

docker run --rm --net=host -e SONAR_HOST_URL="http://localhost:9000" -e SONAR_TOKEN="<<token đã tạo ở project>>" -e SONAR_PROJECT_KEY="key đã tạo ở project" -v "$(pwd):/usr/src" sonarsource/sonar-scanner-cli

```

## Kết luận

SonarSource là một công cụ mạnh mẽ giúp các đội phát triển phần mềm duy trì chất lượng mã nguồn ngay cả trong những giai đoạn phát triển khẩn trương. Bằng cách tích hợp SonarSource vào quy trình phát triển, chúng ta có thể phát hiện sớm các vấn đề về mã, tối ưu hóa hiệu suất và giảm thiểu rủi ro bảo mật. Kết quả là mã nguồn trở nên dễ quản lý hơn, giảm bớt nợ kỹ thuật và đảm bảo dự án có thể phát triển bền vững hơn về lâu dài.