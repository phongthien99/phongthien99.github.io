---
title: "Infrastructure as Code (IaC): Automating Infrastructure Management"
author: phongthien99
date: 2025-11-17 00:40:00 +0800
categories: [devops, iac]
tags: [infrastructure-as-code, terraform, ansible, automation, devops]
math: true
media_subpath: '/posts/iac'
---
## **1. Giới thiệu về Infrastructure as Code (IaC)**

Infrastructure as Code (IaC) là phương pháp quản lý và triển khai hạ tầng bằng cách mô tả chúng dưới dạng mã nguồn. Thay vì thao tác thủ công trên giao diện cloud hoặc đăng nhập vào server để cấu hình, mọi thứ được mô tả bằng code và thực thi tự động.

IaC biến hạ tầng thành một phần của codebase:

- có version
- có code review
- có CI/CD
- có khả năng rollback
- có thể tái sử dụng và nhân bản môi trường

Khi hệ thống ngày càng phức tạp, IaC trở thành nền tảng quan trọng để đảm bảo tính ổn định và tốc độ của DevOps/Platform Engineering.

---

## **2. Lợi ích của IaC**

### **2.1. Tự động hóa toàn diện**

Mọi bước từ tạo hạ tầng đến cấu hình ứng dụng đều có thể chạy chỉ bằng một lệnh.

Giảm thời gian triển khai từ hàng ngày xuống vài phút.

### **2.2. Nhất quán giữa các môi trường**

Dev/Staging/Prod giống nhau 100%.

Không còn tình trạng “chạy ở máy tao thì được”.

### **2.3. Dễ kiểm soát & audit**

Do IaC nằm trong Git:

- ghi lại lịch sử thay đổi
- quy trình review rõ ràng
- dễ dàng rollback

### **2.4. Giảm lỗi con người**

Bớt thao tác thủ công → ít cấu hình sai → ít downtime.

### **2.5. Mở rộng, tái sử dụng & chuẩn hóa**

Bạn có thể tạo module, role hoặc template để tạo thêm môi trường mới nhanh chóng.

---

## **3. Các loại IaC chính và các công cụ tương ứng**

IaC trong thực tế luôn xoay quanh **hai nhóm nhiệm vụ lớn nhất**:

---

## **3.1. Infrastructure Provisioning**

Đây là nhóm công cụ dùng để **tạo, thay đổi và quản lý tài nguyên hạ tầng**:

- VM / Server
- VPC, Subnet, Security Group
- Database service (RDS, Aurora, Cloud SQL…)
- Load Balancer
- Kubernetes cluster
- Object storage, Messaging service…

Mục tiêu: **mô tả trạng thái mong muốn (desired state)** và để tool thực hiện.

### **Đặc điểm:**

- Declarative (khai báo trạng thái)
- Quản lý lifecycle (create → update → destroy)
- Dựa trên state để đảm bảo idempotent
- Phù hợp cho cloud-native infra

### **Ví dụ công cụ:**

- **Terraform** – phổ biến nhất, cloud-agnostic
- **AWS CloudFormation** – native của AWS
- **Pulumi** – IaC dùng TypeScript, Python, Go
- **Azure Bicep**, **Google Deployment Manager**

---

## **3.2. Configuration Management**

Nhóm công cụ này dùng để **cấu hình phần mềm và môi trường bên trong server**:

- cài đặt package
- cấu hình file
- khởi tạo user, quyền
- deploy ứng dụng
- cấu hình service (Nginx, MySQL, ClickHouse, Docker)
- quản lý OS-level tasks

Mục tiêu: **điều khiển máy chủ ở tầng hệ điều hành và ứng dụng**.

### **Đặc điểm:**

- Chủ yếu imperative (mệnh lệnh từng bước)
- Chạy qua SSH hoặc agent
- Phù hợp cho config server và deploy app

### **Ví dụ công cụ:**

- **Ansible** – phổ biến nhất, agentless
- **Chef / Puppet** – dùng agent
- **SaltStack**
- **Ansible + SSH / Inventory dynamic**

---

## **Kết hợp hai loại IaC trong thực tế**

Mô hình chuẩn production:

```
Terraform → tạo hạ tầng
    ↓
Ansible → cài đặt & cấu hình bên trong server

```

Ví dụ rất phổ biến:

- Terraform tạo EC2 + Security Group → Ansible cài ClickHouse
- Terraform tạo Kubernetes cluster → Helm deploy workload
- Terraform tạo RDS → Ansible seed data hoặc cấu hình ứng dụng kết nối

Đây là mô hình **hybrid** được hầu hết các team DevOps hiện nay sử dụng.

---

# **4. Kết luận**

Infrastructure as Code (IaC) ngày nay không chỉ là một kỹ thuật mà đã trở thành nền tảng cốt lõi của cách vận hành hạ tầng hiện đại. Bằng việc mô tả toàn bộ môi trường dưới dạng mã nguồn, IaC giúp đội ngũ DevOps và Platform Engineering triển khai hệ thống nhanh hơn, chính xác hơn và ít phụ thuộc vào thao tác thủ công.

Khi chia IaC thành hai nhóm chính — **Infrastructure Provisioning** và **Configuration Management** — chúng ta có một mô hình rõ ràng và thực tiễn để xây dựng pipeline tự động hóa. Công cụ Provisioning đảm bảo hạ tầng cloud được tạo và quản lý ổn định theo state; trong khi đó công cụ Configuration Management giúp cấu hình máy chủ và ứng dụng theo cách linh hoạt và có thể lặp lại.

Sự kết hợp này tạo ra một quy trình triển khai mạnh mẽ, cho phép tổ chức dễ dàng mở rộng, quản lý nhiều môi trường và giảm thiểu rủi ro cấu hình sai lệch. IaC vì vậy trở thành chìa khóa giúp doanh nghiệp vận hành hiệu quả hơn, tăng tốc phát triển sản phẩm và xây dựng nền tảng kỹ thuật ổn định cho tương lai.
