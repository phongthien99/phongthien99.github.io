---
title: "Context Engineering – A Method for Designing Efficient AI Agent Systems with SpecKit"
author: phongthien99
date: 2025-10-18 00:00:00 +0800
categories: [ai]
tags: [ai, context-engineering, prompt-engineering, llm, ai-agent]
math: true
---

# Context Engineering – A Method for Designing Efficient AI Agent Systems with SpecKit

## 1. Bối cảnh: Vì sao phát triển phần mềm với AI Agent lại khó đến vậy?

AI Agents như Claude Code, GitHub Copilot, hay các LLM-based assistant đang dần trở thành đồng đội trong lập trình. Tuy nhiên, nếu bạn từng làm việc với chúng trong dự án thật, có lẽ bạn đã gặp những vấđề quen thuộc:

- **Mất ngữ cảnh (Context Loss)**: AI quên kiến trúc, quy tắc coding, hoặc quyết định kỹ thuật đã thống nhất.

- **Hallucination & sai lệch**: AI gợi ý sai công nghệ (ví dụ: đề xuất Redux dù project đang dùng React Query).

- **Thiếu truy vết (Traceability)**: Không rõ vì sao một quyết định kỹ thuật được đưa ra.

- **Tốn token (Token Inefficiency)**: Mỗi lần làm việc lại phải "nạp" toàn bộ context — tốn kém và chậm.

Trong dự án `next-solid`, nhóm triển khai 5 features. Nhưng khi đến feature thứ 3, AI Agent lại quên rằng project dùng `@tanstack/react-query` — và đề xuất Redux. Lỗi nhỏ, nhưng hậu quả lớn: mất thời gian, sai hướng, và giảm niềm tin vào AI.

**Nguyên nhân?**

👉 Không có hệ thống quản lý ngữ cảnh có cấu trúc, truy vết và tự kiểm chứng.

## 2. Giải pháp: Context Engineering

**Context Engineering** là một phương pháp để thiết kế, quản lý, và duy trì ngữ cảnh (context) cho AI Agents một cách có hệ thống.

**Công thức nền tảng:**

```
Context Engineering = Prompt Engineering + Knowledge Design + Memory Management + Reasoning Alignment
```

### Bốn trụ cột của Context Engineering

1. **Prompt Engineering** – thiết kế prompts có cấu trúc, reusable và có validation gates.

2. **Knowledge Design** – tổ chức kiến thức dự án thành artifacts có schema rõ ràng.

3. **Memory Management** – lưu trữ và truy xuất context hiệu quả, tránh lãng phí token.

4. **Reasoning Alignment** – đảm bảo AI suy luận đúng hướng với triết lý dự án.

## 3. SpecKit – Framework hiện thực hóa Context Engineering

Để áp dụng Context Engineering vào thực tế, chúng tôi xây dựng **SpecKit** – một framework gồm các slash commands giúp AI làm việc có tổ chức và tự kiểm soát hơn.

### Cấu trúc thư mục chính:

```
.claude/commands/
├── speckit.specify.md      # Tạo spec từ mô tả tự nhiên
├── speckit.clarify.md      # Làm rõ yêu cầu mơ hồ
├── speckit.plan.md         # Lập kế hoạch và technical design
├── speckit.tasks.md        # Sinh tasks theo thứ tự phụ thuộc
├── speckit.implement.md    # Triển khai có checkpoint
├── speckit.analyze.md      # Kiểm tra consistency giữa artifacts
├── speckit.checklist.md    # Sinh checklist kiểm thử
└── speckit.constitution.md # Bộ nguyên tắc dự án (memory layer)
```

### Workflow tổng quát

```
User mô tả → /specify → spec.md (+ checklist)
                 ↓
            /clarify → resolved spec.md
                 ↓
              /plan → plan.md + research.md + contracts/
                 ↓
             /tasks → tasks.md (theo dependencies)
                 ↓
            /analyze → báo cáo consistency
                 ↓
          /implement → triển khai + validation checkpoints
```

## 4. Bên trong Context Engineering – Cách SpecKit hiện thực 4 trụ cột

### 1️⃣ Prompt Engineering – Prompts có cấu trúc, có kiểm chứng

Mỗi slash command chỉ làm một nhiệm vụ duy nhất.

- Ví dụ `/specify` chỉ tạo spec, `/plan` chỉ lập kế hoạch.
- Điều này giúp phân tách trách nhiệm rõ ràng và dễ kiểm soát chất lượng.

SpecKit còn dùng kỹ thuật **Progressive Disclosure** – chỉ nạp đúng context cần cho mỗi giai đoạn.

→ Giảm **60–70%** lượng token tiêu thụ mỗi vòng.

Cuối mỗi phase có **Validation Gates**: checklist tự động kiểm tra chất lượng spec, completeness, readiness trước khi qua bước tiếp theo.

### 2️⃣ Knowledge Design – Kiến thức có cấu trúc, có schema

SpecKit chia artifacts thành 3 tầng:

- **Tier 1 – Project Memory**: `CLAUDE.md` (AI context hiện tại) + `constitution.md` (nguyên tắc dự án).

- **Tier 2 – Feature Artifacts**: mỗi feature có `spec.md`, `plan.md`, `tasks.md`, `research.md`, `contracts/`, `data-model.md`.

- **Tier 3 – Templates**: lưu schema chuẩn cho mọi loại artifact.

Mọi file đều tuân theo schema định nghĩa trước, giúp AI hiểu và truy xuất chính xác thay vì phải "đoán".

### 3️⃣ Memory Management – Lưu trữ và truy xuất hiệu quả

SpecKit thiết kế memory theo 4 cấp độ:

| Level | Loại bộ nhớ | Vai trò |
|-------|-------------|---------|
| 1 | Session Memory | Tạm thời, phục vụ conversation hiện tại |
| 2 | Feature Memory | Lưu từng feature, có version control |
| 3 | Project Memory | Ghi nhận state toàn dự án |
| 4 | Template Memory | Đảm bảo consistency và reuse |

Nhờ đó, AI chỉ cần đọc những phần liên quan, không phải "nuốt" toàn bộ 5000 tokens của project mỗi lần.

### 4️⃣ Reasoning Alignment – Căn chỉnh suy luận bằng "Constitution"

SpecKit định nghĩa bộ nguyên tắc trong file `constitution.md`.

Ví dụ:

```markdown
## Principle VI: Clean Architecture with SOLID
- 4 layers: Presentation, Application, Domain, Infrastructure
- Repository pattern bắt buộc
- Hooks chỉ phụ thuộc interface, không phụ thuộc implementation
```

Mỗi lần AI lên plan, SpecKit chạy **Constitution Check** – nếu có vi phạm, quy trình dừng lại ngay.

Ví dụ: feature 003 ban đầu dùng Redux → bị chặn vì "Redux không nằm trong approved dependencies".

**Kết quả**: dùng lại React Query, đảm bảo consistency.

## 5. Kết quả: Khi AI làm việc như một kỹ sư thật

Sau khi áp dụng Context Engineering với SpecKit:

| Lợi ích | Mô tả |
|---------|-------|
| **Consistency** | 5 features cùng tuân theo Clean Architecture, không sai lệch. |
| **Traceability** | Mọi quyết định đều có rationale rõ ràng trong `research.md`. |
| **Quality** | 3 tầng validation – từ nội dung đến tiêu chí chấp nhận. |
| **Efficiency** | Giảm token usage mỗi iteration. |
| **Scalability** | Dễ mở rộng dự án, dễ cập nhật principles và templates. |

## 6. Kết luận

AI Agents không chỉ cần "prompt hay" – mà cần **context tốt**.

Và để có context tốt, chúng ta phải **kỹ sư hóa** cách AI hiểu và ghi nhớ.

**Context Engineering** chính là bước tiến đó.

Còn **SpecKit** là công cụ để biến lý thuyết ấy thành hành động — giúp AI phát triển phần mềm một cách có cấu trúc, có kỷ luật, và có trí nhớ.

---

**✳️ Tóm lại**: Prompt Engineering giúp AI hiểu, nhưng Context Engineering giúp AI hành động nhất quán.