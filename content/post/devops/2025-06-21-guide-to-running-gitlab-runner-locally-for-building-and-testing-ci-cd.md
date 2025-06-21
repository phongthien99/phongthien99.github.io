---
title: "Guide to Running GitLab Runner Locally for Building and Testing CI/CD Pipelines"
date: 2025-06-21T00:00:00Z
draft: false
tags:
  - gitlab
  - ci/cd
  - gitlab-runner
  - devops
categories:
  - DevOps
  - CI/CD
author: Phong Thien
---

## Guide to Running GitLab Runner Locally for Building and Testing CI/CD Pipelines

## **Đặt Vấn Đề**

Trong quá trình phát triển phần mềm, việc kiểm thử và triển khai ứng dụng thường xuyên là điều không thể thiếu. Tuy nhiên, việc đợi pipeline CI/CD chạy trên GitLab server có thể mất thời gian, đặc biệt khi cần thử nghiệm nhanh các thay đổi nhỏ trong cấu hình hoặc script. Điều này làm giảm hiệu quả làm việc và gây khó khăn trong việc debug.

## **Giải Pháp**

Chạy **GitLab Runner** ngay trên máy local là một giải pháp hiệu quả. Điều này cho phép bạn kiểm thử các job CI/CD một cách nhanh chóng mà không cần push code lên GitLab, từ đó tiết kiệm thời gian và giảm thiểu rủi ro phát sinh khi deploy lên môi trường thật.

## **Thực Hiện**

### **Bước 1: Cài Đặt GitLab Runner**

Đối với hệ điều hành Ubuntu, bạn có thể cài đặt GitLab Runner bằng cách chạy:

```bash

curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
sudo apt-get install gitlab-runner=15.11.1

```

> ⚠️ Lưu ý: Bạn có thể chọn phiên bản mới nhất hoặc giữ cố định một phiên bản ổn định tùy theo yêu cầu dự án.

---

### **Bước 2: Viết File `gitlab-ci.yaml`**

Đây là file định nghĩa pipeline. Dưới đây là ví dụ một job đơn giản dùng để build Docker image:

```yaml
image: docker:dind
stages:
  - build

build-image:
  stage: build
  tags:
    - shel
  script:
    - echo "Build image"
```

---

---

### **Bước 4: Chạy GitLab Runner Local**

Khi đã có đầy đủ file cấu hình, bạn chỉ cần chạy lệnh sau để thực thi pipeline ngay trên máy:

```bash

gitlab-runner exec docker build-image
```

> Lệnh này sẽ chạy job build-image bằng executor docker – tức là trực tiếp trên môi trường máy bạn, không cần đăng ký runner hay kết nối với GitLab server.

---

## **Kết Luận**

Việc chạy GitLab Runner cục bộ là một công cụ cực kỳ hữu ích trong quá trình phát triển phần mềm hiện đại. Nó giúp bạn kiểm thử nhanh pipeline, tiết kiệm thời gian debug, đảm bảo cấu hình chính xác trước khi đưa code lên môi trường CI/CD chính thức. Hãy tận dụng tính năng này để tối ưu hóa quy trình phát triển và nâng cao chất lượng phần mềm của bạn.
