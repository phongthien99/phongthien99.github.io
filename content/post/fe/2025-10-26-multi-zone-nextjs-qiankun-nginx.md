---
title: "Multi-Zone in Modern Web Development: Solution with Next.js, Qiankun, and Nginx"
author: phongthien99
date: 2025-10-26 23:09:00 +0700
categories: [Fe]
tags: [nginx,fe,nextjs,qiankun,micro-frontend]
math: true
media_subpath: '/posts/20251026'

---
### Đặt vấn đề

Trong phát triển ứng dụng web hiện đại, các hệ thống phức tạp thường được chia thành nhiều module độc lập. Tuy nhiên, triển khai toàn bộ ứng dụng dưới dạng **mono frontend** gây khó khăn trong việc mở rộng, bảo trì và tối ưu hiệu năng. Các thay đổi nhỏ trong một module có thể ảnh hưởng đến toàn bộ hệ thống, đồng thời việc tải tất cả module cho mọi người dùng dẫn đến lãng phí tài nguyên.

**Multi-Zone** ra đời nhằm giải quyết các vấn đề này bằng cách tách ứng dụng thành các **khu vực (zone) độc lập**, cho phép triển khai, cập nhật và vận hành từng zone mà không ảnh hưởng đến các phần còn lại. Giải pháp này giúp tối ưu trải nghiệm người dùng, tăng khả năng mở rộng, phân tách trách nhiệm rõ ràng và hỗ trợ phát triển, vận hành độc lập giữa các nhóm.


### Giải pháp và lựa chọn công nghệ

Để đáp ứng yêu cầu xây dựng ứng dụng **multi-zone**, team lựa chọn **Next.js** làm framework chính nhờ tính phổ biến, cộng đồng lớn và hỗ trợ tốt cho nhiều mô hình triển khai. Next.js cung cấp khả năng **SSR (Server-Side Rendering)** và **multi-zone**, giúp tối ưu hiệu năng và quản lý routing linh hoạt.

Tuy nhiên, với yêu cầu **triển khai động dựa trên cấu hình mà không cần build lại** và có thể sử dụng **Nginx hoặc server tĩnh** để phục vụ ứng dụng, việc chỉ dùng Next.js sẽ gặp hạn chế vì multi-zone của Next.js mặc định gắn với SSR.

Để giải quyết, team quyết định tích hợp thêm **Qiankun**, một framework **micro frontend**, giúp:

- Triển khai các ứng dụng con **độc lập**, có thể load dynamic theo cấu hình runtime.
- Cho phép **tách biệt zone**, triển khai trên server tĩnh hoặc Nginx mà không cần build lại toàn bộ ứng dụng.
- Dễ dàng quản lý **routing và chia sẻ thư viện** giữa các zone, đồng thời giữ được khả năng mở rộng và bảo trì.

---

### Giải pháp Next.js + Qiankun + Nginx + Dynamic Config

Để tăng tính linh hoạt, hệ thống kết hợp **dynamic config** thông qua file `env.js`:

- File **env.js** chứa thông tin runtime của các micro app như tên, URL và entry point, cho phép **shell app load các micro app theo cấu hình mà không cần rebuild**.
- **Shell App (Next.js)** đóng vai trò khung chính, quản lý layout, routing và có thể sử dụng SSR/SSG.
- **Micro App** triển khai độc lập, mỗi micro app là một zone trong kiến trúc multi-zone.
- **Qiankun** chịu trách nhiệm load các micro app runtime, quản lý sandbox, shared library và isolation giữa các micro app.
- **Nginx** phục vụ static files của shell app và micro app, đồng thời cung cấp cơ chế **dynamic mapping** thông qua file config để triển khai linh hoạt theo môi trường.

Giải pháp này đảm bảo:

- **Dynamic deployment**: Thêm, xóa, hoặc cập nhật micro app mà không cần rebuild shell app.
- **Độc lập triển khai**: Micro app build và deploy riêng, tách biệt với shell app.
- **Tối ưu resource và hiệu năng**: Nginx phục vụ static files, giảm tải server Node.
- **Dễ quản lý cấu hình runtime**: Cho phép triển khai khác nhau theo môi trường (dev, staging, prod) bằng các file config riêng.

---

### Kết luận

Kết hợp **Next.js**, **Qiankun** và **Nginx** với cơ chế **dynamic config** mang lại giải pháp linh hoạt cho kiến trúc **multi-zone**.

- Next.js cung cấp nền tảng mạnh mẽ với SSR/SSG, routing và quản lý layout.
- Qiankun cho phép triển khai **micro frontend ở cấp ứng dụng/zone**, load dynamic theo runtime, tách biệt các module và chia sẻ thư viện.
- Nginx đảm bảo triển khai **static, nhẹ nhàng và linh hoạt**, kết hợp với file `env.js` để load cấu hình runtime mà không cần build lại shell app.

Giải pháp này giúp team phát triển độc lập, dễ bảo trì, tối ưu trải nghiệm người dùng và quản lý cấu hình runtime hiệu quả. Đây là hướng đi phù hợp cho các dự án lớn, cần mở rộng theo từng module và triển khai linh hoạt trên nhiều môi trường, đồng thời giảm thiểu rủi ro khi cập nhật hoặc thay đổi từng phần của hệ thống.