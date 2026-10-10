---
title: "Giới thiệu ArchCode: Visual Architecture as Code cho đội ngũ dùng Git"
author: phongthien99
date: 2026-10-10 22:20:00 +0700
categories: [SA]
tags: [architecture-as-code, software-architecture, c4, documentation, tooling]
slug: "introducing-archcode-visual-architecture-as-code"
---

# Introducing ArchCode: Visual Architecture as Code

Hầu hết sơ đồ kiến trúc đều bắt đầu với một ý định rất tốt.

Ở đầu dự án, ai đó vẽ một sơ đồ rõ ràng: service, database, queue, gateway, hệ thống bên thứ ba, và vài mũi tên quan trọng. Nhìn vào đó, cả đội hiểu hệ thống nhanh hơn.

Rồi hệ thống thay đổi.

Một service mới được thêm vào. Một database bị tách ra. Một queue trở thành một phần của luồng xử lý. API Gateway nhận thêm trách nhiệm mới. Nhưng sơ đồ thì thường vẫn đứng yên.

Sau vài tháng, sơ đồ đó không còn là tài liệu kiến trúc nữa. Nó chỉ là ký ức về kiến trúc ở một thời điểm cũ.

**ArchCode** là một thử nghiệm để làm cho sơ đồ kiến trúc dễ sống lâu hơn.

Ý tưởng cốt lõi rất đơn giản:

> Sơ đồ kiến trúc nên đủ trực quan để hiểu nhanh, nhưng cũng phải đủ có cấu trúc để review, validate và version như code.

## Vấn đề với sơ đồ kiến trúc truyền thống

Các công cụ vẽ sơ đồ truyền thống rất tốt cho việc vẽ, nhưng yếu ở phần bảo trì lâu dài.

Chúng thường có vài vấn đề:

- sơ đồ được lưu dưới dạng ảnh hoặc file proprietary,
- thay đổi khó review trong Pull Request,
- model phía sau sơ đồ không dễ validate,
- team khó chạy kiểm tra kiến trúc trong CI,
- tài liệu nhanh bị cũ vì việc cập nhật sơ đồ tách rời khỏi flow phát triển hằng ngày.

Điều này tạo ra khoảng cách giữa hệ thống thật và hệ thống được tài liệu hóa.

Kết quả rất quen thuộc: sơ đồ vẫn tồn tại, nhưng team không còn hoàn toàn tin vào nó.

## Architecture as Code

Architecture as Code cố gắng giải quyết vấn đề này bằng cách biến kiến trúc thành một artifact dạng text.

Thay vì xem sơ đồ như một hình ảnh tĩnh, source of truth trở thành một file có cấu trúc, có thể sống trong Git:

```yaml
version: 1

system:
  id: document-platform
  name: Document Platform

components:
  - id: web-app
    type: frontend
    technology: React

  - id: document-service
    type: service
    technology: Go

  - id: document-db
    type: database
    technology: PostgreSQL

relationships:
  - from: web-app
    to: document-service
    type: http
    label: REST /documents

  - from: document-service
    to: document-db
    type: sql

views:
  - id: overview
    name: System Overview
    include: ['*']
    direction: LR
```

Cách này tạo ra một workflow tốt hơn:

- thay đổi kiến trúc được review như YAML diff,
- diagram được sinh ra từ model,
- CI có thể validate rule kiến trúc,
- tài liệu có thể publish tự động,
- model có thể tiến hóa cùng codebase.

## ArchCode nằm ở đâu?

ArchCode không cố thay thế mọi công cụ vẽ sơ đồ.

Nó tập trung vào một workflow cụ thể:

> Developer hoặc architect mô tả hệ thống phần mềm bằng YAML, chỉnh trực quan khi cần, validate trong CI, rồi publish thành tài liệu tĩnh.

Hướng đi hiện tại:

- **YAML-first**: architecture model là plain text.
- **Visual-first editing**: người dùng vẫn có thể chỉnh qua canvas.
- **Git-friendly**: thay đổi semantic nằm trong YAML; layout có thể nằm ở file riêng.
- **Browser-only**: không cần backend để render hoặc edit.
- **CI-ready**: cùng core engine có thể validate model từ CLI.
- **Docs-friendly**: diagram có thể embed vào Hugo hoặc các static documentation site khác.

Chính tổ hợp này mới là điểm đáng chú ý.

## Chỉnh trực quan nhưng không đánh mất source of truth

Trong nhiều team thường có hai nhóm người dùng:

- người thích sửa trực tiếp bằng text,
- người thích nhìn và thao tác trên diagram.

ArchCode cố gắng hỗ trợ cả hai.

Developer có thể sửa YAML trực tiếp:

```yaml
relationships:
  - from: api-gateway
    to: document-service
    type: http
```

Architect có thể dùng canvas để thêm node, nối service, kéo thả component.

Điểm quan trọng là cả hai thao tác đều cập nhật cùng một model.

Diagram không phải artifact tách biệt. Nó là một projection của architecture model.

## Rule và validation

Sơ đồ sẽ hữu ích hơn khi nó có thể phát hiện vấn đề kiến trúc.

ArchCode hỗ trợ các rule đơn giản như:

```yaml
rules:
  - id: no-cycles
    type: no-circular
    severity: error

  - id: no-orphans
    type: no-orphan
    severity: warning

  - id: no-cross-domain-db
    type: forbidden
    severity: error
    from: { type: service }
    to: { type: database }
    unless: same-group
```

Những rule này giúp team bắt các vấn đề như:

- vòng phụ thuộc,
- component bị mồ côi,
- dependency bị cấm,
- service truy cập database không thuộc quyền sở hữu của nó.

Cùng một cơ chế validation có thể chạy local hoặc trong CI:

```bash
pnpm archcode validate architecture.yaml
```

Điều này biến tài liệu kiến trúc thành một thứ gần giống fitness function.

## Tài liệu tĩnh

ArchCode có thể sinh output tĩnh, nhưng cũng có thể embed trực tiếp vào documentation site.

Ví dụ, trong Hugo site này, một architecture block có thể được viết như sau:

```md
{{</* archcode title="Document Platform" mode="both" view="overview" */>}}
version: 1

system:
  id: document-platform
  name: Document Platform

components:
  - id: web-app
    type: frontend
  - id: document-service
    type: service

relationships:
  - from: web-app
    to: document-service
    type: http

views:
  - id: overview
    name: Overview
    include: ['*']
    direction: LR
{{</* /archcode */>}}
```

Embed có ba chế độ hiển thị:

- `preview`: chỉ hiện diagram đã render,
- `code`: chỉ hiện YAML source,
- `both`: hiện cả diagram và YAML.

Dưới đây là ví dụ render trực tiếp ngay trong bài viết.

### View diagram

{{< archcode title="Document Platform - Diagram" mode="preview" view="overview" >}}
version: 1

system:
  id: document-platform
  name: Document Platform

components:
  - id: web-app
    type: frontend
  - id: document-service
    type: service

relationships:
  - from: web-app
    to: document-service
    type: http

views:
  - id: overview
    name: Overview
    include: ['*']
    direction: LR
{{< /archcode >}}

### View code

{{< archcode title="Document Platform - Source" mode="code" view="overview" >}}
version: 1

system:
  id: document-platform
  name: Document Platform

components:
  - id: web-app
    type: frontend
  - id: document-service
    type: service

relationships:
  - from: web-app
    to: document-service
    type: http

views:
  - id: overview
    name: Overview
    include: ['*']
    direction: LR
{{< /archcode >}}

Nếu muốn reviewer vừa xem diagram vừa kiểm tra YAML trong cùng một block, dùng `mode="both"`:

```md
{{</* archcode title="Document Platform" mode="both" view="overview" */>}}
...
{{</* /archcode */>}}
```

Cách này hữu ích cho blog post, tài liệu thiết kế kỹ thuật, hoặc architecture portal nội bộ.

Bạn có thể thử demo tại đây:

[Mở ArchCode](/archcode/)

## Điểm khác biệt là gì?

Trong không gian này đã có nhiều công cụ mạnh như Structurizr, LikeC4, Mermaid và D2.

ArchCode không có giá trị chỉ vì nó tự phát minh lại diagram-as-code. Nó chỉ thật sự có giá trị nếu làm cho một workflow cụ thể trở nên mượt hơn:

- chỉnh trực quan cho người không muốn viết mọi thứ bằng tay,
- YAML source vẫn dễ đọc trong Pull Request,
- chạy hoàn toàn trên browser cho team không muốn vận hành server,
- embed tốt vào tài liệu tĩnh,
- validation nhẹ cho architecture governance.

Đó là niche của ArchCode.

## So sánh với Mermaid và draw.io

Mermaid và draw.io đều là những công cụ rất phổ biến, nhưng chúng phục vụ những nhu cầu hơi khác ArchCode.

| Tiêu chí | Mermaid | draw.io | ArchCode |
|---|---|---|---|
| Cách làm việc chính | Viết diagram bằng text | Vẽ thủ công trên canvas | Mô tả architecture model bằng YAML, rồi render thành diagram |
| Review trong Git | Tốt, vì là text | Khó hơn, vì diff thường không semantic | Tốt, vì thay đổi nằm trong YAML có cấu trúc |
| Chỉnh trực quan | Hạn chế | Rất mạnh | Có, nhưng vẫn giữ YAML làm source of truth |
| Validation kiến trúc | Không phải trọng tâm | Không phải trọng tâm | Có rule để bắt lỗi kiến trúc |
| Phù hợp nhất cho | Flowchart, sequence diagram, tài liệu kỹ thuật nhanh | Sơ đồ tự do, workshop, bản vẽ trình bày | System architecture, C4-style model, tài liệu sống trong Git |
| CI/CD | Có thể render được | Khó tự động hóa hơn | Có thể validate và publish tự động |

Nếu chỉ cần một sequence diagram hoặc flowchart nhỏ trong README, Mermaid là lựa chọn rất gọn.

Nếu cần vẽ tự do, làm workshop với stakeholder, hoặc tạo một sơ đồ trình bày đẹp nhanh chóng, draw.io vẫn rất tiện.

ArchCode phù hợp hơn khi sơ đồ không chỉ để nhìn, mà còn là một phần của engineering workflow:

- kiến trúc được review cùng Pull Request,
- diagram được sinh từ model thay vì vẽ rời,
- rule kiến trúc có thể chạy trong CI,
- tài liệu được publish lại mỗi khi source thay đổi.

Nói ngắn gọn: Mermaid mạnh ở diagram-as-text, draw.io mạnh ở free-form drawing, còn ArchCode nhắm tới architecture-as-code có validation và workflow Git rõ ràng.

## Trạng thái hiện tại

ArchCode vẫn còn rất sớm.

Những phần hữu ích đã có:

- YAML schema,
- parser và semantic validation,
- diagram rendering,
- visual canvas,
- CLI validation,
- export HTML/SVG,
- Hugo embed mode.

Các bước tiếp theo đáng làm:

- thêm nhiều ví dụ tốt hơn,
- import/export với ecosystem hiện có,
- rule kiến trúc phong phú hơn,
- report thân thiện cho Pull Request,
- package publishing sạch hơn,
- tài liệu tốt hơn cho team muốn áp dụng.

## Kết luận

Tài liệu kiến trúc không nên là một screenshot chậm rãi trở nên sai.

Nó nên là một artifact sống:

- gần với code,
- dễ review,
- dễ render,
- dễ validate,
- dễ publish.

Đó là hướng mà ArchCode đang thử nghiệm.
