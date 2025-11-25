---
title: Sum Type and Pattern Matching
author: phongthien99
date: 2025-11-25 00:40:00 +0800
categories: [Design-pattern]
tags: []
math: true
media_subpath: '/posts/20180809'
---

# **Sum Type and Pattern Matching**

## **1. Đặt vấn đề**

Trong lập trình truyền thống, việc biểu diễn trạng thái và luồng xử lý thường dùng:

- `if/else`
- `switch/case`
- `null` để đại diện cho “không có giá trị”
- ném exception để biểu diễn lỗi

Những cách làm này tồn tại nhiều vấn đề:

- **Null quá nguy hiểm** → dễ dẫn tới `null pointer exception`.
- **Exception khó kiểm soát** → hàm có thể throw bất kỳ lúc nào.
- **Kiểu dữ liệu không diễn đạt được trạng thái thật** → ví dụ hàm `getUser()` trả về `User | null`, rất mơ hồ.
- **`if/else` chồng chất** → logic dễ vỡ, khó mở rộng.
- **Không buộc compiler kiểm tra đầy đủ các nhánh** → dễ bỏ sót trường hợp.

Khi hệ thống lớn lên, việc quản lý các trạng thái dạng "một trong nhiều khả năng" trở nên phức tạp. Đây chính là lúc **Sum Type** và **Pattern Matching** trở thành giải pháp mạnh mẽ.

---

## **2. Giải pháp: Sum Type + Pattern Matching**

### **2.1 Sum Type là gì?**

**Sum Type** (còn gọi là *Tagged Union*, *Discriminated Union*, *Variant Types*, *Union Types an toàn*) là kiểu dữ liệu mô tả một giá trị có thể ở **một trong nhiều dạng**, nhưng **chỉ một dạng tại một thời điểm**.

Ví dụ – một kết quả của API có thể là:

```
Result = Success(data) | Failure(error)

```

Tức là:

- Hoặc thành công → có data
- Hoặc thất bại → có lỗi
- *Không có trường hợp nào khác*

Điều này an toàn hơn rất nhiều so với `null`, boolean flag, hay exception.

### **2.2 Pattern Matching là gì?**

Pattern Matching là kỹ thuật **so khớp cấu trúc** của Sum Type để xử lý từng trường hợp một cách rõ ràng và tường minh.

Thay vì:

```tsx
if (result.status === "success") { ... }
else if (result.status === "error") { ... }

```

Bạn viết:

```tsx
match(result)
  .with({ type: "success" }, ({ data }) => ...)
  .with({ type: "error" }, ({ error }) => ...)
  .exhaustive();

```

`.exhaustive()` buộc compiler kiểm tra *tất cả nhánh đã được xử lý* → không cho phép quên case.

---

## **3. Ứng dụng của Sum Type + Pattern Matching**

### **3.1 Quản lý lỗi (thay thế exception)**

Trong nhiều ngôn ngữ hiện đại (Rust, Haskell, TS, Swift…):

```
Result<T, E> = Ok(T) | Err(E)

```

Lợi ích:

- Không throw lung tung → flow logic dễ dự đoán.
- Compiler ép phải xử lý đầy đủ.
- Code test và code gọi hàm rõ ràng hơn.

### **3.2 Thay thế null bằng Option**

```
Option<T> = Some(T) | None

```

Ứng dụng:

- Optional field
- Giá trị có thể tồn tại hoặc không
- Tránh null pointer exception

### **3.3 Biểu diễn trạng thái domain**

Ví dụ trạng thái đơn hàng:

```
OrderState = Created | Paid | Shipping | Completed | Cancelled

```

Pattern Matching đảm bảo mọi trường hợp đều được xử lý khi viết workflow.

### **3.4 Xử lý API response**

```
ApiResponse =
  | { type: "loading" }
  | { type: "error"; message: string }
  | { type: "success"; data: User }

```

Pattern Matching loại bỏ if/else cồng kềnh → UI logic rất sạch.

### **3.5 State Machine / Workflow Engine**

Sum Type cực phù hợp cho mô hình State Machine vì bản chất nó mô tả *các trạng thái rời rạc*.

---

## **4. Kết luận**

Sum Type + Pattern Matching là một tư duy kiểu dữ liệu hiện đại giúp:

- **Loại bỏ null**
- **Loại bỏ exception tràn lan**
- **Ép compiler kiểm tra đầy đủ** → code an toàn hơn
- **Biểu diễn domain rõ ràng**
- **Đơn giản hóa state machine**
- **Giảm if/else, giảm bug**