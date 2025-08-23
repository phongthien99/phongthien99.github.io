---
title: "Golang Module Design: Simplifying Struct and Interface Reuse"
date: 2025-08-24 01:17:00 +0800
draft: false
tags: ["golang", "module-design", "principle", "architecture"]
categories: ["Principle", "Programming"]
author: "phongthien99"
---

### 1. Đặt vấn đề

Trong các dự án Golang lớn, thường gặp tình huống:

- Module mới cần sử dụng một **struct** hoặc **interface** đã tồn tại trong module cũ.
- Nếu import trực tiếp module cũ: module mới sẽ phụ thuộc mạnh vào module cũ → **tight coupling**, khó bảo trì.
- Nếu di chuyển struct/interface vào **shared module**: phải sửa code cũ → dễ gây rủi ro phá vỡ logic đã ổn định.

Câu hỏi đặt ra:

> Làm sao để module mới dùng lại struct và interface của module cũ thông qua một shared module, mà không thay đổi code cũ, đồng thời giảm phức tạp và dễ maintain?
> 

---

### 2. Giải pháp: Type Alias trong Shared Module

Golang hỗ trợ **type alias**, cho phép tạo **bí danh** cho struct hoặc interface từ module cũ. Kết hợp với shared module, ta có thể giải quyết vấn đề:

```go
// shared/types.go
package shared

import oldmodule "project/module_old"

// Alias cho struct
type User = oldmodule.User

// Alias cho interface
type UserRepository = oldmodule.UserRepository

```

- Module mới chỉ cần import `shared`:

```go
import "project/shared"

func UseSharedTypes(repo shared.UserRepository) {
    var u shared.User
    u.Name = "Alice"
    repo.Save(u)  // Gọi phương thức qua interface alias
}

```

- **Lợi ích:**
    - Module mới **không cần biết module cũ**.
    - Code cũ **không bị ảnh hưởng**.
    - Nếu struct/interface gốc thay đổi tên hoặc di chuyển, chỉ cần cập nhật alias trong shared module.

**Có thể mở rộng bằng wrapper function:**

```go
func NewUser(name string) shared.User {
    return shared.User{Name: name}
}

```

- Giúp module mới thêm logic khởi tạo object mà không sửa module cũ.

---

### 3. Giản hóa sự phức tạp

Sử dụng alias cho **struct và interface** giúp:

1. **Giảm phụ thuộc trực tiếp** giữa module mới và module cũ → tránh tight coupling.
2. **Dễ dàng refactor hoặc mở rộng** → thay đổi struct/interface gốc chỉ cần sửa alias trong shared module.
3. **Chuẩn hóa truy cập các type chung** → developer dễ nhìn, maintain hơn.
4. **Giảm code lặp lại**, tránh copy/paste struct hoặc interface.

Nhờ đó, module mới có thể dùng lại cả struct lẫn interface cũ một cách **an toàn, rõ ràng và linh hoạt**.

---

### 4. Tuân thủ và vi phạm nguyên tắc

**Tuân thủ:**

- **DRY (Don't Repeat Yourself):** Tránh copy struct/interface, tái sử dụng thông qua alias.
- **Encapsulation & Separation of Concerns:** Module mới không cần biết chi tiết module cũ.
- **Maintainability:** Thay đổi struct/interface gốc không phá vỡ module mới.

**Có thể vi phạm:**

- **Leak abstraction:** Alias quá nhiều hoặc không rõ ràng → dễ nhầm lẫn giữa alias và struct/interface gốc.
- **Single Responsibility Principle (SRP):** Shared module chứa quá nhiều alias → module trở nên “quá tải”.
- **Overuse:** Lạm dụng alias cho mọi struct/interface → giảm readability, khó trace code.

---

### 5. Kết luận

Sử dụng **type alias trong shared module cho cả struct và interface** là một giải pháp hiệu quả để:

- Cho phép module mới **sử dụng struct và interface của module cũ** mà không phá vỡ code hiện tại.
- Giảm sự **phức tạp trong quản lý module**, chuẩn hóa điểm truy cập các type chung.
- Duy trì **tính maintainable và mở rộng được dự án**, đồng thời tuân thủ nhiều nguyên tắc cơ bản của lập trình module.