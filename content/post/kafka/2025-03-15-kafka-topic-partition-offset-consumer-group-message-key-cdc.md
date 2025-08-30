---
title: Kafka Topic, Partition, Offset, Consumer Group, and Message Key in Change Data Capture (CDC)
author: phongthien99
date: 2025-08-30 10:17:00 +0800
categories: [queue]
tags: [queue, kafka, cdc]
math: true
media_subpath: '/posts/20180809'
---
## 1. Đặt vấn đề

Trong các hệ thống phân tán hiện đại, **Kafka** thường được sử dụng như một nền tảng trung gian để xử lý dữ liệu theo thời gian thực. Khi triển khai **CDC (Change Data Capture)** nhằm đồng bộ dữ liệu giữa hai cơ sở dữ liệu, việc hiểu rõ các khái niệm **Topic, Partition, Offset, Consumer Group và Key Message** là cực kỳ quan trọng. Những thành phần này quyết định đến hiệu năng, khả năng mở rộng và tính toàn vẹn dữ liệu.


## 2. Định nghĩa các khái niệm quan trọng

### 2.1 Kafka Topic

- Là **danh mục (channel)** nơi producer gửi message và consumer đọc message.
- Mỗi topic được chia nhỏ thành nhiều **partition** để hỗ trợ song song (parallelism).



### 2.2 Partition

- Mỗi partition là một **log file append-only** được lưu trên broker.
- Partition giúp Kafka scale: nhiều consumer có thể xử lý song song dữ liệu từ nhiều partition khác nhau.
- **Message trong 1 partition được đảm bảo thứ tự (ordering).**
- Tổng throughput của topic = tổng throughput của các partition.



### 2.3 Offset

- Offset là **chỉ số tuần tự** gắn với mỗi message trong một partition.
- Ví dụ: Partition P0 có message với offset 0,1,2…
- Consumer khi đọc sẽ ghi nhớ offset để biết mình đã xử lý đến đâu.



### 2.4 Consumer Group

- Consumer group = tập hợp nhiều consumer cùng đọc 1 topic.
- Kafka phân chia mỗi partition chỉ cho **1 consumer duy nhất trong group**.
- Nếu số consumer = số partition → mỗi consumer đọc 1 partition.
- Nếu số consumer > số partition → có consumer sẽ idle (ngồi chơi).


### 2.5 Key trong message Kafka

- Key là giá trị tùy chọn gắn với mỗi message.
- Kafka dùng **key** để quyết định message sẽ vào partition nào.
- **Cùng key → luôn cùng partition** → đảm bảo thứ tự.
- Key quan trọng khi cần grouping theo entity (vd: user_id, order_id).



## 3. Case Study: CDC trong hệ thống Order Service của E-commerce

Giả sử một hệ thống **Order Service** trong E-commerce cần đồng bộ dữ liệu đơn hàng từ Database A sang Database B qua Kafka:

- Khi đơn hàng được tạo/cập nhật/xóa, **Debezium** sẽ capture sự kiện và đẩy vào Kafka Topic `orders`.
- Key message = `order_id` → đảm bảo toàn bộ sự kiện liên quan đến cùng một đơn hàng đều nằm trong cùng một partition.
- Consumer Group “DB-Sync” (chạy ở Database B) sẽ nhận dữ liệu và áp dụng thay đổi tương ứng.

**Lợi ích:**

- Đảm bảo **tính toàn vẹn** khi đồng bộ dữ liệu.
- Hỗ trợ **scale-out** bằng cách tăng số partition và số consumer group.
- Vẫn giữ được **thứ tự update** cho cùng một bản ghi.



## 4. Biểu đồ minh họa

### Quan hệ Topic - Partition - Consumer Group

```
@startuml
title Quan hệ Topic - Partition - Consumer Group

skinparam componentStyle rectangle
skinparam shadowing false
skinparam rectangle {
  BorderColor #555
  RoundCorner 12
}

rectangle "Topic: customers" as Topic {
  rectangle "Partition P0\nOffsets: 0..n" as P0
  rectangle "Partition P1\nOffsets: 0..n" as P1
  rectangle "Partition P2\nOffsets: 0..n" as P2
}

package "Consumer Group A" as CGA {
  rectangle "Consumer A1" as A1
  rectangle "Consumer A2" as A2
}

package "Consumer Group B" as CGB {
  rectangle "Consumer B1" as B1
  rectangle "Consumer B2" as B2
  rectangle "Consumer B3" as B3
}

P0 -right-> A1 : read
P1 -right-> A2 : read
P2 -right-> A2 : read

P0 -right-> B1 : read
P1 -right-> B2 : read
P2 -right-> B3 : read

note bottom of Topic
  - Message có **Key** (thường = Primary Key/Unique Key)
  - Partition = hash(Key) % numPartitions
  - Ordering chỉ được đảm bảo **trong 1 partition**
end note

note right of A2
  Điều kiện:
  - Mỗi partition chỉ có 1 consumer **trong cùng group**
  - Consumer > Partition → consumer thừa không đọc
  - Consumer < Partition → consumer có thể đọc nhiều partition
end note
@enduml

```

### Luồng message CDC với Key và Offset

```
@startuml
title Luồng CDC: Producer -> Topic/Partition -> Consumer (với Key & Offset)

actor App as Producer
participant "Topic: customers\nPartition P1" as P1
participant "Consumer A2\n(Group A)" as A2

== Publish ==
Producer -> P1 : Produce message\nKey = customer_id=123\nValue = {... after ...}
activate P1
P1 -> P1 : Ghi log append-only\nAssign Offset = 1024
deactivate P1

note over P1
  Ví dụ:
  - Key: 123  -> hash(Key) -> P1
  - Offset tăng dần: ...1023, **1024**, 1025...
end note

== Consume ==
P1 -> A2 : Deliver record @Offset=1024
activate A2
A2 -> A2 : Xử lý & commit offset=1024
deactivate A2

== Tiếp theo ==
Producer -> P1 : Key=123 (update tiếp)\n-> vẫn vào P1
activate P1
P1 -> P1 : Offset=1025
deactivate P1
P1 -> A2 : Deliver @Offset=1025
A2 -> A2 : Commit offset=1025

note right of A2
  Consumer nhớ offset (commit)
  -> restart vẫn tiếp tục từ 1026
end note
@enduml

```



## 5. Kết luận

Trong CDC với Kafka:

- **Key message** đóng vai trò then chốt trong việc đảm bảo dữ liệu cùng bản ghi được xử lý tuần tự.
- **Partition** quyết định khả năng song song.
- **Consumer Group** đảm bảo tính cân bằng tải và mở rộng quy mô.
- **Offset** là cơ chế quan trọng giúp consumer có thể tiếp tục đọc từ đúng vị trí khi xảy ra restart.
- **Điều kiện quan trọng**: số lượng consumer trong group không nên vượt quá số partition, và key cần được chọn đúng (primary/unique key) để đảm bảo tính toàn vẹn.

Việc thiết kế đúng số lượng partition, cơ chế consumer group, và key trong message là nền tảng cho một hệ thống CDC đáng tin cậy và hiệu quả.