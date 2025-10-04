---
title: "Evolutionary Architecture: Integrating C4 Model, ADR, and Version Control"
author: phongthien99
date: 2025-10-04 00:40:00 +0800
categories: [SA]
tags: [SA]
math: true
---

# Evolutionary Architecture: Integrating C4 Model, ADR, and Version Control

---

## 🧩 1. Đặt vấn đề

Trong vòng đời phát triển phần mềm, kiến trúc **luôn thay đổi**.

Một hệ thống có thể:

- Bắt đầu từ monolith,
- Chuyển sang microservices,
- Rồi tiến hóa thành cloud-native với hạ tầng tự động (IaC).

Mỗi thay đổi đó đều là **một quyết định kiến trúc**.

Nếu không được ghi lại và quản lý có hệ thống, đội ngũ sẽ gặp các vấn đề sau:

1. **Không rõ hệ thống hiện tại khác gì so với trước đây.**
2. **Không biết vì sao các quyết định kiến trúc được đưa ra.**
3. **Không thể kiểm tra, khôi phục hay audit lịch sử kiến trúc.**

Điều này dẫn đến hiện tượng “**architecture drift**” — kiến trúc ban đầu dần bị sai lệch so với định hướng thiết kế.

Để tránh tình trạng đó, giải pháp là kết hợp:

- **C4 Model** – mô tả kiến trúc trực quan,
- **ADR (Architecture Decision Record)** – ghi lại lý do thay đổi,
- **Version Control (Git)** – quản lý lịch sử tiến hóa kiến trúc.

---

## ⚙️ 2. Giải pháp: Hệ thống quản lý kiến trúc có phiên bản

### 🔹 2.1. C4 Model – Trực quan hóa kiến trúc ở 4 cấp độ

**C4 Model**, được giới thiệu bởi **Simon Brown**, là một cách tiếp cận mô hình hóa kiến trúc giúp mọi người trong nhóm — từ kỹ sư đến quản lý — đều có thể **hiểu được hệ thống ở đúng mức độ chi tiết cần thiết**.

Tên gọi **C4** xuất phát từ bốn cấp độ mô tả kiến trúc:

### **1. System Context (Ngữ cảnh hệ thống)**

- Mục tiêu: Mô tả hệ thống đang nói đến là gì và nó tương tác với ai hoặc hệ thống nào khác.
- Người đọc: Quản lý dự án, Product Owner, Business Analyst, hoặc bất kỳ ai cần hiểu tổng thể.
- Trả lời câu hỏi: *Hệ thống của chúng ta phục vụ ai và giao tiếp với hệ thống nào?*
- Thể hiện: Các tác nhân bên ngoài (người dùng, hệ thống khác) và mối quan hệ giữa chúng với hệ thống chính.

### **2. Container (Các khối triển khai chính)**

- Mục tiêu: Mô tả hệ thống được chia thành những phần lớn nào — chẳng hạn như web app, API service, database, queue, hoặc worker.
- Người đọc: Kiến trúc sư, kỹ sư DevOps, tech lead.
- Trả lời câu hỏi: *Hệ thống gồm những ứng dụng hay dịch vụ nào và chúng giao tiếp ra sao?*
- Thể hiện: Các container phần mềm chính, công nghệ sử dụng, và mối quan hệ giữa chúng.

### **3. Component (Các thành phần bên trong container)**

- Mục tiêu: Cho biết cấu trúc bên trong của từng container, ví dụ một API có những module nào (controller, service, repository, client...).
- Người đọc: Nhà phát triển và kiến trúc sư hệ thống.
- Trả lời câu hỏi: *Bên trong một service hoặc ứng dụng gồm những phần nào và chúng đảm nhiệm trách nhiệm gì?*

### **4. Code (Chi tiết ở mức lớp hoặc hàm)**

- Mục tiêu: Mô tả cấu trúc chi tiết của code — cách các class, function hoặc package tương tác.
- Người đọc: Nhà phát triển trực tiếp làm việc với codebase.
- Trả lời câu hỏi: *Cấu trúc code thực tế phản ánh kiến trúc ở mức nào?*

| Cấp độ | Mục tiêu | Người đọc chính | Trả lời câu hỏi |
| --- | --- | --- | --- |
| **System Context** | Toàn cảnh hệ thống và mối quan hệ bên ngoài | PO, BA, Business | Hệ thống phục vụ ai, tương tác với ai |
| **Container** | Phân rã hệ thống thành các dịch vụ chính | Architect, DevOps | Có những ứng dụng và công nghệ nào |
| **Component** | Cấu trúc bên trong một container | Developer, Architect | Có những module nào và vai trò ra sao |
| **Code** | Cấu trúc chi tiết của code | Developer | Class, hàm, thư viện tương tác thế nào |

💡 Khi mô hình được viết dưới dạng **diagram as code** (như Mermaid hoặc Structurizr DSL), các file này có thể được version hóa trong Git, giúp theo dõi và so sánh dễ dàng giữa các phiên bản kiến trúc.

---

### 🔹 2.2. ADR – Ghi lại lý do của mỗi quyết định kiến trúc

**ADR (Architecture Decision Record)** là cách ghi nhận *vì sao* một quyết định kỹ thuật được đưa ra tại một thời điểm cụ thể.

Mỗi ADR nên trả lời được ba câu hỏi:

1. **Bối cảnh:** Tại sao cần thay đổi?
2. **Quyết định:** Giải pháp được chọn là gì?
3. **Hệ quả:** Tác động của quyết định này (tích cực hoặc tiêu cực) là gì?

Ví dụ:

```
# 003 - Chuyển từ REST sang Event-driven Architecture
Ngày: 2025-07-01
Trạng thái: Superseded by ADR-005
Người đề xuất: Giang

Bối cảnh:
REST API không đáp ứng khối lượng giao dịch cao trong giờ cao điểm.

Quyết định:
Chuyển sang mô hình Event-driven sử dụng SNS + SQS.

Hệ quả:
+ Tăng throughput, giảm độ trễ.
- Tăng độ phức tạp trong kiểm thử và giám sát.

```

Khi một quyết định mới thay thế quyết định cũ, bạn chỉ cần cập nhật trạng thái:

```
Status: Superseded by ADR-005

```

Toàn bộ lịch sử thay đổi được lưu giữ rõ ràng và có thể tra cứu bất kỳ lúc nào.

---

### 🔹 2.3. Version Control – Theo dõi tiến hóa kiến trúc

Lưu **C4 Model** và **ADR** trong Git giúp quản lý kiến trúc như code:

| Hành động | Cách thực hiện |
| --- | --- |
| Xem lại kiến trúc cũ | `git log docs/c4/container.mmd` |
| So sánh giữa hai phiên bản | `git diff v1.2 v1.3 docs/adr/` |
| Quay lại kiến trúc cũ | `git checkout <commit-id> docs/c4/` |
| Gắn liên kết giữa code và quyết định | Commit message: `feat(payment): implement SNS flow (ADR-003)` |

Một cấu trúc dự án có thể như sau:

```
main
├── docs/
│   ├── c4/
│   │   ├── system-context.mmd
│   │   ├── container.mmd
│   │   └── component.mmd
│   └── adr/
│       ├── 001-use-rest-api.md
│       ├── 002-use-postgresql.md
│       └── 003-move-to-event-driven.md
└── src/

```

Mỗi commit nên bao gồm thay đổi **code + ADR + diagram**, đảm bảo rằng kiến trúc, quyết định và hiện thực luôn đồng bộ.

---

## 🧪 3. Thực hành: Quy trình đề xuất

1. **Khi có thay đổi lớn:**
    
    Viết ADR mô tả lý do, cập nhật C4 diagram tương ứng.
    
2. **Commit chuẩn:**
    
    ```bash
    git add docs/adr/004-use-step-functions.md docs/c4/container.mmd src/
    git commit -m "ADR-004: Replace Lambda chain with Step Functions"
    
    ```
    
3. **Gắn tag kiến trúc:**
    
    ```bash
    git tag arch-v1.0
    
    ```
    
4. **Tích hợp CI/CD:**
    
    Tự động render biểu đồ, xuất ADR sang wiki hoặc documentation portal, và triển khai tài liệu kiến trúc mới nhất.
    

---

## 🧭 4. Kết luận

Khi kết hợp **C4 Model**, **ADR**, và **Version Control**, bạn tạo ra một **kiến trúc có thể sống (living architecture)** – nơi mà:

| Thành phần | Vai trò |
| --- | --- |
| 🧩 **C4 Model** | Giúp hiểu kiến trúc hiện tại |
| 🧾 **ADR** | Ghi lại lý do và bối cảnh quyết định |
| 🕓 **Version Control (Git)** | Theo dõi và khôi phục lịch sử thay đổi |

> 🔁 Code thay đổi → Kiến trúc thay đổi → Quyết định được ghi lại → Lịch sử được bảo tồn.
> 

Nhờ đó, đội ngũ có thể:

- Dễ dàng audit và review kiến trúc,
- Giúp người mới nhanh chóng hiểu lịch sử kỹ thuật,
- Và đảm bảo kiến trúc tiến hóa có kiểm soát, thay vì trôi dạt theo thời gian.