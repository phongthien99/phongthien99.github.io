---
title: Database-Centric Architecture
author: phongthien99
date: 2025-12-12 00:40:00 +0800
categories: [Design-pattern]
tags: [gest]
math: true
media_subpath: '/posts/20240503'
---

# **Database-Centric Architecture**

## **1. Giới thiệu**

**Database-Centric Architecture** là kiểu kiến trúc phần mềm mà **cơ sở dữ liệu đóng vai trò trung tâm của hệ thống**.

Thay vì xử lý logic nghiệp vụ ở tầng ứng dụng, phần lớn logic được đưa vào database thông qua:

- Stored Procedures
- Triggers
- Views / Materialized Views
- Constraints
- RLS / CLS
- Transaction logic

Ứng dụng đóng vai trò chủ yếu là **giao diện tương tác** (presentation + API gateway) hơn là nơi xử lý logic.

---

## **2. Lợi ích**

### **2.1. Dữ liệu nhất quán và integrity cao**

- DB đảm nhiệm toàn bộ constraint, validation và loại bỏ logic trùng lặp.
- Nhiều clients truy cập vẫn đảm bảo ACID, hạn chế lỗi race-condition hoặc mismatch logic.

### **2.2. Business logic tập trung**

- Quy tắc nghiệp vụ tập trung tại một nơi duy nhất.
- Dễ kiểm soát, audit và giảm sai lệch giữa các service hoặc ứng dụng client khác nhau.

### **2.3. Bảo mật mạnh và chặt chẽ**

- Dùng RLS, CLS, roles giúp kiểm soát truy cập theo từng dòng dữ liệu hoặc từng cột.
- Không phụ thuộc ứng dụng để enforce security — tăng tính an toàn.

### **2.4. Phù hợp cho hệ thống giao dịch**

- Banking, ERP, CRM, healthcare,… nơi consistency cực kỳ quan trọng.
- Các stored procedure chạy nội bộ giúp giảm round-trip và tăng tốc xử lý giao dịch.

---

## **3. Nhược điểm**

### **3.1. Vấn đề về Scale**

- **Database trở thành bottleneck**: CPU, RAM, IO đều dồn vào DB.
- **Khó scale ngang**: sharding, replication phức tạp khi logic nằm trong DB.
- **Tăng latency** khi nhiều trigger/procedure nặng.
- **Read replicas** không thực hiện được logic write, gây lệ thuộc master.

### **3.2. Vấn đề về Maintainability**

- **Ở đâu cũng có logic**: triggers, procedure, view → khó debug, khó trace.
- **Khó version control**: không đồng bộ với Git/CI/CD tốt như code application.
- **Schema thay đổi phức tạp**: dễ gây lỗi dây chuyền trong trigger/procedure.
- **Đòi hỏi skill đặc thù** như SQL nâng cao, locking, query planning.
- **Coupling cao**: đổi DB hoặc chuyển sang microservices rất tốn kém.

---

## **4. Case Study — Khi nào nên dùng Database-Centric Architecture**

### **4.1. Ngành tài chính – ngân hàng**

- Cần ACID tuyệt đối.
- Giao dịch tiền tệ: chuyển khoản, thanh toán, kiểm tra số dư.
- Stored procedure đảm bảo logic nhất quán, rollback an toàn.

**Triển khai điển hình:**

- SP xử lý giao dịch (debit/credit).
- Trigger tạo lịch sử giao dịch.
- Constraint ngăn mất cân bằng tài khoản.
- RLS đảm bảo user chỉ thấy tài khoản của họ.
- Materialized view dành cho báo cáo tổng hợp.

### **4.2. ERP / CRM / Accounting**

- Dữ liệu phức tạp nhưng consistency quan trọng hơn performance.
- Quy trình nghiệp vụ ổn định, ít thay đổi.



## **5. Kết luận**

Database-Centric Architecture phù hợp cho các hệ thống:

- **Dữ liệu là trung tâm**
- **Tính toàn vẹn (integrity) và nhất quán là ưu tiên hàng đầu**
- **Nghiệp vụ ổn định, ít thay đổi**
- **Số lượng giao dịch lớn nhưng user không quá đông**

Tuy nhiên, không phù hợp cho:

- Hệ thống cần **scale-out lớn**, traffic cao (social network, ecommerce lớn)
- **Microservices architecture**
- Hệ thống cần release nhanh, thay đổi thường xuyên

**Tóm lại:**

Database-Centric Architecture phát huy tối đa trong môi trường yêu cầu **độ chính xác cao**, **tính nhất quán mạnh**, và **quy tắc nghiệp vụ ổn định**. Nhưng nó phải đánh đổi bằng khả năng mở rộng và chi phí bảo trì lâu dài.