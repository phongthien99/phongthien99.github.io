---
title: "JavaScript Bundle Size as a Frontend Architecture Concern"
author: phongthien99
date: 2025-12-20 11:09:00 +0700
categories: [Fe]
tags: [micro-frontend]
math: true
media_subpath: '/posts/20251026'

---
## Đặt vấn đề

Trong các ứng dụng frontend hiện đại, **kích thước JavaScript bundle ban đầu** là yếu tố then chốt ảnh hưởng đến **thời gian tải, khả năng tương tác sớm (Time to Interactive) và hiệu năng thực tế trên client**.

Bundle lớn làm tăng chi phí download, parse và execute trên main thread, đặc biệt rõ rệt trên thiết bị di động và môi trường mạng có độ trễ cao. Hệ quả là **perceived performance suy giảm**, ngay cả khi backend và UI đã được tối ưu tốt.

Bên cạnh tác động đến trải nghiệm người dùng, mỗi KB JavaScript được phân phối qua CDN sẽ **nhân lên theo số lượng request**, khiến **chi phí CDN tăng tuyến tính theo traffic nếu không có chiến lược chia chunk và cache phù hợp**.

Do đó, kiểm soát bundle size không chỉ là một bước tối ưu ở cuối pipeline, mà là một **quyết định kiến trúc frontend** cần được cân nhắc từ sớm.

---

## Giải pháp

Để kiểm soát kích thước bundle và tối ưu hiệu năng tổng thể, có thể kết hợp ba kỹ thuật cốt lõi theo đúng thứ tự từ build-time đến runtime:

- **Tree Shaking**: loại bỏ code không sử dụng ở giai đoạn build
- **Vendor Chunking**: tách và cache hiệu quả các thư viện ổn định
- **Lazy Loading**: trì hoãn tải code không cần thiết cho luồng ban đầu

Ba kỹ thuật này bổ trợ lẫn nhau, giúp giảm bundle từ gốc, tối ưu phân phối qua CDN và cải thiện trải nghiệm người dùng.

---

## Thực hiện và Ví dụ minh họa

### 1. Tree Shaking

Tree Shaking được thực hiện ở giai đoạn build nhằm loại bỏ các module, function và export không được sử dụng khỏi bundle cuối cùng.

### ❌ BAD

```tsx
import *asIconsfrom'lucide-react';

functionHeader() {
return<Icons.Home />;
}

```

- Import toàn bộ module
- Bundler không thể tree shake hiệu quả
- Làm tăng kích thước bundle không cần thiết

### ✅ GOOD

```tsx
import {Home }from'lucide-react';

functionHeader() {
return<Home />;
}

```

- Import theo named export
- Chỉ bundle code thực sự được sử dụng

---

### 2. Vendor Chunking

Vendor Chunking tách code ứng dụng khỏi các thư viện bên thứ ba có vòng đời ổn định nhằm cải thiện hiệu quả cache.

### ❌ BAD

```tsx
// Không cấu hình chia chunk
// Vendor và app code bị gộp chung

```

- Mỗi lần deploy invalidate toàn bộ bundle
- Cache CDN và trình duyệt kém hiệu quả

### ✅ GOOD (Vite)

```tsx
// vite.config.ts
exportdefault {
build: {
rollupOptions: {
output: {
manualChunks: {
'vendor-react': ['react','react-dom'],
'vendor-ui': ['lucide-react','@radix-ui/react-icons'],
        },
      },
    },
  },
};

```

- Vendor chunk ổn định
- Giảm lượng JavaScript cần tải lại khi deploy

---

### 3. Lazy Loading

Lazy Loading được áp dụng ở runtime nhằm trì hoãn việc tải các route và component không thuộc luồng sử dụng ban đầu.

### ❌ BAD

```tsx
importDashboardfrom'@/pages/Dashboard';
importSettingsfrom'@/pages/Settings';

<Routes>
<Routepath="/dashboard"element={<Dashboard />} />
<Routepath="/settings"element={<Settings />} />
</Routes>

```

- Eager load toàn bộ route
- Initial bundle lớn

### ✅ GOOD

```tsx
import { lazy,Suspense }from'react';

constDashboard =lazy(() =>import('@/pages/Dashboard'));
constSettings =lazy(() =>import('@/pages/Settings'));

<Suspensefallback={<Loading />}>
<Routes>
<Routepath="/dashboard"element={<Dashboard />} />
<Routepath="/settings"element={<Settings />} />
</Routes>
</Suspense>

```

- Route chỉ được load khi cần
- Giảm initial bundle
- Cải thiện Time to Interactive

---

## Kết luận

Việc kết hợp **Tree Shaking → Vendor Chunking → Lazy Loading** tạo thành một chiến lược tối ưu bundle toàn diện cho React:

- Giảm kích thước bundle ngay từ build-time
- Tận dụng cache CDN và trình duyệt hiệu quả hơn
- Cải thiện trải nghiệm người dùng ở thời điểm tải ban đầu