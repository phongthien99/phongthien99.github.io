---
title: "Streamlining Deployment with GitLab CI/CD"
date: 2025-06-21T01:00:00Z
draft: false
tags: ["GitLab", "CI/CD", "DevOps", "Deployment", "Automation"]
categories: ["DevOps", "Continuous Integration", "Continuous Deployment"]
author: "Phong Thien"
---

# Streamlining Deployment with GitLab CI/CD

## 🧩 1. Đặt Vấn Đề

Trong quá trình phát triển phần mềm, việc kiểm thử và debug nhanh các job CI/CD là rất quan trọng. Tuy nhiên, GitLab CI/CD thường gặp một số vấn đề khi test local bằng `gitlab-runner exec`:

- ❌ `gitlab-runner exec` **không hỗ trợ `include:` hoặc `extends:` từ file ngoài**
- ❌ Pipeline thực tế quá phức tạp, chứa nhiều môi trường như `dev`, `prod`, `staging`
- ❌ Không dễ test nhanh từng job do phải khai báo phân tán, phụ thuộc nhiều nơi

---

## 💡 2. Giải Pháp

| Vấn đề                        | Giải pháp                                      |
| ----------------------------- | ---------------------------------------------- |
| Không test local được         | Gom job + biến vào **1 file duy nhất** để test |
| Quản lý nhiều môi trường      | Mỗi môi trường tách riêng `vars-*.yml`         |
| Tái sử dụng script            | Dùng `.job-base` làm template job              |
| Giữ file `.gitlab-ci.yml` gọn | Chỉ `include` file tổng `.ci/include.yml`      |

---

## 🛠️ 3. Thực Hiện

### 📁 3.1. Cấu trúc thư mục

```

.gitlab-ci.yml

.ci/
  ├─ job-template.yml
  ├─ vars-local.yml
  ├─ vars-dev.yml
  ├─ vars-prod.yml
  ├─ workflow-dev.yml
  ├─ workflow-prod.yml
  ├─ include.yml


```

> 📝 Ghi chú: Chúng ta tách cấu hình CI/CD thành các thành phần nhỏ gọn và rõ ràng:
>
> - Template dùng chung
> - Biến theo từng môi trường
> - Workflow cụ thể cho từng môi trường
> - File gom tổng cho chạy thực tế & chạy local

---

### 🧱 3.2. Định nghĩa job template `.ci/job-template.yml`

```yaml
.job-base:
  stage: test
  tags:
    - dind
  script:
    - echo "ENV=$ENV"
    - echo "API_URL=$API_URL"
  variables:
    ENV: "local"
    API_URL: "http://localhost:3000"
```

> 📌 Ghi chú: Đây là template chính, định nghĩa các bước và biến mặc định.
>
> Mọi job thực tế sẽ `extends` từ `.job-base`.

---

### ⚙️ 3.3. Biến môi trường

### `.ci/vars-dev.yml`

```yaml
.variables-dev:
  variables:
    ENV: "dev"
    API_URL: "https://api.dev.example.com"
```

### `.ci/vars-local.yml`

```yaml
.variables-local:
  variables:
    ENV: "local"
    API_URL: "http://localhost:3000"
```

### `.ci/vars-prod.yml`

```yaml
.variables-prod:
  variables:
    ENV: "prod"
    API_URL: "https://api.prod.example.com"
```

> 📌 Ghi chú: Tách biến theo từng môi trường giúp:
>
> - Không bị nhầm biến khi deploy
> - Dễ test từng môi trường độc lập
> - Linh hoạt mở rộng về sau (thêm `qa`, `uat`, v.v.)

---

### 🔁 3.4. Định nghĩa các job

### `.ci/workflow-dev.yml`

```yaml
job-dev:
  extends: [.job-base, .variables-dev]
```

### `.ci/workflow-prod.yml`

```yaml
job-prod:
  extends: [.job-base, .variables-prod]
```

> 📌 Ghi chú: Mỗi job được build bằng cách kế thừa job-base và biến riêng môi trường.
>
> Giúp cấu hình đơn giản, không lặp lại.

---

### 📦 3.5. File `include.yml`

```yaml
include:
  - local: ".ci/job-template.yml"
  - local: ".ci/vars-dev.yml"
  - local: ".ci/vars-prod.yml"
  - local: ".ci/workflow-dev.yml"
  - local: ".ci/workflow-prod.yml"
```

> 📌 Ghi chú: Gom tất cả file cần thiết cho pipeline GitLab CI chạy thực tế trên server.
>
> File `.gitlab-ci.yml` chỉ cần `include` file này là đủ.

---

---

### 🧪 3.7. Chạy test local

```bash

gitlab-runner exec shell .job-base --cicd-config-file .ci/job-template.yml

```

> 🛠️ Ghi chú:
>
> Lệnh này sẽ chạy job `.job-base` bằng cấu hình trong file `.ci/job-template.yml`.
>
> Có thể chỉnh sửa nhanh script, biến, v.v. để test mà không ảnh hưởng pipeline thật.

---

## ✅ 4. Kết Luận

### ✅ **Ưu điểm đạt được**

- **✔️ Dễ test local**:
  → Test được từng job thông qua giá trị của biến môi trường (variable).
- **✔️ Quản lý môi trường rõ ràng**:
  → Mỗi môi trường (`dev`, `staging`, `prod`,...) có file biến riêng (`vars-*.yml`).
- **✔️ Dễ bảo trì & mở rộng**:
  → Dùng `job-base` như template để tránh lặp lại, dễ mở rộng khi thêm job mới.
- **✔️ CI gọn nhẹ**:
  → File `.gitlab-ci.yml` chính chỉ cần `include` một file tổng hợp.

---

### 🔧 **Nguyên lý kỹ thuật phần mềm & Ứng dụng trong GitLab CI/CD**

- **Function/Module Reuse (Tái sử dụng hàm/module)**:
  → `job-base` hoạt động như một **hàm dùng lại**, các job chỉ cần `extends` để kế thừa cấu trúc.
- **Encapsulation (Đóng gói)**:
  → Biến môi trường được **đóng gói** theo từng môi trường riêng biệt (`vars-dev.yml`, `vars-prod.yml`), giúp tránh rò rỉ hoặc ghi đè sai lệch.
- **Composition over Inheritance (Thành phần hơn kế thừa)**:
  → Kết hợp `job-base` + file biến (`.variables-*`) cho phép tái sử dụng linh hoạt hơn là copy/paste hoặc kế thừa toàn bộ job.
- **Separation of Concerns (Phân tách trách nhiệm)**:
  → Tách biệt từng thành phần CI/CD:
  - Template job (`job-template.yml`)
  - Biến môi trường (`vars-*.yml`)
  - Luồng chính (`.gitlab-ci.yml`)
    → Mỗi file chỉ đảm nhận **một vai trò duy nhất**, dễ đọc và dễ debug.
- **Unit Test / Test độc lập hàm**:
  → Có thể test từng job độc lập trong `job-template.yml` như cách viết và test unit test trong lập trình.
