---
title: Efficient Configuration Management in NestJS
author: phongthien99
date: 2025-03-15 10:17:00 +0800
categories: [Nestjs]
tags: [nodejs]
math: true
media_subpath: '/posts/20180809'
---
# Efficient Configuration Management in NestJS

## **1. Problem**

Trong NestJS, nếu bạn đang sử dụng cấu hình theo môi trường nhưng lại load vào một biến global và import khắp nơi, thì sẽ gặp các vấn đề sau:

- **Khó kiểm soát và quản lý**: Biến global có thể bị thay đổi hoặc được sử dụng không nhất quán trong nhiều module.
- **Khó tái sử dụng module**: Nếu module phụ thuộc trực tiếp vào biến global, việc tái sử dụng module đó ở nơi khác sẽ khó khăn hoặc không thể thay đổi config một cách linh hoạt.
- **Khó kiểm thử (testability kém)**: Khi config được load trực tiếp từ biến global, việc mock dữ liệu trong unit test trở nên phức tạp, ảnh hưởng đến khả năng kiểm thử và bảo trì code.

Để giải quyết những vấn đề trên, chúng ta có thể sử dụng mô hình **forRoot** và **forRootAsync** để quản lý config một cách linh hoạt và dễ kiểm soát hơn.

---

## **2. Solution**

### **Bước 1: Khởi tạo AppModule theo dạng `forRoot`**

Thay vì sử dụng biến global, ta có thể truyền config qua `forRoot`, giúp module có thể nhận cấu hình từ bên ngoài.

### **app.module.ts**

```tsx
import { DynamicModule, Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';

@Module({})
export class AppModule {
  static forRoot(config: Record<string, any>): DynamicModule {
    return {
      module: AppModule,
      providers: [],
      controllers: [],
    };
  }
}

```

### **main.ts**

```tsx
const loadConfig = () => {
  return {
    http: {
      port: 5000,
    },
  };
};

const conf = loadConfig();

async function bootstrap(config: Record<string, any>) {
  const app = await NestFactory.create(AppModule.forRoot(config));
  await app.listen(config.http.port, () => {
    console.log(`Listening on port ${config.http.port}`);
  });
}
bootstrap(conf);

```

Cách làm này giúp chúng ta quản lý cấu hình tập trung và dễ dàng thay đổi.

---

### **Bước 2: Sử dụng `ConfigModule` của NestJS**

NestJS có sẵn `@nestjs/config`, giúp quản lý cấu hình theo môi trường dễ dàng hơn.

### **app.module.ts** (cập nhật thêm `ConfigModule`)

```tsx
import { DynamicModule, Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';

@Module({})
export class AppModule {
  static forRoot(config: Record<string, any>): DynamicModule {
    return {
      module: AppModule,
      imports: [
        ConfigModule.forRoot({
          isGlobal: true, // Đảm bảo module có thể được sử dụng ở bất kỳ đâu
          load: [() => config], // Load config từ biến config
        }),
      ],
    };
  }
}

```

Việc sử dụng `ConfigModule` giúp dễ dàng inject config vào bất kỳ module nào trong ứng dụng mà không cần dùng biến global.

---

### **Bước 3: Sử dụng `forRootAsync` để inject config vào từng module**

Nếu một module cần sử dụng biến môi trường, ta nên inject thông qua dependency injection thay vì sử dụng biến global trực tiếp.

### **some.module.ts**

```tsx
import { DynamicModule, Module, Provider } from '@nestjs/common';

@Module({})
export class SomeModule {
  static forRootAsync(...configProviders: Provider[]): DynamicModule {
    return {
      module: SomeModule,
      providers: [...configProviders],
    };
  }
}

```

Bằng cách này, chúng ta có thể truyền config vào từng module một cách linh hoạt và kiểm soát tốt hơn.

### **Cập nhật `app.module.ts`**

```tsx
import { ConfigModule, ConfigService } from '@nestjs/config';
import { SomeModule } from './some.module';

@Module({})
export class AppModule {
  static forRoot(config: Record<string, any>): DynamicModule {
    return {
      module: AppModule,
      imports: [
        ConfigModule.forRoot({
          isGlobal: true, // Đảm bảo module có thể được sử dụng ở bất kỳ đâu
          load: [() => config], // Load config từ biến config
        }),
        SomeModule.forRootAsync({
          provide: 'DATABASE_HOST',
          useFactory: (configService: ConfigService) =>
            configService.get<string>('mongo.host'),
          inject: [ConfigService],
        }),
      ],
    };
  }
}

```

Sử dụng `forRootAsync` giúp module tự động lấy được config phù hợp mà không phụ thuộc vào biến global, làm tăng tính linh hoạt và dễ bảo trì.

---

## **3. Conclusion**

Việc quản lý cấu hình trong NestJS một cách khoa học sẽ giúp ứng dụng dễ mở rộng, bảo trì và kiểm thử hơn. Dưới đây là một số điểm quan trọng:

1. **Tránh sử dụng biến global** để lưu trữ config, vì nó gây khó khăn trong kiểm soát và tái sử dụng module.
2. **Sử dụng `forRoot` hoặc `forRootAsync`** để truyền config vào module thay vì hardcode giá trị.
3. **Tận dụng `ConfigModule` của NestJS** để quản lý biến môi trường một cách chuyên nghiệp.
4. **Sử dụng `forRootAsync` để inject config vào từng module** giúp module linh hoạt và dễ dàng kiểm thử hơn.

Bằng cách làm theo phương pháp trên, bạn sẽ có một hệ thống quản lý config tốt hơn, giúp ứng dụng của bạn trở nên bền vững và dễ dàng phát triển về sau.