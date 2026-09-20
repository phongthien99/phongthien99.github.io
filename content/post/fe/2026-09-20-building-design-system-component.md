---
title: "Building a Design System Component"
author: phongthien99
date: 2026-09-20 00:00:00 +0700
categories: [Fe]
tags: [frontend, design-system]
math: false
---
# Building a Design System Component

*Trạng thái số liệu tính đến ngày 20/09/2026.*

## Tóm tắt

Khi nhiều dự án frontend cùng công ty dùng chung một Design System, cách phân phối phổ biến nhất là đóng gói thành một thư viện npm. Cách này đơn giản nhưng dẫn đến sự ràng buộc phiên bản giữa các sản phẩm, khó tùy biến cục bộ và khó kiểm soát thay đổi phá vỡ tương thích (breaking change). Bài viết mô tả một hướng khác, lấy cảm hứng từ shadcn/ui: component được lưu trong một registry tập trung, và mỗi sản phẩm dùng công cụ dòng lệnh (CLI) để sao chép mã nguồn vào repository của mình. Hệ thống gồm bốn thành phần: (i) kiến trúc phân tầng Token → Theme → Primitive → Component; (ii) mô hình token hai lớp; (iii) registry mô tả bằng metadata theo schema của shadcn; (iv) CLI TypeScript tự xây dựng. Bài viết kết thúc bằng các kết quả đạt được và hướng phát triển tiếp theo.

**Từ khóa:** Design System, registry, phân phối mã nguồn, Base UI, design token, CLI, khả năng truy cập (accessibility).

---

## 1. Đặt vấn đề

### 1.1. Bối cảnh

Trong một tổ chức có nhiều dự án frontend cùng dùng React và TypeScript, các sản phẩm đều cần một tập component nền tảng như Button, Input, Select, Dialog, Table và Form. Design System ra đời để đáp ứng nhu cầu này, và giá trị của nó thể hiện rõ nhất qua ba hệ quả khi thiếu nguồn chung, tức là khi mỗi nhóm tự xây dựng lại:

1. **Tốc độ phát triển thấp.** Không có component dựng sẵn thì mỗi biểu mẫu mới phải viết lại từ các thành phần nền tảng, có thể mất hai ngày thay vì ghép từ component có sẵn. Chi phí này lặp lại ở từng dự án.
2. **Trải nghiệm người dùng thiếu nhất quán.** Cùng một thuộc tính như `border-radius` hay màu chủ đạo nhưng mỗi dự án dùng một giá trị khác, khiến người dùng bị "giật mình" khi chuyển giữa các trang. Hành vi tương tác như tiêu điểm hay điều hướng bàn phím cũng khác nhau từ nơi này sang nơi khác.
3. **Khó bảo trì.** Giá trị màu được viết cứng rải rác trong hàng trăm tệp, nên đổi nhận diện thương hiệu phải sửa ở rất nhiều chỗ thay vì chỉ một.

Ba vấn đề này tương ứng với ba lợi ích mà một Design System dùng chung cần mang lại: tăng tốc độ phát triển, giữ trải nghiệm nhất quán và dễ bảo trì. Câu hỏi còn lại là nên phân phối Design System đó đến các sản phẩm bằng cách nào.

### 1.2. Phương án cơ sở: gói thư viện `@company/ui`

Phương án trực tiếp nhất là đóng gói component thành một package npm để các sản phẩm cài đặt và import. Khi số sản phẩm tăng, phương án này bộc lộ bốn hạn chế:

| Hạn chế | Biểu hiện |
|---|---|
| Ràng buộc phiên bản | Sửa một prop của Dialog ảnh hưởng đến mọi sản phẩm nâng cấp, còn sản phẩm chưa nâng cấp bị tụt lại |
| Tùy biến khó | Một biến thể nhỏ buộc phải fork hoặc ghi đè CSS bằng selector dài |
| Kiểm soát breaking change kém | Hành vi thay đổi âm thầm, sản phẩm lỗi mà không rõ nguyên nhân |
| Phụ thuộc vào thư viện primitive | Nếu API công khai lộ Base UI hoặc Radix, việc thay thư viện đồng nghĩa với major release cho toàn bộ sản phẩm |

**Câu hỏi nghiên cứu.** Làm thế nào duy trì một nguồn chuẩn duy nhất mà vẫn cho phép mỗi sản phẩm chủ động về tiến độ nâng cấp và mức độ tùy biến?

---

## 2. Phương pháp đề xuất

### 2.1. Mô hình Registry

Thay vì cài đặt tại thời điểm chạy (runtime), component được lưu trong registry tập trung, và sản phẩm sao chép mã nguồn vào repository của mình bằng CLI:

```bash
company-ui add button
# → components/ui/button.tsx và lib/utils.ts được ghi vào sản phẩm
```

```tsx
import { Button } from "@/components/ui/button"; // import từ mã nguồn cục bộ
```

**Bảng 1.** So sánh hai mô hình phân phối.

| Tiêu chí | Gói runtime (`@company/ui`) | Registry (sao chép mã nguồn) |
|---|---|---|
| Vị trí mã nguồn | `node_modules` | Repository của sản phẩm |
| Nâng cấp | Tăng version, có hiệu lực ngay | Chủ động: check → xem diff → apply |
| Tùy biến | Fork hoặc ghi đè CSS | Sửa trực tiếp tệp |
| Breaking change | Ảnh hưởng ngay khi nâng version | Không tự động tác động lên sản phẩm |
| Chi phí vận hành | Thấp | Cần CLI và cơ chế theo dõi độ lệch |

Đánh đổi cốt lõi là sau khi sao chép, **sản phẩm sở hữu mã nguồn**. Vì vậy hệ thống phải có cơ chế xác định sản phẩm đang lệch khỏi chuẩn ở đâu.

### 2.2. Kiến trúc phân tầng

```
Token → Theme → Primitive → Component → Pattern → Product
```

Nguyên tắc phụ thuộc là mỗi tầng chỉ phụ thuộc vào tầng đứng trước. `@company/tokens` là package duy nhất bắt buộc dùng chung ở runtime.

### 2.3. Token hai lớp

```
raw token (--ds-color-primary-600) → semantic (--color-primary) → component
```

Lớp raw có tiền tố `--ds-` để tránh xung đột với biến của sản phẩm. Lớp semantic là lớp duy nhất component được tham chiếu. Kết quả là việc đổi thương hiệu chỉ cần thay đổi ở lớp ánh xạ, không chạm đến component nào.

### 2.4. Phân loại component theo năm cấp

| Cấp | Tên | Ví dụ |
|---|---|---|
| L1 | Primitive | Button |
| L2 | Component | Input, Select |
| L3 | Composite | FormField |
| L4 | Pattern | DataTable |
| L5 | Template | ListPage |

Phân cấp này tạo ngôn ngữ chung cho BA, design, developer khi yêu cầu giao diện mới. Ràng buộc loại trừ là bất kỳ UI nào gọi API, biết route, quyền truy cập hoặc thực thể nghiệp vụ (như Order, Invoice) đều nằm ngoài phạm vi Design System.

---

## 3. Triển khai

### 3.1. Cấu trúc monorepo

Hệ thống dùng pnpm workspace kết hợp Turborepo để build song song, Changesets để quản lý phiên bản, cùng ESLint (typescript-eslint, react-hooks, jsx-a11y) và Prettier để chặn lỗi sớm.

```
design-system/
├── apps/storybook/
├── packages/{tokens, theme, primitives, patterns, icons}/
├── registry/{company/ui, stories}/
├── tooling/company-ui/     # CLI
└── registry.json
```

### 3.2. Từ token đến Tailwind

`@company/theme` ánh xạ token raw sang tên semantic, đồng thời xuất cùng bộ biến qua Tailwind preset để sản phẩm dùng utility class mà vẫn tuân theo token:

```css
:root {
  --color-primary: var(--ds-color-primary-600);
  --color-ring: var(--ds-color-primary-500);
  --radius-md: var(--ds-radius-md);
}
```

Cần lưu ý rằng các component hiện tại **không** dùng utility class của Tailwind. Style nằm trong CSS thường theo quy ước `ds-*` và chỉ tham chiếu biến semantic. Tailwind preset tồn tại để sản phẩm dùng token trong mã của riêng họ.

### 3.3. Ví dụ minh họa: component Button

**Bảng 2.** Các quyết định thiết kế trong `Button` và lý do.

| Quyết định | Lý do |
|---|---|
| `type="button"` làm mặc định | HTML mặc định là `submit`, nên nút trong `<form>` sẽ gửi biểu mẫu ngoài ý muốn |
| `disabled={disabled \|\| isLoading}` | Vô hiệu hóa thực sự khi đang tải, chặn click đúp và gửi trùng |
| `ComponentPropsWithoutRef<typeof ButtonPrimitive>` | Kế thừa toàn bộ prop của Base UI nhưng API vẫn thuộc công ty |
| `forwardRef` | Cho phép kết hợp với trigger của Tooltip, Popover, Dialog |
| Spinner có `aria-hidden` | Phần tử thuần trang trí |

### 3.4. Metadata registry

Mỗi item được khai báo bằng JSON theo schema của shadcn:

```json
{
  "name": "dialog",
  "type": "registry:ui",
  "dependencies": ["@base-ui/react", "@company/theme"],
  "registryDependencies": ["button", "utils"],
  "files": [
    { "path": "components/dialog/dialog.tsx", "type": "registry:ui", "target": "@ui/dialog.tsx" }
  ]
}
```

- `dependencies` là các package npm cần cài.
- `registryDependencies` là các item khác trong registry mà item này phụ thuộc.
- `target` dùng placeholder như `@ui/dialog.tsx`, không phải đường dẫn cố định. CLI đọc `components.json` của sản phẩm để xác định `@ui` tương ứng `src/components/ui` hay `components/ui`.

---

## 4. Kết luận

Bài viết đã trình bày một cách tiếp cận cho bài toán dùng chung Design System giữa nhiều dự án frontend: thay vì phân phối component dưới dạng gói thư viện, hệ thống phân phối chính mã nguồn qua một registry tập trung theo hướng *code distribution* và *open code*. Sự khác biệt cốt lõi nằm ở quyền sở hữu. Registry giữ vai trò nguồn chuẩn, còn mỗi sản phẩm sở hữu bản sao của mình nên tự quyết định thời điểm nâng cấp và mức độ tùy biến, và breaking change không tự động tác động lên sản phẩm. Đổi lại, sản phẩm phải chịu trách nhiệm về mã đã sao chép, và tổ chức phải có công cụ để nhìn ra sự lệch chuẩn.