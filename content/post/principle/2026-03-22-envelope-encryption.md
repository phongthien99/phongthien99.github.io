---
title: "Envelope Encryption"
author: phongthien99
date: 2026-03-22 00:00:00 +0700
categories: ["Security"]
tags: [Design-pattern, Security Data]
math: true

---
# Envelope Encryption


## 1. Đặt vấn đề

Trong kiến trúc hệ thống hiện đại, mã hóa dữ liệu (encryption at rest, encryption in transit) không còn là lựa chọn — nó là yêu cầu bắt buộc. Tuy nhiên, khi khối lượng dữ liệu tăng lên hàng terabyte, thậm chí petabyte, một câu hỏi kỹ thuật trọng yếu xuất hiện: **làm thế nào để mã hóa hiệu quả mà vẫn đảm bảo an toàn cho khóa mã?**

Hãy hình dung tình huống: bạn vận hành một hệ thống lưu trữ hàng triệu file trên object storage hoặc một cụm database chứa hàng tỷ bản ghi. Nếu sử dụng một khóa duy nhất (master key) để mã hóa trực tiếp toàn bộ dữ liệu, bạn đối mặt với ba vấn đề nghiêm trọng:

**Thứ nhất, hiệu năng.** Các thuật toán mã hóa bất đối xứng (RSA, ECC) cung cấp mức bảo mật cao nhưng cực kỳ chậm khi xử lý dữ liệu lớn. 

**Thứ hai, blast radius.** Nếu master key bị lộ, toàn bộ dữ liệu trong hệ thống đều bị compromise. Không có cơ chế cách ly thiệt hại, một sự cố nhỏ biến thành thảm họa toàn diện.

**Thứ ba, key rotation.** Khi cần xoay khóa (key rotation) — một thực hành bảo mật bắt buộc theo các chuẩn như PCI-DSS, HIPAA — bạn phải giải mã rồi mã hóa lại toàn bộ dữ liệu. Với hệ thống petabyte-scale, việc này đòi hỏi thời gian downtime không thể chấp nhận được và chi phí tính toán khổng lồ.

Mô hình dưới đây minh họa rõ điểm yếu của cách tiếp cận single-key truyền thống

```mermaid
flowchart LR
    subgraph Problem["❌ Single-Key Encryption"]
        MK["🔑 Master Key<br/><i>Một khóa duy nhất</i>"]
        D1["📄 Data 1<br/>"]
        D2["📄 Data 2<br/>"]
        D3["📄 Data 3<br/>"]
        DN["📄 ...<br/>"]

        MK -->|"RSA / trực tiếp"| D1
        MK -->|"RSA / trực tiếp"| D2
        MK -->|"RSA / trực tiếp"| D3
        MK -->|"RSA / trực tiếp"| DN
    end

    style Problem fill:#FFF0F0,stroke:#CC3333
    style MK fill:#FF6666,stroke:#CC0000,color:#fff
```

Khi master key bị lộ → toàn bộ Data 1, 2, 3... N đều bị compromise. Khi cần rotation → phải giải mã và mã hóa lại toàn bộ N terabyte. Rõ ràng, mô hình này không mở rộng được.

---

## 2. Giải pháp: Envelope Encryption

Envelope Encryption (mã hóa phong bì) là mô hình phân tầng khóa, trong đó dữ liệu không bao giờ được mã hóa trực tiếp bằng master key. Thay vào đó, quy trình hoạt động theo nguyên tắc **"dùng khóa để mã hóa khóa"**.

### 2.1. Kiến trúc phân tầng

Mô hình bao gồm hai lớp khóa:

**Data Encryption Key (DEK)** — khóa đối xứng (AES-256) được sinh ngẫu nhiên cho **mỗi** đối tượng dữ liệu. DEK thực hiện việc mã hóa/giải mã dữ liệu thực tế. Do là khóa đối xứng, DEK xử lý dữ liệu lớn với tốc độ rất cao.

**Key Encryption Key (KEK)** — khóa cấp cao hơn, được quản lý bởi dịch vụ chuyên biệt (KMS). KEK chỉ làm một việc duy nhất: mã hóa (wrap) và giải mã (unwrap) các DEK. Vì DEK chỉ là 32 bytes, thao tác này diễn ra gần như tức thì.

```mermaid
flowchart TB
    subgraph Envelope["✅ Envelope Encryption"]
        KEK["🔐 KEK <i>(Master Key)</i><br/>Nằm trong KMS / Vault<br/>Không bao giờ rời khỏi hệ thống quản lý khóa"]

        DEK1["🔑 DEK 1"]
        DEK2["🔑 DEK 2"]
        DEK3["🔑 DEK 3"]
        DEKN["🔑 DEK N"]

        D1["📄 Data 1"]
        D2["📄 Data 2"]
        D3["📄 Data 3"]
        DN["📄 Data N"]

        KEK -->|"AES-256-KW<br/>(RFC 3394)"| DEK1
        KEK -->|"AES-256-KW<br/>(RFC 3394)"| DEK2
        KEK -->|"AES-256-KW<br/>(RFC 3394)"| DEK3
        KEK -->|"AES-256-KW<br/>(RFC 3394)"| DEKN

        DEK1 -->|"AES-256-GCM<br/>(nhanh)"| D1
        DEK2 -->|"AES-256-GCM<br/>(nhanh)"| D2
        DEK3 -->|"AES-256-GCM<br/>(nhanh)"| D3
        DEKN -->|"AES-256-GCM<br/>(nhanh)"| DN
    end

    style Envelope fill:#F0FFF0,stroke:#339933
    style KEK fill:#2E75B6,stroke:#1A4D80,color:#fff
    style DEK1 fill:#70AD47,stroke:#507D32,color:#fff
    style DEK2 fill:#70AD47,stroke:#507D32,color:#fff
    style DEK3 fill:#70AD47,stroke:#507D32,color:#fff
    style DEKN fill:#70AD47,stroke:#507D32,color:#fff
    style D1 fill:#FFC000,stroke:#BF9000,color:#333
    style D2 fill:#FFC000,stroke:#BF9000,color:#333
    style D3 fill:#FFC000,stroke:#BF9000,color:#333
    style DN fill:#FFC000,stroke:#BF9000,color:#333
```

### 2.2. Luồng Encrypt / Decrypt

```mermaid
sequenceDiagram
    participant App as 🖥️ Application
    participant KMS as 🔐 Key Management Service
    participant Store as 🗄️ Object Storage

    Note over App,Store: ── ENCRYPT ──

    App->>App: 1. Sinh DEK ngẫu nhiên (32 bytes)
    App->>App: 2. AES-256-GCM Encrypt(data, DEK) → ciphertext
    App->>KMS: 3. Wrap(DEK) bằng KEK
    KMS-->>App: Encrypted DEK
    App->>Store: 4. Lưu ciphertext + Encrypted DEK
    App->>App: 5. 🗑️ Xóa plaintext DEK khỏi RAM

    Note over App,Store: ── DECRYPT ──

    App->>Store: 6. Đọc ciphertext + Encrypted DEK
    Store-->>App: ciphertext + Encrypted DEK
    App->>KMS: 7. Unwrap(Encrypted DEK)
    KMS-->>App: Plaintext DEK
    App->>App: 8. AES-256-GCM Decrypt(ciphertex, DEK) → data
    App->>App: 9. 🗑️ Xóa plaintext DEK khỏi RAM
```

### 2.3. Lợi ích

**Hiệu năng** — Dữ liệu lớn được mã hóa bằng AES-256 (symmetric), nhanh gấp hàng nghìn lần so với RSA. KEK chỉ xử lý 32 bytes DEK nên không tạo bottleneck.

**Blast radius** — Mỗi object có DEK riêng. Nếu một DEK bị lộ, chỉ object tương ứng bị ảnh hưởng. Master key vẫn an toàn trong KMS.

**Key rotation** — Đây là lợi thế lớn nhất. Nhờ phân tầng, key rotation **không bao giờ chạm vào dữ liệu gốc**. Ciphertext giữ nguyên 100%.

---

## 3. Kết luận

Envelope Encryption không phải là một ý tưởng mới — nó đã là tiêu chuẩn ngành từ nhiều năm. AWS S3 SSE, Google Cloud Storage, Azure Blob Storage đều sử dụng mô hình này bên dưới.

Bản chất của Envelope Encryption là phân tách hai mối quan tâm: **hiệu năng** (AES symmetric cho dữ liệu lớn) và **bảo mật khóa** (KMS/Vault quản lý KEK, key master không bao giờ rời khỏi hệ thống chuyên dụng). Sự phân tách này cho phép mỗi lớp tối ưu cho đúng mục tiêu của nó.