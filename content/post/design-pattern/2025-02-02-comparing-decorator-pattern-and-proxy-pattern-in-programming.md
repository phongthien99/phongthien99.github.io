---
title: Comparing Decorator Pattern and Proxy Pattern in Programming
author: phongthien99
date: 2025-02-02 00:40:00 +0800
categories: [Design-pattern]
tags: [interview]
math: true
---

# Comparing Decorator Pattern and Proxy Pattern in Programming

## Giới thiệu
Trong lập trình hướng đối tượng, **Decorator Pattern** và **Proxy Pattern** đều là những mẫu thiết kế thuộc nhóm **Structural Pattern** (mẫu thiết kế cấu trúc). Tuy nhiên, chúng có mục đích sử dụng khác nhau. Nếu bạn đang phân vân khi nào nên dùng Decorator và khi nào nên dùng Proxy, bài viết này sẽ giúp bạn hiểu rõ hơn.

---

## 1. Định nghĩa và Mục đích

### **Decorator Pattern** – Mở rộng hành vi của đối tượng một cách linh hoạt
Decorator Pattern cho phép bạn **thêm hoặc thay đổi hành vi của một đối tượng mà không cần chỉnh sửa mã nguồn của nó**. Điều này đặc biệt hữu ích khi bạn muốn mở rộng tính năng mà không làm thay đổi cấu trúc bên trong.

**Ứng dụng thực tế:**
- **Thêm logging, caching** cho một class mà không sửa đổi class gốc.
- **Mở rộng UI component** trong các framework frontend như React hoặc Vue.

### **Proxy Pattern** – Kiểm soát truy cập đến đối tượng thật
Proxy Pattern cung cấp **một lớp thay thế** để kiểm soát cách truy cập vào một đối tượng. Mục đích chính là **bảo vệ, caching, hoặc lazy loading** đối tượng thực.

**Ứng dụng thực tế:**
- **Lazy loading** hình ảnh trong UI.
- **Proxy database** để kiểm soát quyền truy cập.
- **Remote Proxy** để gọi API từ xa.

---

## 2. So sánh chi tiết

| Tiêu chí          | **Decorator Pattern** | **Proxy Pattern** |
|------------------|----------------------|--------------------|
| **Mục đích**    | Thêm hoặc mở rộng chức năng mà không thay đổi code gốc. | Kiểm soát quyền truy cập, caching, lazy-loading. |
| **Cách hoạt động** | Gói (wrap) đối tượng ban đầu bằng một lớp mới. | Cung cấp một đối tượng thay thế để kiểm soát truy cập. |
| **Tính năng chính** | - Cho phép mở rộng hành vi linh hoạt. <br> - Có thể kết hợp nhiều decorator. | - Có thể chặn truy cập, lazy load đối tượng thật. <br> - Dùng để bảo vệ tài nguyên. |
| **Ví dụ thực tế** | - Thêm logging, caching, thay đổi giao diện UI. | - Proxy database, API Gateway, Virtual Proxy. |

---

## 3. Ví dụ minh họa trong Go

### **Decorator Pattern** – Thêm SMS Notification vào hệ thống Email
```go
package main

import (
	"fmt"
)

// Interface gốc
type Notifier interface {
	Send(msg string)
}

// Cấu trúc gốc
type EmailNotifier struct{}

func (e *EmailNotifier) Send(msg string) {
	fmt.Println("Sending Email:", msg)
}

// Decorator thêm chức năng SMS
type SMSDecorator struct {
	Notifier Notifier
}

func (s *SMSDecorator) Send(msg string) {
	s.Notifier.Send(msg) // Gọi chức năng gốc
	fmt.Println("Sending SMS:", msg)
}

func main() {
	email := &EmailNotifier{}
	notifier := &SMSDecorator{Notifier: email}
	notifier.Send("Hello, Robin!") // Gửi cả email và SMS
}
```

### **Proxy Pattern** – Kiểm soát quyền truy cập vào Server
```go
package main

import (
	"fmt"
)

// Interface gốc
type Server interface {
	HandleRequest(url string)
}

// Đối tượng thực
type RealServer struct{}

func (r *RealServer) HandleRequest(url string) {
	fmt.Println("Real server processing:", url)
}

// Proxy kiểm soát quyền truy cập
type ProxyServer struct {
	realServer *RealServer
}

func (p *ProxyServer) HandleRequest(url string) {
	if url == "/restricted" {
		fmt.Println("Access Denied!")
		return
	}
	p.realServer.HandleRequest(url) // Gọi server thật nếu được phép
}

func main() {
	proxy := &ProxyServer{realServer: &RealServer{}}
	proxy.HandleRequest("/public")
	proxy.HandleRequest("/restricted")
}
```

---

## 4. Khi nào nên sử dụng?

- **Sử dụng Decorator khi** bạn muốn **mở rộng hành vi của một đối tượng mà không thay đổi mã nguồn**.
  - Ví dụ: Thêm logging, caching, hoặc định dạng lại dữ liệu đầu ra.

- **Sử dụng Proxy khi** bạn muốn **kiểm soát cách truy cập vào một đối tượng**.
  - Ví dụ: Chặn truy cập tài nguyên, proxy cho API, hoặc lazy-load đối tượng nặng.

---

## 5. Kết luận
Decorator Pattern và Proxy Pattern đều là những công cụ hữu ích trong lập trình, nhưng chúng phục vụ những mục đích khác nhau:

- **Decorator Pattern** giúp mở rộng hành vi của một đối tượng một cách linh hoạt.
- **Proxy Pattern** giúp kiểm soát cách truy cập vào đối tượng thực.

Tùy vào yêu cầu của dự án mà bạn có thể chọn mẫu thiết kế phù hợp nhất! 🚀

