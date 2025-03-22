---
title: Context-Aware Request Logger with Async Context for NestJS
author: phongthien99
date: 2025-03-22 10:17:00 +0800
categories: [Nestjs]
tags: [nodejs]
math: true
media_subpath: '/posts/20180809'
---
# Context-Aware Request Logger with Async Context for NestJS

## 1. The Problem

Một hệ thống logging tốt không chỉ ghi nhận sự kiện mà còn cung cấp đủ thông tin để phân tích và truy vết lỗi. Trong NestJS, việc sử dụng **context-aware logger** giúp gắn kết dữ liệu như `requestId`, `userAgent` hoặc thông tin người dùng vào mỗi log, đảm bảo khả năng theo dõi và nhóm các request cụ thể một cách hiệu quả.

Theo nguyên tắc **12-Factor App**, logging đóng vai trò quan trọng trong giám sát và duy trì ứng dụng. Cụ thể:

- **Ứng dụng nên ghi log ở dạng văn bản đơn giản (`stdout` / `stderr`).**
- **Không lưu trữ log trong ứng dụng, mà để môi trường vận hành xử lý.**
- **Mỗi log cần có đủ ngữ cảnh để dễ dàng truy vết.**

## 2. The Solution

Sử dụng `async_hooks` để gán ngữ cảnh .`async_hooks` là một module trong Node.js giúp theo dõi vòng đời của các tác vụ bất đồng bộ như **Promise, setTimeout, HTTP request**. Một trong những ứng dụng quan trọng của nó là quản lý **ngữ cảnh request**, đặc biệt hữu ích khi logging hoặc theo dõi truy vết lỗi.

Trong NestJS, có thể sử dụng`async_hooks` thông qua `AsyncLocalStorage` để gán và duy trì dữ liệu theo từng request mà không cần truyền thủ công. Điều này giúp:

- **Tự động gán `requestId` vào logger**, hỗ trợ truy vết request dễ dàng.
- **Lưu thông tin người dùng trong vòng đời request**, không cần global state.
- **Duy trì trace ID để debugging hệ thống phân tán**.

Cách triển khai phổ biến là dùng **Middleware** để khởi tạo ngữ cảnh request, kết hợp với **Custom Logger** để tự động lấy thông tin từ `AsyncLocalStorage`.

## 3. Implement

Trước tiên, hãy cài đặt NestJS và các thư viện cần thiết:

```
npm install @nestjs/common @nestjs/core @nestjs/platform-express uuid
```

### 1. **LoggerStorageService** (Lưu trữ ngữ cảnh logging)

```
ts
CopyEdit
import { Injectable } from '@nestjs/common';
import { AsyncLocalStorage } from 'async_hooks';

@Injectable()
export class LoggerStorageService {
  private readonly _storage: AsyncLocalStorage<Record<string, any>>;

  constructor() {
    this._storage = new AsyncLocalStorage<Record<string, any>>();
  }

  getStore(): Record<string, any> | undefined {
    return this._storage.getStore();
  }

  run(store: Record<string, any>, callback: () => any) {
    return this._storage.run(store, callback);
  }
}

```

- `AsyncLocalStorage` giúp lưu trữ ngữ cảnh của mỗi request một cách riêng biệt.
- `getStore()` lấy dữ liệu hiện tại trong ngữ cảnh bất đồng bộ.
- `run(store, callback)` thiết lập một ngữ cảnh mới và thực thi callback bên trong nó.

---

### 2. **GestLogger** (Logger có ngữ cảnh)

```
import { Injectable, Logger, LoggerService } from '@nestjs/common';
import { LoggerStorageService } from './storage.service';

export class GestLogger implements LoggerService {
  private readonly _logger: Logger;

  constructor(
    private readonly _storageService: LoggerStorageService,
    context?: string,
    options?: { timestamp?: boolean },
  ) {
    this._logger = new Logger(context, options);
  }

  contextDetail(...fields: string[]): string {
    const context = this._storageService.getStore();
    if (!context) return '';

    // If fields are specified, only include those fields
    if (fields && fields.length > 0) {
      return fields
        .filter((field) => field in context)
        .map((field) => `[${field}:${context[field]}]`)
        .join('');
    }

    // Otherwise include all fields
    return Object.entries(context)
      .map(([key, value]) => `[${key}:${value}]`)
      .join('');
  }

  log(message: any, ...optionalParams: any[]): void {
    this._logger.log(message, ...optionalParams);
  }

  error(message: any, stack?: string, ...optionalParams: any[]): void {
    this._logger.error(message, ...optionalParams);
  }

  warn(message: any, ...optionalParams: any[]): void {
    this._logger.warn(message, ...optionalParams);
  }

  debug(message: any, ...optionalParams: any[]): void {
    this._logger.debug(message, ...optionalParams);
  }

  verbose(message: any, ...optionalParams: any[]): void {
    this._logger.verbose(message, ...optionalParams);
  }
}

@Injectable()
export class GestLoggerFactory {
  constructor(private readonly _storageService: LoggerStorageService) {}
  create(context?: string, options?: { timestamp?: boolean }): GestLogger {
    return new GestLogger(this._storageService, context, options);
  }
}

```

- **`contextDetail()`**: Lấy thông tin từ `AsyncLocalStorage` và định dạng dưới dạng `[key:value]`.

---

### 3. **GestLoggerFactory** (Factory để tạo logger)

```

@Injectable()
export class GestLoggerFactory {
  constructor(private readonly _storageService: LoggerStorageService) {}

  create(context?: string, options?: { timestamp?: boolean }): GestLogger {
    return new GestLogger(this._storageService, context, options);
  }
}

```

- Factory Pattern giúp tạo các instance của `GestLogger`.

---

### 4. **GestLoggerModule** (Module Logging)

```

import {
  Global,
  MiddlewareConsumer,
  Module,
  DynamicModule,
} from '@nestjs/common';
import { GestLoggerFactory } from './logger.service';
import { LoggerStorageService } from './storage.service';
import { HttpContextMiddlewareFactory } from '../middleware/http-context.middleware';
import { v4 as uuidv4 } from 'uuid';

export interface LoggerModuleAsyncOptions {
  httpMiddleware?: {
    enabled?: boolean;
    getContext?: (req: any, res: any) => any;
  };
}

@Global()
@Module({})
export class GestLoggerModule {
  private static _options: LoggerModuleAsyncOptions;

  static forRoot(options: LoggerModuleAsyncOptions): DynamicModule {
    this._options = options;
    return {
      module: GestLoggerModule,
      providers: [GestLoggerFactory, LoggerStorageService],
      exports: [GestLoggerFactory, LoggerStorageService],
    };
  }

  configure(consumer: MiddlewareConsumer) {
    const enabled = GestLoggerModule._options?.httpMiddleware?.enabled ?? true;
    if (!enabled) return;

    const getContext =
      GestLoggerModule._options?.httpMiddleware?.getContext ||
      ((req, res) => ({
        requestId: req.headers['x-request-id'] || uuidv4(),
        userAgent: req.headers['user-agent'] || null,
      }));

    consumer.apply(HttpContextMiddlewareFactory(getContext)).forRoutes('*');
  }
}

```

- `forRoot()` cấu hình module với middleware tùy chỉnh.
- `configure()` tự động thêm `requestId` vào mỗi request.

---

### 5. **Sử dụng trong `AppModule`**

```
ts
CopyEdit
import { Module } from '@nestjs/common';
import { GestLoggerModule } from './logger/logger.module';
import { AppService } from './app.service';

@Module({
  imports: [
    GestLoggerModule.forRoot({
      httpMiddleware: {
        enabled: true,
        getContext: (req, res) => ({
          requestId: req.headers['x-request-id'] || uuidv4(),
          userAgent: req.headers['user-agent'] || null,
        }),
      },
    }),
  ],
  providers: [AppService],
})
export class AppModule {}

```

- Import `GestLoggerModule` vào ứng dụng chính.
- Kích hoạt middleware tự động thêm `requestId`.

---

### 6. **Sử dụng trong `AppService`**

```

import { Injectable } from '@nestjs/common';
import { GestLoggerFactory } from './logger/logger.service';
import { GestLogger } from './logger/gest-logger';

@Injectable()
export class AppService {
  private _logger: GestLogger;
  constructor(private readonly gestLoggerFactory: GestLoggerFactory) {
    this._logger = gestLoggerFactory.create(AppService.name);
  }

  getHello(): string {
    this._logger.log(`${this._logger.contextDetail('requestId')} Hello World!`);
    return 'Hello World!';
  }
}

```

- `this._logger.log()` sẽ tự động kèm theo `requestId` từ `AsyncLocalStorage`.

**Kết quả:**

```jsx
[Nest] 800395  - 22/03/2025, 03:28:44     LOG [AppService] [requestId:c003fca7-4d48-4ce3-ae3d-979a693eaf0d] Hello World!

```

## 4. The conclusion

Với việc sử dụng `async_hooks` và `AsyncLocalStorage`, chúng ta có thể duy trì ngữ cảnh logging xuyên suốt các request trong NestJS. Điều này giúp việc debug trở nên dễ dàng hơn và tăng cường khả năng theo dõi request trong hệ thống.