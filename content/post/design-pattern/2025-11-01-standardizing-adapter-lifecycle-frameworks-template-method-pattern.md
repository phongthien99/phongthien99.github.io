---
title: Standardizing Adapter Lifecycle in Frameworks with the Template Method Pattern
author: phongthien99
date: 2025-11-01 00:40:00 +0800
categories: [Design-pattern]
tags: [gest]
math: true
media_subpath: '/posts/20240503'
---


# Standardizing Adapter Lifecycle in Frameworks with the Template Method Pattern

## 1. Đặt vấn đề

Trong một framework nền tảng, ta thường cần hỗ trợ nhiều loại **adapter** khác nhau — chẳng hạn HTTP, gRPC, Kafka, hay Cron.

Dù mục đích của chúng khác nhau (xử lý request, tiêu thụ message, lên lịch job…), nhưng tất cả đều có **chu trình vòng đời tương tự**:

```
Start → ... → Stop
```

Cụ thể:

- Mỗi adapter cần **khởi động (OnStart)** khi ứng dụng chạy.
- Và **dừng lại (OnStop)** khi ứng dụng shutdown.

Uber Fx cung cấp cơ chế `fx.Lifecycle` giúp đăng ký các hook này:

```go
lc.Append(fx.Hook{
    OnStart: func(ctx context.Context) error { ... },
    OnStop:  func(ctx context.Context) error { ... },
})

```

Tuy nhiên, khi framework mở rộng để hỗ trợ nhiều adapter, ta bắt đầu nhận thấy một vấn đề kiến trúc phổ biến:

> Các adapter là những thành phần khác nhau, nhưng lại có nhiều điểm tương đồng đáng kể
> 
> 
> mà **không có cơ chế chứng minh hoặc tận dụng được khả năng tái sử dụng giao diện hay xử lý chung**.
> 

Kết quả là:

- ❌ Mỗi adapter tự viết lại lifecycle giống nhau.
- ❌ Code lặp lại 60–70% (boilerplate Fx hook).
- ❌ Framework không có khung chuẩn để điều phối adapter.
- ❌ Nếu muốn thay đổi logic chung (ví dụ thêm logging, metrics...), phải sửa ở tất cả adapter.

Nói cách khác, **các adapter có chung cấu trúc hoạt động, nhưng framework chưa nắm quyền điều phối vòng đời**,

và điều đó khiến việc mở rộng hoặc bảo trì tốn rất nhiều effort.

Tình huống này cũng phổ biến trong các hệ thống **nhiều adapter** — nơi từng nhóm phát triển có thể tự định nghĩa lifecycle riêng,

khiến framework trở nên rời rạc, thiếu tính thống nhất, và khó chứng minh khả năng tái sử dụng.


## 2. Giải pháp: Template Method Pattern

**Template Method Pattern** là mẫu thiết kế cho phép:

- Xác định **bộ khung (template)** của một quy trình trong lớp cha.
- Cho phép lớp con **ghi đè các bước cụ thể** mà không thay đổi khung tổng thể.

> Nói cách khác: lớp cha quyết định “quy trình diễn ra thế nào”,
> 
> 
> còn lớp con chỉ cần mô tả *“chi tiết từng bước”*.
> 

Áp dụng vào bài toán adapter:

- **Lớp cha (template)** quy định vòng đời chuẩn gồm `OnStart` và `OnStop`.
- **Các adapter con (HTTP, gRPC, Kafka, …)** chỉ cần triển khai logic riêng trong hai bước đó.
- **Uber Fx** (hoặc thư viện tương tự) chỉ đóng vai trò *điều phối* lifecycle, chứ không can thiệp vào cấu trúc template.


## 3. Thực hiện

### a. Định nghĩa interface lifecycle

Trước hết, ta mô hình hóa giao diện vòng đời chung cho mọi adapter:

```go
type AdapterLifecycle interface {
	OnStart(ctx context.Context) error
	OnStop(ctx context.Context) error
}

```

Tất cả adapter (HTTP, gRPC, Kafka, ...) đều tuân theo giao diện này.


### b. Xây dựng lớp template `BaseAdapter`

Lớp này chịu trách nhiệm **định nghĩa khung lifecycle chung**,

và tự động đăng ký vào hệ thống (ví dụ Uber Fx):

```go
type BaseAdapter[T any] struct {
	Config T
}

func (b *BaseAdapter[T]) RegisterLifecycle(lc fx.Lifecycle, impl AdapterLifecycle) {
	lc.Append(fx.Hook{
		OnStart: impl.OnStart,
		OnStop:  impl.OnStop,
	})
}

```

Ở đây:

- `RegisterLifecycle()` là **template method** – quy định quy trình cố định cho mọi adapter.
- `OnStart` / `OnStop` là các bước mở rộng mà lớp con có thể tùy biến.
- `fx.Lifecycle` chỉ là cơ chế điều phối, không ảnh hưởng đến cấu trúc template.


### c. Cụ thể hóa adapter – Sub Class

Mỗi adapter chỉ cần kế thừa `BaseAdapter` và định nghĩa logic riêng của mình:

```go
type HTTPAdapter struct {
	BaseAdapter[Config]
}

func (a *HTTPAdapter) OnStart(ctx context.Context) error {
	fmt.Println("Starting HTTP server...")
	return nil
}

func (a *HTTPAdapter) OnStop(ctx context.Context) error {
	fmt.Println("Stopping HTTP server...")
	return nil
}

```

Và đăng ký adapter thông qua template:

```go
func ForRoot(cfg Config) fx.Option {
	return fx.Invoke(func(lc fx.Lifecycle, adapter *HTTPAdapter) {
		adapter.RegisterLifecycle(lc, adapter)
	})
}

```

Kết quả:

- Toàn bộ adapter tuân theo **vòng đời chuẩn**.
- Chỉ cần định nghĩa phần riêng (OnStart/OnStop).
- Không còn phải copy-paste code lifecycle.


## 4. Kết luận

Việc sử dụng Template Method Pattern giúp framework duy trì :

- **Framework layer** nắm quyền điều phối vòng đời (template).
- **Adapter layer** chỉ định nghĩa phần thực thi đặc thù.
- **Logic chung** (logging, metrics, tracing) có thể thêm một lần, áp dụng cho toàn hệ thống.

Điều này biến lifecycle trở thành một phần **có thể kiểm soát, mở rộng và bảo trì tập trung**,

thay vì nằm rải rác trong từng adapter cụ thể.

Khi framework mở rộng với nhiều adapter khác nhau, việc mỗi adapter tự triển khai vòng đời riêng sẽ nhanh chóng dẫn đến **trùng lặp, khó mở rộng và tốn effort bảo trì**.

Bằng cách áp dụng **Template Method Pattern**, ta chuẩn hóa được quy trình `Start → Stop`,

tách biệt phần cố định (template lifecycle) khỏi phần tùy biến (logic adapter),

giúp framework:

- Dễ mở rộng thêm adapter mới.
- Giảm 60–70% code lặp.
- Quản lý lifecycle thống nhất.
- Giữ cấu trúc sạch, tuân theo Clean Architecture.