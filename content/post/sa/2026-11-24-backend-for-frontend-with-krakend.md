---
title: "Backend-for-Frontend (BFF) Pattern"
date: "2026-11-23T00:00:00Z"
draft: false
categories: [SA]
tags: [bff, krakend, api-gateway, microservices, software-architecture]
slug: "backend-for-frontend-with-krakend"
author: "phongthien"
---

# Backend-for-Frontend (BFF) Pattern

> Một màn hình, ba API, hai loại client và một câu hỏi: có thể xây lớp BFF chỉ bằng file cấu hình không?

Mở trang chủ (Dashboard) của một ứng dụng thương mại điện tử, bạn thấy ba khối thông tin: tên và avatar người dùng, vài đơn hàng gần đây, và một dải sản phẩm gợi ý. Với người dùng, đó là một màn hình. Với frontend (FE), đó là ba lời gọi tới ba microservice. Mỗi service trả về nhiều dữ liệu hơn cần thiết, và FE phải tự ghép chúng lại.

Bài viết này kể lại cách chúng tôi giải quyết vấn đề đó bằng pattern **Backend-for-Frontend (BFF)**. Chúng tôi hiện thực BFF bằng **KrakenD**, một API gateway cấu hình bằng file JSON, rồi ghi lại những gì KrakenD làm tốt và những gì nó không làm được.

---

## 1. Đặt vấn đề

### Bài toán gốc: một backend chung cho mọi client

Nhiều hệ thống bắt đầu với một ứng dụng web và một backend phục vụ nó. Khi có thêm ứng dụng mobile, cách nhanh nhất là cho mobile dùng chung backend đó.

Hai loại client này có nhu cầu khác nhau. Mobile có màn hình nhỏ và mạng không ổn định, nên cần payload gọn và ít request. Web hiển thị nhiều thông tin hơn và đổi giao diện theo lịch riêng. Backend chung phải đáp ứng cả hai, và dần gặp ba vấn đề:

- **Bị kéo về nhiều hướng.** Mỗi thay đổi cho Web có thể ảnh hưởng đến Mobile, và ngược lại. API có thêm tham số như `?client=mobile` và nhánh `if` cho từng loại client.
- **Trở thành nút cổ chai.** Mọi team FE phải chờ team backend sửa API cho mình. Tài liệu kiến trúc của Microsoft mô tả tình trạng này: backend chung *"encounters competing demands from multiple frontend systems"*. Kết quả là cập nhật liên tục và tắc nghẽn trong phát triển.
- **Tách rời team FE và team backend.** Team FE biết màn hình cần gì nhưng không kiểm soát API. Team backend kiểm soát API nhưng phải phục vụ nhiều team cùng lúc.

Sam Newman, người đặt tên cho pattern BFF, gọi kiểu backend này là *general-purpose API backend*. Nó ôm nhiều trách nhiệm và thành điểm nghẽn mỗi khi cần ra tính năng mới.

### Khi FE gọi thẳng vào microservice

Một số hệ thống chọn hướng ngược lại: bỏ backend chung và cho FE gọi thẳng các microservice. Microservice chia hệ thống theo **nghiệp vụ**: user-service, order-service, product-service. FE lại nhìn hệ thống theo **màn hình**. Khoảng cách giữa hai cách nhìn này gây ra bốn vấn đề.

**Nhiều round-trip cho một màn hình.** Màn hình Dashboard ở phần mở đầu cần 3 request. Trên mạng di động, mỗi request thêm độ trễ kết nối. FE cũng phải xử lý trạng thái loading và lỗi cho từng request.

**Over-fetching.** API `/users/{id}` trả về cả số điện thoại, địa chỉ và ngày tạo tài khoản, trong khi Mobile chỉ cần tên và avatar. API đơn hàng trả về cả danh sách item và địa chỉ giao hàng, trong khi màn hình chỉ hiển thị trạng thái và tổng tiền.

**Lộ dữ liệu nội bộ.** Đây là vấn đề nghiêm trọng nhất. Trong dữ liệu mẫu của chúng tôi, product-service trả về `supplier_cost` (giá nhập), order-service trả về `warehouse_code`, user-service trả về `internal_risk_score`. Các service này được thiết kế cho lời gọi nội bộ. Khi FE gọi thẳng, các field đó đi ra tới trình duyệt.

**Logic ghép dữ liệu nằm ở FE.** FE phải lấy đơn hàng, đọc `customer_id`, rồi gọi user-service. FE cũng phải gom các `product_id` để lấy tên sản phẩm. Web, iOS và Android mỗi bên viết lại logic này theo một cách.

Cả hai hướng thất bại vì cùng một lý do: **không có chỗ nào dành riêng cho nhu cầu của từng loại client**. Backend chung thì quá chung, còn microservice thì quá chi tiết.

---

## 2. Giải pháp: Backend-for-Frontend

### Pattern BFF

Sam Newman mô tả Backend-for-Frontend năm 2015 với một nguyên tắc: **mỗi trải nghiệm người dùng một BFF** (*"one experience, one BFF"*). Trong bài này, mỗi trải nghiệm tương ứng với một loại client: Web hoặc Mobile.

BFF là phần phía server của một loại client. BFF nằm giữa FE và các microservice. Theo Newman, *"The BFF is tightly focused on a single UI, and just that UI"*.

FE gọi một endpoint duy nhất, ví dụ `/web/v1/dashboard`, thay cho ba service. BFF làm những việc **riêng cho loại client đó**:

- Gọi các service phía sau, **song song** khi có thể.
- **Gộp** kết quả thành một response theo đúng cấu trúc màn hình cần.
- **Lọc** bỏ field thừa và field nội bộ.
- **Định dạng riêng** cho từng loại client: BFF Web trả đầy đủ, BFF Mobile trả rút gọn.
- **Chịu lỗi khi gọi service:** timeout, circuit breaker, trả về dữ liệu từng phần khi một service không phản hồi. Newman xếp việc này vào trách nhiệm của BFF. Client cũng phải hiển thị được một response chưa đầy đủ.

Mỗi loại client có một BFF riêng, và **team FE của loại client đó sở hữu BFF**. Newman viết rằng BFF *"will typically be maintained by the same team as the user interface"*. Team FE tự sửa API cho màn hình của mình, nên không phải chờ team backend, và hai team không còn bị tách rời.

{{< archcode title="Backend-for-Frontend - Mỗi loại client một BFF" mode="preview" view="overview" layout="/diagrams/bff-krakend.layout.yaml" >}}
version: 1

system:
  id: bff-krakend
  name: Backend-for-Frontend với KrakenD

groups:
  - id: clients
    name: Client
    type: network
  - id: bff
    name: Lớp BFF — KrakenD (team FE sở hữu)
    type: network
  - id: services
    name: Microservice theo nghiệp vụ
    type: cluster

components:
  - id: web-fe
    name: Web FE
    type: frontend
    group: clients
  - id: mobile-fe
    name: Mobile FE
    type: frontend
    group: clients

  - id: bff-web
    name: BFF Web
    type: gateway
    group: bff
    technology: KrakenD
    description: "/web/* · payload đầy đủ"
  - id: bff-mobile
    name: BFF Mobile
    type: gateway
    group: bff
    technology: KrakenD
    description: "/mobile/* · payload rút gọn"

  - id: user-svc
    name: user-service
    type: service
    group: services
  - id: order-svc
    name: order-service
    type: service
    group: services
  - id: product-svc
    name: product-service
    type: service
    group: services

relationships:
  - from: web-fe
    to: bff-web
    type: http
    label: 1 request / màn hình
  - from: mobile-fe
    to: bff-mobile
    type: http
    label: 1 request / màn hình
  - from: bff-web
    to: user-svc
    type: http
  - from: bff-web
    to: order-svc
    type: http
  - from: bff-web
    to: product-svc
    type: http
  - from: bff-mobile
    to: user-svc
    type: http
  - from: bff-mobile
    to: order-svc
    type: http

views:
  - id: overview
    name: Overview
    include: ['*']
    direction: TB
{{< /archcode >}}

---

## 3. Thực hiện với KrakenD

### Vì sao chọn KrakenD

BFF có thể viết bằng code (Node.js, Go, Kotlin…) hoặc khai báo bằng cấu hình. Viết bằng code thì làm được mọi loại logic, nhưng team phải phát triển, test, deploy và vận hành thêm một service. Khai báo bằng cấu hình thì cần ít code hơn nhiều, nhưng chỉ làm được những gì công cụ hỗ trợ.

Chúng tôi thử cách thứ hai với **KrakenD**, một API gateway mã nguồn mở viết bằng Go, lấy aggregation làm tính năng cốt lõi. KrakenD **stateless**, không cần database, và toàn bộ hành vi nằm trong một file JSON. Bản Community Edition (CE) có sẵn các tính năng BFF cần: gộp, lọc, đổi tên dữ liệu, gọi tuần tự, timeout và circuit breaker. Khi cấu hình không đủ, có hai lựa chọn mở rộng: script **Lua** và **plugin Go**.

Câu hỏi đặt ra: **có thể xây lớp BFF chủ yếu bằng cấu hình không, và giới hạn nằm ở đâu?**

### Môi trường thử nghiệm

Chúng tôi dựng hai BFF cho hai loại client, Web và Mobile, trước cùng ba service:

| Thành phần | Vai trò |
|---|---|
| KrakenD 2.9 | BFF Web và BFF Mobile, chạy chung một instance, tách theo prefix `/web` và `/mobile` |
| 3 mock service (Node.js) | user, order, product; độ trễ 30, 80 và 50 ms; có thể bật lỗi giả lập |
| k6 | Đo độ trễ khi gọi qua BFF |

Chúng tôi chọn hai màn hình mẫu: **Dashboard** (gộp 3 service) và **Chi tiết đơn hàng** (gọi tuần tự). Mỗi màn hình có một endpoint trong BFF Web và một endpoint trong BFF Mobile.

### Mỗi loại client một BFF

Cấu hình của mỗi BFF nằm trong một file riêng:

```
gateway/templates/
├── web_endpoints.tmpl       # BFF Web
├── mobile_endpoints.tmpl    # BFF Mobile
└── _shared.tmpl             # block dùng chung, ví dụ circuit breaker
```

Team Web sửa `web_endpoints.tmpl`, team Mobile sửa `mobile_endpoints.tmpl`, nên hai team không đụng vào BFF của nhau. Khi lên production, mỗi file có thể chạy thành một instance KrakenD riêng, đúng nguyên tắc mỗi trải nghiệm một BFF.

### Gộp dữ liệu và lọc field

Endpoint Dashboard của BFF Web khai báo ba `backend`. KrakenD gọi cả ba **song song** rồi gộp kết quả:

```json
{
  "endpoint": "/web/v1/dashboard",
  "backend": [
    {
      "host": ["http://user-svc:3000"],
      "url_pattern": "/users/me",
      "group": "profile",
      "allow": ["id", "name", "email", "avatar", "tier", "address"]
    },
    {
      "host": ["http://order-svc:3000"],
      "url_pattern": "/orders/recent",
      "extra_config": {
        "proxy": {
          "flatmap_filter": [
            { "type": "del",  "args": ["orders.*.warehouse_code"] },
            { "type": "move", "args": ["orders", "recent_orders"] }
          ]
        }
      }
    },
    {
      "host": ["http://product-svc:3000"],
      "url_pattern": "/products/recommended",
      "extra_config": {
        "proxy": {
          "flatmap_filter": [
            { "type": "del",  "args": ["products.*.supplier_cost"] },
            { "type": "move", "args": ["products", "recommended_products"] }
          ]
        }
      }
    }
  ]
}
```

- `group` gói kết quả của một backend vào một key (`profile`).
- `allow` / `deny` giữ hoặc bỏ field ở cấp đầu tiên của object.
- `flatmap_filter` thao tác được với field bên trong mảng (`orders.*.warehouse_code`).

Trace của Dashboard xác nhận các lời gọi chạy song song. Request tổng mất 82 ms, trong khi ba service mất 32 ms (user), 82 ms (order) và 52 ms (product). Thời gian của màn hình bằng thời gian của service chậm nhất, không bằng tổng ba service.

**Bài học đầu tiên** xuất hiện ở bước này. Bản cấu hình đầu tiên dùng `mapping` và `allow`, và chúng tôi nghĩ đã lọc xong. Smoke test cho thấy `supplier_cost` vẫn nằm trong response. Lý do: `allow`, `deny` và `mapping` **không áp dụng cho field bên trong phần tử của mảng**. Chúng tôi phải chuyển sang `flatmap_filter`. Lỗi này không sinh ra thông báo nào và chỉ âm thầm làm lộ dữ liệu. Vì vậy, mọi endpoint trả về mảng cần được review riêng.

### Cùng service, hai loại payload

Endpoint `/mobile/v1/dashboard` của BFF Mobile gọi cùng user-service và order-service. Endpoint này chỉ giữ `name` và `avatar`, bỏ danh sách item và địa chỉ trong đơn hàng, và không gọi product-service.

| Cách gọi | Số request từ FE | Payload |
|---|---|---|
| FE gọi thẳng 3 service | 3 | 1509 B |
| BFF Web | 1 | ~1200 B |
| BFF Mobile | 1 | 301 B |

Payload của Mobile nhỏ hơn khoảng 80% so với khi gọi thẳng, và các service không phải sửa dòng code nào.

### Gọi tuần tự

Màn hình Chi tiết đơn hàng cần lấy đơn hàng trước, rồi dùng `customer_id` trong kết quả để lấy thông tin khách hàng. KrakenD làm việc này bằng `sequential`. Biến `{resp0_customer_id}` lấy giá trị từ response của backend đầu tiên:

```json
{
  "endpoint": "/web/v1/orders/{id}",
  "extra_config": { "proxy": { "sequential": true } },
  "backend": [
    { "url_pattern": "/orders/{id}" },
    { "url_pattern": "/users/{resp0_customer_id}", "group": "customer", "allow": ["name", "email", "phone"] }
  ]
}
```

### Chịu lỗi khi gọi service phía sau

- **Partial response:** khi tắt product-service, Dashboard vẫn trả mã 200 với `profile` và `recent_orders`. Header `X-KrakenD-Completed: false` báo cho FE biết dữ liệu chưa đủ. Một service ngừng hoạt động không làm hỏng cả màn hình.
- **Circuit breaker:** khi product-service lỗi 100%, circuit breaker chuyển sang `open` sau 4 lỗi liên tiếp. Sau đó BFF ngừng gọi service này. Trong 8 request thử, product-service chỉ nhận 4 request.

### Khi cấu hình không đủ

Màn hình Chi tiết đơn hàng cần gắn tên sản phẩm cho từng item. `sequential` chỉ lấy được **một** giá trị từ response trước, nên không lặp được qua danh sách `items[*].product_id`.

Giải pháp gồm hai bước. Thứ nhất, thêm API lấy nhiều sản phẩm trong một lần gọi ở product-service (`/products?ids=p-1,p-3`). Thứ hai, viết script Lua gom các id, gọi API đó và gắn `product_name` vào từng item. Môi trường Lua của KrakenD **không có module `json`**, nên chúng tôi phải tự viết bộ giải mã JSON, dài khoảng 70 dòng.

Lua làm được, nhưng khó test và debug. Với logic gọi HTTP và xử lý JSON như trường hợp này, KrakenD còn hỗ trợ **plugin Go**: có đủ thư viện chuẩn và viết được unit test, đổi lại phải build plugin khớp đúng phiên bản KrakenD. Khi một màn hình cần nhiều logic như vậy, BFF viết bằng code là lựa chọn hợp lý hơn.

### Đo độ trễ

Chúng tôi chạy load test bằng k6: 50 người dùng ảo cho mỗi kịch bản, trong 20 giây. Không có request nào lỗi.

| Kịch bản | avg | p95 | p99 |
|---|---|---|---|
| FE gọi thẳng 3 service song song | 81.2 ms | 83 ms | 84 ms |
| Qua BFF Web | 82.6 ms | 84.6 ms | 89.3 ms |

BFF cộng thêm khoảng **1.5 ms** cho mỗi request. Số đo này lấy trên máy local với mock service, nên chỉ dùng để so sánh tương đối. Dù vậy, số đo cho thấy chi phí của lớp BFF rất nhỏ so với độ trễ của các service.

---

## 4. Kết luận

### Lợi ích của BFF

- **Một màn hình, một request.** FE gọi một endpoint thay vì nhiều service. FE ít phải xử lý loading và lỗi hơn, và màn hình chịu ít độ trễ mạng hơn.
- **Payload đúng nhu cầu của từng loại client.** Mobile chỉ nhận những field nó hiển thị. Trong thử nghiệm, payload của Mobile nhỏ hơn khoảng 80% so với khi gọi thẳng các service.
- **Một điểm kiểm soát dữ liệu đi ra ngoài.** Field nội bộ được lọc tại BFF, trước khi tới trình duyệt.
- **Logic ghép dữ liệu nằm ở một chỗ.** Web, iOS và Android không phải viết lại cùng một logic theo ba cách.
- **Team FE tự chủ.** Team FE sở hữu BFF của mình, nên tự đổi API cho màn hình mà không phải chờ team backend. Các service giữ API chung theo nghiệp vụ.
- **Màn hình chịu được lỗi một phần.** Khi một service không phản hồi, BFF vẫn trả phần dữ liệu còn lại và màn hình vẫn hiển thị.

### Chi phí và khi nào không nên dùng

- **Thêm một bước mạng.** Mỗi request đi qua BFF trước khi tới service. Trong thử nghiệm, chi phí này khoảng 1.5 ms, nhỏ so với độ trễ của chính các service.
- **Thêm thành phần phải vận hành.** Mỗi loại client một BFF nghĩa là thêm một thành phần phải deploy, giám sát và nâng cấp.
- **Trùng lặp giữa các BFF.** BFF Web và BFF Mobile thường gọi cùng các service. Newman chấp nhận phần trùng lặp này, vì gộp các BFF lại sẽ đưa hệ thống quay về một backend chung.
- **Không cần BFF** khi hệ thống chỉ có một loại client, hoặc khi các loại client gọi API giống nhau.

### Bài học rút ra

1. **BFF thuộc về FE.** Ranh giới của BFF là một loại client, không phải một nghiệp vụ.
2. **BFF ghép dữ liệu, service giữ nghiệp vụ.** Logic nghiệp vụ và phân quyền theo dữ liệu thuộc về service. Kiểm tra ở BFF chỉ là lớp bảo vệ thứ hai.
3. **Kiểm thử dữ liệu đi ra từ BFF.** BFF là lớp cuối cùng trước trình duyệt. Một bộ lọc sót field sẽ làm lộ dữ liệu mà không có lỗi nào được báo, nên cần test cho các field nhạy cảm.
4. **Thiết kế cho response chưa đầy đủ.** BFF trả dữ liệu từng phần khi một service lỗi. FE phải biết dữ liệu nào đang thiếu và vẫn hiển thị được phần còn lại.
5. **Chọn cách hiện thực theo độ phức tạp.** Gộp, lọc và định dạng dữ liệu làm tốt bằng cấu hình. Khi một màn hình cần logic điều kiện, gọi lặp qua danh sách hoặc phân quyền phức tạp, BFF viết bằng code dễ test và bảo trì hơn.

BFF không bắt buộc phải là một service viết bằng code. Giá trị của pattern nằm ở chỗ mỗi loại client có một lớp riêng, do đúng team sở hữu, đặt giữa FE và các service.

---

**Tham khảo**

- Sam Newman, [Pattern: Backends For Frontends](https://samnewman.io/patterns/architectural/bff/) (2015)
- Microsoft Azure Architecture Center, [Backends for Frontends pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/backends-for-frontends)
