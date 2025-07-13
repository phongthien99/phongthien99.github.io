---
title: "Sharing JS Code Made Easy with pnpm Workspaces & Git Submodules"
date: "2025-07-13T00:00:00Z"
draft: false
tags:
  - pnpm
  - git
categories:
  - git
author: "phongthien"
---

# Sharing JS Code Made Easy with pnpm Workspaces & Git Submodules

##  🎯 Đặt Vấn Đề:

Hãy tưởng tượng bạn có nhiều dự án front-end (ví dụ: một ứng dụng web chính, một cổng admin, một landing page riêng) và nhiều dự án back-end (API Gateway, microservices). Tất cả những dự án này đều cần dùng chung một số thành phần như:

- **UI Components:** Các thành phần giao diện tái sử dụng (button, modal, form input…).
- **Utility Functions:** Các hàm tiện ích (validation, format ngày giờ, helpers gọi API…).
- **Type Definitions:** Các định nghĩa kiểu TypeScript chung (interfaces, enums…).
- **Business Logic:** Các module xử lý nghiệp vụ lõi.

Nếu không có chiến lược quản lý tốt, bạn sẽ gặp phải những vấn đề sau:

1. **Sao chép mã nguồn (Code Duplication):** Phải copy-paste code giữa nhiều dự án. Khi sửa bug hay update logic, phải sửa nhiều nơi, dễ sót hoặc gây lỗi mới.
2. **Khó khăn khi cập nhật (Difficult Updates):** Mỗi lần cần cập nhật thư viện chung, phải update thủ công từng dự án, dễ quên và dễ phá vỡ.
3. **Quản lý phiên bản phức tạp (Version Management):** Làm sao đảm bảo tất cả dự án chạy ổn định với các version phù hợp?

## 💡 Giải Pháp: Kết Hợp `pnpm workspace` và `git submodule`

  Để giải quyết triệt để, chúng ta có thể kết hợp **pnpm workspace** và **git submodule**.

### 🟢 `pnpm workspace`

  - Cho phép quản lý nhiều package (dự án con hoặc thư viện) trong cùng một repo hoặc các folder con.

### 🔵 `git submodule`

  - Cho phép nhúng một repository Git khác vào như một thư mục con, nhưng vẫn giữ commit và lịch sử riêng.
  - Bạn có thể lock từng dự án vào một commit cụ thể của shared lib.
  - Tách biệt rõ ràng trách nhiệm giữa repo chính và repo thư viện.

---

### ⚙️ Cơ Chế Hoạt Động

- Tạo repo riêng cho shared libraries, ví dụ: `shared-libs`.
- Trong mỗi dự án chính, thêm `shared-libs` làm submodule nếu cần dùng .
- Cấu hình `pnpm workspace` để nhận diện các package con bên trong `shared-libs`.
- Các dự án sử dụng thư viện bằng cách khai báo trực tiếp trong `package.json`.

---

## 🚀 Thực Hiện Chi Tiết

### Bước 1️⃣: Thêm `shared-libs` vào dự án chính bằng `git submodule`

```bash

git submodule add https://github.com/your-org/shared-libs lib/shared-libs
git submodule update --init --recursive

```

> 📁 Sau bước này, thư mục lib/shared-libs chứa toàn bộ mã nguồn thư viện dùng chung.
> 

---

### Bước 2️⃣: Cấu hình `pnpm workspace`

Tại thư mục gốc dự án chính, thêm file `pnpm-workspace.yaml` (hoặc chỉnh sửa nếu đã có):

```yaml

packages:
  - 'lib/shared-libs/packages/*'

```

> ✅ pnpm sẽ tự nhận diện các package bên trong lib/shared-libs/packages như các package nội bộ.
> 

---

### Bước 3️⃣: Khai báo dependency trong `package.json`

Mở file `package.json` của dự án chính, chỉnh `dependencies` (hoặc `devDependencies`) như sau:

```json

{
  "name": "web-app",
  "version": "1.0.0",
  "private": true,
  "dependencies": {
    "@shared/utils": "workspace:*",
    "@shared/ui-components": "workspace:*"
  }
}

```

> 🔥 Dòng "workspace:*" có nghĩa: pnpm sẽ liên kết trực tiếp package nội bộ (không dùng bản publish trên npm), tiện cho phát triển song song.
> 

---

### Bước 4️⃣: Cài đặt

Sau khi chỉnh xong `package.json`, chạy:

```bash

pnpm install

```

---

### Bước 5️⃣: Làm việc với submodule

#### Clone dự án mới

```bash

git clone <your-main-project-url>
git submodule update --init --recursive
pnpm install

```

#### Khi cần cập nhật `shared-libs`

```bash
cd lib/shared-libs
git pull origin main
cd ../..
git add lib/shared-libs
git commit -m "Update shared-libs to latest commit"

```

---

## 🛠️ Một số lệnh hữu ích với `git submodule`

| Lệnh | Mục đích |
| --- | --- |
| `git submodule` | Liệt kê submodule hiện có |
| `git submodule update --init` | Khởi tạo và đồng bộ submodule sau khi clone |
| `git submodule add <repo> <path>` | Thêm submodule mới |
| `cd lib/shared-libs && git pull` | Lấy version mới nhất của submodule |
| `git add lib/shared-libs && git commit` | Commit thay đổi submodule |






---

## 🏁 Kết Luận

---

### 💥 Lợi Ích Khi Kết Hợp

✅ **Giảm trùng lặp mã nguồn** — không còn copy-paste.

✅ **Quản lý dependency chính xác** — mỗi dự án lock commit submodule riêng, không bị vỡ.

✅ **Phát triển nhanh & debug dễ** — sửa code shared-libs, dự án chính thấy ngay.

✅ **Quản lý phiên bản rõ ràng** — submodule tách biệt, chủ động cập nhật.

Việc kết hợp **pnpm workspace** và **git submodule** mang đến một giải pháp cực kỳ mạnh mẽ và thực dụng để quản lý thư viện dùng chung giữa nhiều dự án JavaScript/TypeScript.

✅ Không chỉ giúp giảm rủi ro, tiết kiệm thời gian, mà còn tối ưu chi phí build, test và deploy.

Dù có thể cần thêm chút công sức ban đầu để thiết lập, nhưng về lâu dài đây là **giải pháp rất đáng giá** cho những hệ thống lớn, nhiều repo, nhiều nhóm cùng phát triển.