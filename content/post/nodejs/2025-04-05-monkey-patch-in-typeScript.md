---
title: Monkey Patch In TypeScript
author: phongthien99
date: 2025-04-05 10:17:00 +0800
categories: [Nestjs]
tags: [nodejs]
math: true
media_subpath: '/posts/20180809'
---
# Monkey Patch In TypeScript

Trong quá trình phát triển ứng dụng, đôi lúc bạn muốn **thêm một method mới vào class có sẵn** – đặc biệt khi class đó đến từ thư viện bên ngoài hoặc framework, và bạn **không thể sửa trực tiếp mã nguồn gốc**.

Giải pháp? 👉 **Monkey patch**.

Tuy nhiên, monkey patch nếu dùng không cẩn thận dễ gây ra:

- Ghi đè nhầm method
- Gây xung đột trong nhiều nơi cùng patch
- Khó trace bug vì method "tự nhiên mà có"

Trong bài viết này, mình sẽ hướng dẫn cách monkey patch một cách **có kiểm soát, rõ ràng, và tương thích TypeScript**.

---

## ❓ Monkey Patch Là Gì?

Monkey patch là kỹ thuật cho phép **bổ sung hoặc ghi đè method vào prototype của class đang tồn tại**.

Ví dụ:

```

UserService.prototype.countActiveUsers = function () {
  return 42;
};

```

Cách này đơn giản nhưng **rủi ro cao**. Vậy nên, ta sẽ xây dựng một tiện ích nhỏ để làm việc này an toàn hơn.

---

## 🧱 Tạo Hàm `patchService()` – Monkey Patch Có Kiểm Soát

```

// patchService.ts
type PatchMap<T> = {
  [K in keyof Partial<T>]: T[K];
};

const globalPatchFlag = Symbol.for('__patched_methods__');

export function patchService<T extends object>(
  target: { prototype: T },
  methods: PatchMap<T>,
) {
  if (!(target.prototype as any)[globalPatchFlag]) {
    (target.prototype as any)[globalPatchFlag] = new Set<string>();
  }

  const patched: Set<string> = (target.prototype as any)[globalPatchFlag];

  for (const [key, fn] of Object.entries(methods)) {
    if (patched.has(key)) {
      console.warn(`[patchService] Bỏ qua vì đã patch: ${key}`);
      continue;
    }

    if (typeof fn === 'function') {
      if (key in target.prototype) {
        console.warn(`[patchService] Ghi đè method đã có: ${key}`);
      }

      (target.prototype as any)[key] = fn;
      patched.add(key);
    }
  }
}

```

---

## 📦 Định Nghĩa Class Gốc `UserService`

```

// user.service.ts
export class UserService {
  private name = 'DemoService';

  sayHello() {
    return `Hello from ${this.name}`;
  }
}

```

---

## 🔧 Thêm Method Mới Bằng Patch

```

// user-patch.ts
import { patchService } from './patchService';
import { UserService } from './user.service';

// Mở rộng interface với TypeScript
declare module './user.service' {
  interface UserService {
    countActiveUsers(): Promise<number>;
  }
}

// Patch method mới
patchService(UserService, {
  async countActiveUsers(this: UserService): Promise<number> {
    return 42; // Trả về số cố định cho demo
  },
});

```

---

## 🚀 Chạy Thử

```

// main.ts
import './user-patch';
import { UserService } from './user.service';

async function main() {
  const service = new UserService();

  console.log(service.sayHello()); // ✅ method gốc
  const count = await service.countActiveUsers(); // ✅ method mới
  console.log('Số user active:', count); // Kết quả: Số user active: 42
}

main();

```

---

## ✅ Kết Quả Khi Chạy

```

Hello from DemoService
Số user active: 42

```

---

## ✨ Lợi Ích Khi Dùng `patchService()`

| Tính năng | Lợi ích |
| --- | --- |
| 🧠 Gắn cờ đã patch | Tránh patch lặp lại |
| 🧾 Có log rõ ràng | Dễ debug |
| 🔒 Kiểm soát ghi đè | Giảm rủi ro |
| 🧩 Hỗ trợ interface mở rộng | Tương thích hoàn toàn TypeScript |

---

## 📌 Kết Luận

Monkey patch không phải là kỹ thuật nên lạm dụng, nhưng khi **dùng đúng cách**, bạn có thể **mở rộng hệ thống mà không cần sửa class gốc**. Với `patchService()`, bạn có thể:

- Bổ sung method một cách rõ ràng
- Kiểm soát và hạn chế xung đột
- Viết TypeScript chuẩn, dễ maintain về sau

> Hãy xem monkey patch như một con dao sắc – dùng đúng lúc, đúng chỗ, sẽ cực kỳ hữu dụng.
