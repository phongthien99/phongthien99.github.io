---
title: Implementing i18n in NestJS Using In-Memory Storage
author: phongthien99
date: 2025-04-05 10:17:00 +0800
categories: [Nestjs]
tags: [nodejs]
math: true
media_subpath: '/posts/20180809'
---
# Implementing i18n in NestJS Using In-Memory Storage

## 1.  Giới thiệu

Quốc tế hóa (i18n) là một phần quan trọng trong việc phát triển ứng dụng đa ngôn ngữ. Trong bài viết này, chúng ta sẽ tìm hiểu cách triển khai i18n trong NestJS sử dụng in-memory storage thay vì tải dữ liệu từ file JSON hoặc database.

## 2. Cài đặt NestJS  và các thư viện cần thiết

Trước tiên, bạn cần có một dự án NestJS. Nếu chưa có, bạn có thể tạo mới bằng lệnh:

```bash
npx @nestjs/cli new my-nestjs-app
cd my-nestjs-app

```

Cài đặt thư viện `nestjs-i18n` để hỗ trợ i18n:

```bash
npm install --save @nestjs/i18n

```

## 3. Cấu hình i18n với In-Memory Storage

### 3.1. Tạo I18nMemoryLoader

Chúng ta sẽ tạo một loader để lưu trữ bản dịch trong bộ nhớ:

```tsx
import { Inject } from '@nestjs/common';
import { I18N_LOADER_OPTIONS, I18nLoader, I18nTranslation } from 'nestjs-i18n';

export class I18nMemoryLoader extends I18nLoader {
  private readonly _translations: I18nTranslation;
  private readonly _languages: string[];

  constructor(
    @Inject(I18N_LOADER_OPTIONS)
    private options: any,
  ) {
    super();
    const mergeI18n = (...objects) => {
      return objects.reduce((acc, obj) => {
        Object.keys(obj).forEach((lang) => {
          acc[lang] = { ...(acc[lang] || {}), ...obj[lang] };
        });
        return acc;
      }, {});
    };

    this._translations = mergeI18n(...options.translations);

    this._languages = Object.keys(this._translations);
  }

  async load(): Promise<I18nTranslation> {
    return this._translations;
  }

  async languages(): Promise<string[]> {
    return this._languages;
  }
}

```

### 3.2. Tạo I18nMemoryModule

Chúng ta sẽ tạo một module để sử dụng loader này và hỗ trợ thêm bản dịch từ từng module riêng:

```tsx
import { DynamicModule, Module } from '@nestjs/common';
import { I18nAsyncOptions, I18nModule, I18nTranslation } from 'nestjs-i18n';
import { I18nMemoryLoader } from './i18n-memory';

const I18nTranslationFactory = 'I18nTranslationFactory';

@Module({})
export class I18nMemoryModule {
  private static translations: I18nTranslation[] = [];

  static forRootAsync(options: I18nAsyncOptions): DynamicModule {
    options.inject = [I18nTranslationFactory];
    options.loader = I18nMemoryLoader;
    return {
      module: I18nMemoryModule,
      imports: [I18nModule.forRootAsync(options)],
    };
  }

  static forFeature(translation: I18nTranslation): DynamicModule {
    I18nMemoryModule.translations.push(translation);

    return {
      module: I18nMemoryModule,
      global: true,
      providers: [
        {
          provide: I18nTranslationFactory,
          useFactory: () => I18nMemoryModule.translations,
        },
      ],
      exports: [I18nTranslationFactory],
    };
  }
}

```

### 3.3. Đăng ký module trong AppModule

Thêm `I18nMemoryModule` vào module chính:

```tsx
import { Module } from '@nestjs/common';
import { I18nMemoryModule } from './i18n-memory.module';
import { QueryResolver, AcceptLanguageResolver } from 'nestjs-i18n';

@Module({
  imports: [
    I18nMemoryModule.forRootAsync({
      useFactory: (translationsFactory: I18nTranslation[]) => ({
        fallbackLanguage: 'en',
        loaderOptions: {
          translations: translationsFactory,
        },
      }),
      resolvers: [
        { use: QueryResolver, options: ['lang'] },
        AcceptLanguageResolver,
      ],
    }),
  ],
})
export class AppModule {}

```

### 3.4. Thêm bản dịch từ mỗi module riêng

Mỗi module có thể sử dụng `forFeature` để thêm bản dịch riêng của nó:

```tsx
import { Module } from '@nestjs/common';
import { I18nMemoryModule } from '../i18n-memory.module';

const userTranslations = {
  en: { user: 'User' },
  vi: { user: 'Người dùng' },
};

@Module({
  imports: [I18nMemoryModule.forFeature(userTranslations)],
})
export class UserModule {}

```

### 3.5. Sử dụng i18n trong controller

Tạo một controller để sử dụng i18n:

```tsx
import { Controller, Get, Headers } from '@nestjs/common';
import { I18nService } from 'nestjs-i18n';

@Controller('i18n')
export class I18nController {
  constructor() {}

  @Get('translate')
  translate(@I18n() i18n: I18nContext): string {
  return await i18n.t('user');
  }
}

```

## 4. Kiểm tra API

Chạy ứng dụng bằng lệnh:

```bash
npm run start

```

Gọi API để kiểm tra:

```bash
curl -H "Accept-Language: vi-VN" "http://localhost:3000/i18n/translate"

```

Kết quả:

```json
"Người dùng"

```

## 5. Kết luận

Chúng ta đã triển khai thành công i18n trong NestJS sử dụng in-memory storage. Ngoài ra, chúng ta có thể thêm bản dịch từ từng module riêng bằng `forFeature`, giúp hệ thống linh hoạt hơn mà không cần tải lại toàn bộ ứng dụng.