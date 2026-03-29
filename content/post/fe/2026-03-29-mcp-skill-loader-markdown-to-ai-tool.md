---
title: "Designing MCP Skill Loader — From Markdown to AI Tool"
author: [phongthien99]
date: 2026-03-29 10:00:00 +0700
categories: [AI, System Design]
tags: [mcp, ai-agent,design-pattern]
math: false
---

## 1. Đặt vấn đề

AI agents đang xuất hiện khắp nơi — từ code assistant trong IDE đến chatbot hỗ trợ khách hàng. Nhưng có một bài toán mà hầu như dự án nào cũng gặp: **làm thế nào để "dạy" agent cách thực hiện một tác vụ cụ thể mà không cần viết code?**

Hãy tưởng tượng ta có một agent cần biết cách review code, viết commit message đúng chuẩn, hay debug một lỗi phức tạp. Cách tiếp cận truyền thống thường như sau:

```go
// ❌ Hard-code mọi thứ vào system prompt
func buildSystemPrompt() string {
    return `You are a code review expert.
    When reviewing code: [200 dòng rules]
    When writing commits: [100 dòng rules]
    When debugging: [150 dòng rules]
    ...`
}
```

Vấn đề với cách này:

- **Không tái sử dụng** — mỗi agent copy-paste cùng một đoạn instructions
- **Khó bảo trì** — thay đổi một quy tắc phải sửa nhiều nơi
- **Không version control** — prompt nằm trong code, khó track history
- **Tốn token** — agent load toàn bộ dù chỉ cần một phần nhỏ

Thêm một ràng buộc: **Model Context Protocol (MCP)** đang nổi lên như giao thức chuẩn để AI agent giao tiếp với tools. Claude Desktop, VS Code, và nhiều AI client đều hỗ trợ MCP.

*Câu hỏi: Có cách nào quản lý "knowledge" của agent theo dạng file — version được, chia sẻ được, load động — và expose qua MCP không?*

---

## 2. Giải pháp

> **Ý tưởng cốt lõi:** Mỗi "skill" là một thư mục chứa file `SKILL.md`. Người viết skill chỉ cần biết Markdown — không cần code. Hệ thống tự scan, parse, và expose qua MCP.

Đây là pattern **Convention over Configuration**: đặt file đúng chỗ, mọi thứ tự động chạy.

```mermaid
graph LR
    subgraph without["❌ Không có Skill Loader"]
        A1[Agent 1] --> P1[Hard-coded Prompt]
        A2[Agent 2] --> P2[Copy-paste Prompt]
        A3[Agent 3] --> P3[Another Copy]
    end
    
    subgraph with["✅ Có Skill Loader"]
        S1[SKILL.md] --> L[Skill Loader]
        S2[SKILL.md] --> L
        L --> MCP[MCP Server]
        MCP --> A4[Any Agent]
    end
```

| Vấn đề | Trước | Sau |
|--------|-------|-----|
| Tái sử dụng | Copy-paste | Một file, nhiều agents |
| Bảo trì | Sửa nhiều nơi | Sửa một `SKILL.md` |
| Token | Load toàn bộ | Lazy load qua Resource |
| Thêm skill | Sửa code, redeploy | Thêm file, rescan |

---

## 3. Thiết kế

### 3.1 Cấu trúc Skill

```
~/.skills/
├── code-review/
│   ├── SKILL.md           ← định nghĩa skill (bắt buộc)
│   └── checklist.md       ← tài liệu tham khảo (lazy load)
├── write-commit/
│   └── SKILL.md
└── debug-go/
    └── SKILL.md
```

File `SKILL.md` = YAML frontmatter + Markdown:

```markdown
---
name: code-review
description: Expert code review guidance
references:
  - checklist.md
---

# Code Review Skill
When reviewing code, follow these guidelines...
```

### 3.2 Kiến trúc tổng quan

```mermaid
graph TD
    subgraph Input["📁 Input Layer"]
        F1[SKILL.md files]
    end
    
    subgraph Core["⚙️ Core Layer"]
        P[Parser] --> R[Registry]
        R --> |"validate"| V[Security Checks]
    end
    
    subgraph Output["🔌 Output Layer"]
        MCP[MCP Server]
        MCP --> PR[Prompts]
        MCP --> RS[Resources]
    end
    
    F1 --> P
    R --> MCP
    
    subgraph Clients["🤖 Clients"]
        C1[Claude Desktop]
        C2[VS Code]
        C3[Custom Agent]
    end
    
    PR --> C1
    RS --> C2
    PR --> C3
```

### 3.3 Các thành phần chính

| Component | Trách nhiệm | Input | Output |
|-----------|-------------|-------|--------|
| **Parser** | Đọc SKILL.md, tách frontmatter/content | File path | Skill struct |
| **Registry** | Scan, lưu trữ, detect collision | Root dir | Map[name]Skill |
| **MCP Server** | Expose skills qua protocol | Registry | Prompts + Resources |

### 3.4 Luồng xử lý

```mermaid
sequenceDiagram
    participant FS as File System
    participant R as Registry
    participant MCP as MCP Server
    participant Agent as AI Agent

    Note over R: Startup
    R->>FS: WalkDir(~/.skills)
    FS-->>R: List of SKILL.md paths
    
    loop Each SKILL.md
        R->>R: Parse & Validate
        R->>R: Check duplicates
        R->>R: Store in memory
    end
    
    R->>MCP: Register Prompts
    R->>MCP: Register Resources
    
    Note over Agent: Runtime
    Agent->>MCP: GetPrompt("code_review")
    MCP-->>Agent: Instructions markdown
    
    Agent->>MCP: ReadResource("skill://code-review/checklist.md")
    MCP-->>Agent: Reference content (lazy)
```

### 3.5 MCP Exposure Model

Mỗi skill được expose theo 2 cách:

```
┌─────────────────────────────────────────────────────────┐
│  Skill: code-review                                     │
├─────────────────────────────────────────────────────────┤
│  📝 Prompt: "code_review"                               │
│     → Agent gọi để lấy instructions                     │
│     → Load ngay khi cần                                 │
├─────────────────────────────────────────────────────────┤
│  📎 Resource: "skill://code-review/checklist.md"        │
│     → Agent đọc khi cần thêm context                    │
│     → Lazy load, không tốn token trước                  │
└─────────────────────────────────────────────────────────┘
```

---

## 4. Kết luận

### Tổng kết kiến trúc

```mermaid
graph LR
    A[SKILL.md] -->|scan| B[Registry]
    B -->|expose| C[MCP Server]
    C -->|HTTP| E[Web Agents]
```

Thiết kế này theo nguyên tắc **Convention over Configuration**: đặt file đúng chỗ, system tự lo phần còn lại.

### Lợi ích

**Đơn giản cho người dùng — Simplicity**
- Chỉ cần biết Markdown để viết skill
- Không config, không registration thủ công
- Thêm skill = tạo thư mục + file

**Tích hợp chuẩn — Standards Compliance**
- MCP protocol — hoạt động với mọi MCP client
- File-based — git, backup, share dễ dàng

**Tiết kiệm tài nguyên — Efficiency**
- Lazy load references qua Resource
- Không nhồi nhét vào context window
