---
title: Combining Rclone and Temporal to Ensure High Availability and Avoid Data Duplication"
author: phongthien99
date: 2024-09-07 00:40:00 +0800
categories: [Backup]
tags: [temporal,rclone]
math: true
media_subpath: '/posts/20240503'
---
# Building an Efficient Data Synchronization System: Combining Rclone and Temporal to Ensure High Availability and Avoid Data Duplication

### Đặt Vấn Đề

Trong một hệ thống cần đảm bảo tính sẵn sàng cao (High Availability - HA), bạn có ba máy chủ hoạt động trong một cụm. Các máy chủ này cùng chia sẻ một volume lưu trữ dữ liệu chung. Yêu cầu của hệ thống là phải đồng bộ dữ liệu từ volume chia sẻ này lên nhiều hệ lưu trữ khác nhau như AWS S3, Google Cloud Storage (GCS), và FTP.

Tuy nhiên, việc đảm bảo đồng bộ dữ liệu lên các hệ lưu trữ này mà vẫn giữ được tính HA và tránh trùng lặp dữ liệu lại đặt ra một thách thức không nhỏ.

1. **Triển khai cron job trên một máy chủ duy nhất**: Nếu bạn chọn một máy chủ duy nhất để thực hiện đồng bộ hóa bằng cách sử dụng cron job, hệ thống sẽ gặp rủi ro nếu máy chủ này gặp sự cố. Trong trường hợp đó, việc đồng bộ dữ liệu sẽ bị gián đoạn, dẫn đến khả năng mất mát dữ liệu.
2. **Triển khai cron job trên nhiều máy chủ**: Để đảm bảo tính HA, nếu bạn triển khai cron job trên cả ba máy chủ, có khả năng cả ba máy chủ sẽ thực hiện đồng bộ cùng một lúc. Điều này dẫn đến việc dữ liệu bị đồng bộ nhiều lần lên các hệ lưu trữ khác nhau, gây ra sự trùng lặp dữ liệu. Hệ quả là lãng phí băng thông và làm phức tạp hóa việc quản lý dữ liệu trên các hệ lưu trữ.

Như vậy, hệ thống cần một giải pháp có thể vừa đảm bảo đồng bộ dữ liệu lên các hệ lưu trữ một cách nhất quán, vừa tránh được việc dữ liệu bị trùng lặp, đồng thời vẫn giữ được tính sẵn sàng cao trong trường hợp có sự cố xảy ra

## Giải pháp

Để giải quyết vấn đề đảm bảo tính sẵn sàng cao (HA) và tránh trùng lặp dữ liệu khi đồng bộ từ 3 máy chủ chia sẻ volume lên các hệ lưu trữ như AWS S3, Google Cloud Storage (GCS), và FTP, chúng ta có thể kết hợp Rclone với Temporal. Đây là giải pháp không chỉ tối ưu hóa quá trình đồng bộ mà còn giúp quản lý hiệu quả và tự động hóa quy trình..

### 1. **Rclone: Công Cụ Đồng Bộ Dữ Liệu Linh Hoạt**

Rclone là một công cụ mạnh mẽ để đồng bộ hóa dữ liệu giữa nhiều hệ lưu trữ đám mây và máy chủ cục bộ. Rclone hỗ trợ nhiều giao thức và dịch vụ lưu trữ khác nhau như AWS S3, Google Cloud Storage, FTP, và nhiều dịch vụ khác. Đặc biệt, nó hỗ trợ các tính năng như:

- **Đồng bộ hóa dữ liệu từ volume chia sẻ lên nhiều hệ lưu trữ**.
- **Kiểm tra sự khác biệt của dữ liệu** trước khi đồng bộ để tránh tải lên các dữ liệu không cần thiết.
- **Hỗ trợ nén và mã hóa dữ liệu** khi cần thiết.

### 2. **Temporal: Hệ Thống Quản Lý Workflow Đảm Bảo HA**

Temporal là một hệ thống quản lý workflow giúp bạn điều phối các quy trình phức tạp trong môi trường phân tán. Với Temporal, bạn có thể đảm bảo rằng các tác vụ đồng bộ dữ liệu chỉ được thực hiện bởi một máy chủ tại một thời điểm nhất định, và khi máy chủ đó gặp sự cố, một máy chủ khác sẽ tự động tiếp quản quá trình đồng bộ. Temporal giúp giải quyết các vấn đề liên quan đến tính HA và tránh trùng lặp dữ liệu bằng cách:

- **Quản lý trạng thái**: Temporal giữ trạng thái của các workflow và đảm bảo chỉ có một instance của workflow đang chạy tại một thời điểm.
- **Failover tự động**: Nếu máy chủ đang thực hiện workflow gặp sự cố, Temporal sẽ tự động chuyển trách nhiệm sang một máy chủ khác.

## Thực hiện

### Bước 1: Thiết lập cơ sở dữ liệu SQL chung

- Cài đặt MySQL/PostgreSQL.
- Tạo bảng Temporal và cấu hình Temporal server kết nối đến cơ sở dữ liệu này.

### Bước 2: Cài đặt Temporal trên 3 server

- Tải Temporal server trên mỗi server.
- Cấu hình Temporal để kết nối đến cơ sở dữ liệu SQL chung.

### Bước 3: Cài đặt và cấu hình Worker

- Cài đặt rclone và worker Temporal trên mỗi server.
- Cấu hình worker để sử dụng rclone đồng bộ dữ liệu lên S3.

### Bước 4: Quản lý đồng bộ và HA

- Sử dụng Temporal workflow để quản lý tiến trình đồng bộ, chỉ cho phép một server đồng bộ tại một thời điểm.
- Đảm bảo tính HA, nếu một server gặp sự cố, các server khác tiếp quản tiến trình đồng bộ.

### Bước 5: Chạy và kiểm tra

- Chạy Temporal server và worker trên cả 3 server.
- Kiểm tra tính đồng bộ và HA của hệ thống.

![image.png](./2024-09-06.png)

## Kết Luận

Trong việc đảm bảo tính sẵn sàng cao (HA) và tránh trùng lặp dữ liệu khi đồng bộ từ một volume chia sẻ lên nhiều hệ lưu trữ như AWS S3, Google Cloud Storage (GCS), và FTP, việc sử dụng Rclone kết hợp với Temporal mang lại giải pháp tối ưu và hiệu quả. Ngoài ra chúng ta có thể thử nghiệm trên 1 máy duy nhất bằng cách sử dựng temporal kết hợp với sqlite3.