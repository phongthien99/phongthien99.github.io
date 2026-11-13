---
title: "PostgreSQL High Availability (Part 1): A Production Guide to Patroni, etcd v3, HAProxy and PgCat"
date: "2026-11-13T00:00:00Z"
draft: false
tags:
  - postgresql
  - patroni
  - etcd
  - haproxy
  - pgcat
  - devops
categories:
  - Devops
author: "phongthien"
---

# PostgreSQL High Availability (Part 1): A Production Guide to Patroni, etcd v3, HAProxy and PgCat

Cách dựng một cụm PostgreSQL HA có **failover tự động**, **replication đồng bộ**, định tuyến bằng **health-check của HAProxy** và **connection pooling với PgCat**. Bài viết có kèm số liệu đo thật từ một POC Docker Compose.



## Tóm tắt & Kết luận kiến trúc

- **Vấn đề cốt lõi:** Streaming replication của PostgreSQL không có cơ chế đồng thuận tự động. Khi primary sập hoặc bị cô lập mạng, các script failover tự viết hoặc gây **split-brain** (hai node cùng nhận ghi), hoặc không promote được standby.
- **Topology tham chiếu:** Cụm etcd 3 node giữ leader lock. Patroni quản lý 3 node PostgreSQL 16. HAProxy định tuyến theo REST API của Patroni. PgCat đứng phía trước để gom kết nối và tách đọc/ghi.
- **Mục tiêu khôi phục:** Đo thực tế được **RTO ~31 s** khi primary chết đột ngột và **6,7 s** khi switchover có kế hoạch. Bật `synchronous_mode` thì **RPO = 0**. Không xảy ra split-brain trong bất kỳ kịch bản nào, kể cả khi etcd mất quorum hoặc primary bị cô lập mạng.



## 1. Vì sao HA kiểu truyền thống thất bại khi lên production

Phần lớn các cụm PostgreSQL "HA" tự dựng chỉ gồm một primary, một hoặc hai standby và một script theo dõi. Kiểu thiết lập này thường thất bại vì ba lý do sau.

### 1.1 Thảm họa split-brain

Script theo dõi ping primary, không thấy phản hồi trong 10 giây thì promote standby. Vấn đề là primary **chưa chết**. Nó chỉ bị treo tạm thời, chẳng hạn kernel khựng 15 giây hoặc switch mạng chập chờn. Khi hồi lại, primary cũ vẫn tin mình là leader và tiếp tục nhận ghi. Lúc này hai node cùng ghi dữ liệu và hai timeline tách rời nhau. Việc hợp nhất dữ liệu sau đó gần như phải làm bằng tay, từng bảng một.

Cách xử lý duy nhất đáng tin cậy là **một nguồn sự thật bên ngoài có cơ chế đồng thuận**. Node nào không chứng minh được với nguồn đó rằng mình vẫn là leader thì phải tự hạ cấp.

### 1.2 Cái bẫy của replication bất đồng bộ

Streaming replication mặc định là bất đồng bộ: primary trả lời `COMMIT` cho client **trước khi** WAL tới được standby. Nếu primary chết đúng lúc đó, các giao dịch mà client đã thấy "thành công" sẽ không tồn tại trên node được promote. Ứng dụng không hề biết dữ liệu đã mất.

Bật replication đồng bộ thủ công qua `synchronous_standby_names` lại tạo ra vấn đề ngược lại: khi standby đồng bộ chết, primary **treo mọi lệnh ghi**. Cần một tầng điều khiển biết tự đổi standby đồng bộ khi topology thay đổi.

### 1.3 Bão kết nối khi promote primary

PostgreSQL dùng mô hình **một process cho mỗi kết nối**. Ngay sau failover, hàng trăm instance ứng dụng cùng reconnect vào primary mới, mỗi kết nối phải fork process và xác thực SCRAM. Primary mới bị dồn tải đúng lúc nó vừa promote xong, khi cache còn lạnh.

Ứng dụng mở và đóng kết nối liên tục (PHP, serverless, cron job) chịu chi phí này cả khi không có sự cố.



## 2. Kiến trúc tham chiếu: Patroni + etcd + HAProxy + PgCat

{{< archcode title="PostgreSQL HA - Kiến trúc tham chiếu" mode="preview" view="overview" layout="/diagrams/postgresql-ha.layout.yaml" >}}
version: 1

system:
  id: postgresql-ha
  name: PostgreSQL High Availability

groups:
  - id: access
    name: Tầng truy cập — pooling & định tuyến
    type: network
  - id: pg-cluster
    name: Cụm PostgreSQL 16 + Patroni
    type: cluster
  - id: dcs
    name: DCS — etcd v3 (Raft, quorum = 2)
    type: cluster

components:
  - id: app
    name: Ứng dụng
    type: system

  - id: pgcat
    name: PgCat :6432
    type: gateway
    group: access
    description: Pooling + tách đọc/ghi
  - id: haproxy
    name: HAProxy
    type: gateway
    group: access
    description: ":5000 → GET /primary · :5001 → GET /replica"

  - id: pg1
    name: pg1 — Leader
    type: database
    group: pg-cluster
    technology: PostgreSQL 16 + Patroni
  - id: pg2
    name: pg2 — Sync Standby
    type: database
    group: pg-cluster
    technology: PostgreSQL 16 + Patroni
  - id: pg3
    name: pg3 — Replica
    type: database
    group: pg-cluster
    technology: PostgreSQL 16 + Patroni

  - id: etcd1
    name: etcd1
    type: storage
    group: dcs
  - id: etcd2
    name: etcd2
    type: storage
    group: dcs
  - id: etcd3
    name: etcd3
    type: storage
    group: dcs

relationships:
  - from: app
    to: pgcat
    type: sql
    label: ":6432"
  - from: pgcat
    to: haproxy
    type: tcp
    label: "primary → :5000 · replica → :5001"
  - from: haproxy
    to: pg1
    type: sql
    label: Ghi (/primary = 200)
  - from: haproxy
    to: pg2
    type: sql
    label: Đọc (/replica = 200)
  - from: haproxy
    to: pg3
    type: sql
    label: Đọc (/replica = 200)
  - from: pg1
    to: pg2
    label: Streaming replication (sync)
  - from: pg1
    to: pg3
    label: Streaming replication
  - from: pg-cluster
    to: dcs
    type: http
    label: Leader lock (TTL)

views:
  - id: overview
    name: Overview
    include: ['*']
    direction: TB
{{< /archcode >}}

### 2.1 Trách nhiệm của từng thành phần

- **Cụm etcd (3 node):** Là DCS (Distributed Configuration Store), lưu leader key có TTL và cấu hình động của cụm dưới `/service/pg-ha/`. Quorum là 2/3, nên chịu được mất 1 node. Khi mất quorum, không node nào gia hạn được leader key và primary buộc phải hạ cấp.
- **Patroni (chạy trên mỗi node PostgreSQL):** Mỗi `loop_wait` giây chạy một vòng HA. Leader gia hạn key, replica theo dõi. Khi key hết hạn, replica có WAL mới nhất (lag ≤ `maximum_lag_on_failover`) giành lock và promote. Primary cũ quay lại sẽ được `pg_rewind` thay vì phải clone lại toàn bộ.
- **PostgreSQL 16 (3 node):** Streaming replication dùng replication slot cho từng replica, nên WAL không bị xoá trước khi replica nhận. Bật `data-checksums` và `wal_log_hints` để `pg_rewind` hoạt động.
- **HAProxy:** Proxy TCP thuần, nhưng health-check bằng HTTP tới REST API `:8008` của Patroni:
  - `:5000` chỉ chuyển tới node trả `200` cho `GET /primary` — luôn đúng một node.
  - `:5001` cân bằng round-robin giữa các node trả `200` cho `GET /replica`.
- **PgCat:** Connection pooler viết bằng Rust, có phân tích cú pháp SQL. Ứng dụng chỉ biết một endpoint duy nhất là `pgcat:6432`. `SELECT` được chuyển sang replica, mọi câu lệnh khác sang primary.

> **Về thứ tự tầng:** PgCat đứng **trước** HAProxy, không nối thẳng vào PostgreSQL. PgCat chỉ biết vai trò của server qua cấu hình tĩnh và không tự phát hiện failover. HAProxy, nhờ Patroni, luôn biết primary thật đang ở đâu. Trỏ PgCat vào hai cổng cố định `5000`/`5001` giúp failover hoàn toàn trong suốt với tầng pooling.

---

## 3. Cấu hình production

### 3.1 Cấu hình DCS cho Patroni (`patroni.yml`)

Cấu hình dưới đây dùng chung cho cả ba node. Giá trị riêng của từng node (`PATRONI_NAME`, `PATRONI_RESTAPI_CONNECT_ADDRESS`, `PATRONI_POSTGRESQL_CONNECT_ADDRESS`, mật khẩu) được truyền qua biến môi trường.

> **Ràng buộc bắt buộc:** `loop_wait + 2 × retry_timeout ≤ ttl`.

```yaml
scope: pg-ha
namespace: /service/

restapi:
  listen: 0.0.0.0:8008

etcd3:
  hosts: etcd1:2379,etcd2:2379,etcd3:2379

bootstrap:
  dcs:
    ttl: 30                            # leader key hết hạn sau 30 s nếu không được gia hạn
    loop_wait: 10                      # chu kỳ vòng HA
    retry_timeout: 10                  # thời gian thử lại DCS/PG trước khi hạ cấp
    maximum_lag_on_failover: 1048576   # 1 MB: replica lag hơn mức này không được promote
    synchronous_mode: false            # bật động bằng patronictl khi cần RPO = 0
    postgresql:
      use_pg_rewind: true              # primary cũ tự rewind thay vì clone lại
      use_slots: true                  # replication slot cho mỗi replica
      parameters:
        wal_level: replica
        hot_standby: "on"
        wal_log_hints: "on"            # bắt buộc cho pg_rewind
        max_wal_senders: 10
        max_replication_slots: 10
        wal_keep_size: 256MB
        password_encryption: scram-sha-256
  initdb:
    - encoding: UTF8
    - data-checksums

postgresql:
  listen: 0.0.0.0:5432
  data_dir: /var/lib/postgresql/data/pgdata
  bin_dir: /usr/lib/postgresql/16/bin
  basebackup:
    - checkpoint: fast                 # không chờ spread checkpoint khi clone replica
```

Hãy chú ý dòng `basebackup: checkpoint: fast`. Nếu thiếu nó, `pg_basebackup` chờ một spread checkpoint, mặc định rải trên 90% của `checkpoint_timeout` 5 phút. Replica mới sẽ đứng ở trạng thái `creating replica` tới khoảng **4,5 phút** mà log không báo lỗi gì.

**Về `synchronous_mode`:** Patroni mặc định `synchronous_mode_strict: false`. Khi không còn standby nào khả dụng, primary vẫn tiếp tục nhận ghi, nghĩa là ưu tiên tính sẵn sàng. Chỉ đặt `strict: true` khi việc mất dữ liệu là không thể chấp nhận và có thể chịu được việc primary chặn ghi.

### 3.2 Định tuyến động bằng HAProxy (`haproxy.cfg`)

HAProxy không hiểu giao thức PostgreSQL. Toàn bộ logic định tuyến dựa vào mã HTTP mà Patroni trả về. Trong môi trường container, phải khai báo `resolvers` để HAProxy khởi động được kể cả khi node PostgreSQL chưa lên.

```haproxy
resolvers docker
    nameserver dns1 127.0.0.11:53
    hold valid 5s

defaults
    mode tcp
    timeout connect 4s
    timeout client  30m
    timeout server  30m
    timeout check   5s

listen primary
    bind *:5000
    option httpchk GET /primary
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions resolvers docker init-addr last,libc,none
    server pg1 pg1:5432 maxconn 500 check port 8008
    server pg2 pg2:5432 maxconn 500 check port 8008
    server pg3 pg3:5432 maxconn 500 check port 8008

listen replicas
    bind *:5001
    balance roundrobin
    option httpchk GET /replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions resolvers docker init-addr last,libc,none
    server pg1 pg1:5432 maxconn 500 check port 8008
    server pg2 pg2:5432 maxconn 500 check port 8008
    server pg3 pg3:5432 maxconn 500 check port 8008
```

Hãy chú ý directive `on-marked-down shutdown-sessions`. Khi một node mất vai trò primary, HAProxy cắt ngay mọi kết nối đang mở tới nó, thay vì để client tiếp tục ghi vào một node đã chuyển sang read-only. `inter 3s fall 3` nghĩa là thay đổi vai trò được phát hiện trong khoảng **9 giây**.

---

## 4. Connection Pooling: PgCat

PgCat chạy ở chế độ **transaction**: một kết nối server chỉ bị giữ trong suốt một giao dịch rồi trả về pool. Bật query parser để tách đọc/ghi tự động.

```toml
[general]
port = 6432
connect_timeout = 3000
healthcheck_delay = 10000
ban_time = 10

[pools.appdb]
pool_mode = "transaction"
query_parser_enabled = true
query_parser_read_write_splitting = true
primary_reads_enabled = false          # SELECT chỉ đi replica

[pools.appdb.users.0]
username = "app"
pool_size = 20
min_pool_size = 0                      # mở kết nối lazy, xem caveat bên dưới

[pools.appdb.shards.0]
database = "appdb"
servers = [["haproxy", 5000, "primary"], ["haproxy", 5001, "replica"]]
```

Hãy chú ý `min_pool_size = 0`. Nếu đặt lớn hơn 0, PgCat mở sẵn kết nối tới `haproxy:5001` ngay lúc khởi động, có thể trước khi HAProxy kịp đánh dấu replica nào là UP. Các kết nối đó chết ngay, và request đọc đầu tiên nhận lỗi `AllServersDown`.

> ⚠️ **Lưu ý quan trọng cho production: PgCat không tự retry kết nối hỏng**
>
> Sau mỗi lần failover hoặc restart HAProxy, các kết nối đang nằm trong pool của PgCat bị cắt ở phía sau (do `shutdown-sessions` hoặc do HAProxy dừng). Khi client mượn một kết nối như vậy, PgCat health-check, phát hiện hỏng và trả lỗi `AllServersDown` ngay cho client, thay vì mở kết nối mới rồi thử lại. Lỗi tự hết sau vài request, khi các kết nối hỏng đã bị loại khỏi pool.
>
> **Hệ quả:** ứng dụng **bắt buộc** phải có cơ chế retry cho lỗi kết nối, ví dụ 2–3 lần với backoff khoảng 1 giây. Không tầng HA nào miễn cho ứng dụng trách nhiệm này. Giảm `idle_timeout` và `server_lifetime` thu hẹp được khoảng thời gian bị ảnh hưởng nhưng không loại bỏ được hoàn toàn.

Chế độ transaction còn có các ràng buộc quen thuộc của mọi pooler: `SET` cấp session, advisory lock cấp session, `LISTEN/NOTIFY` và temporary table không hoạt động đúng giữa các giao dịch.

---

## 5. Diễn tập failover & khôi phục sự cố

Một bài test failover nên chạy trước khi go-live: tạo tải ghi liên tục qua PgCat, sau đó `docker kill` (hoặc tắt nguồn VM) node đang là leader.

> `kill -9` tiến trình postgres **không phải** là bài test failover: Patroni vẫn sống, nó sẽ khởi động lại PostgreSQL tại chỗ mà không có cuộc bầu leader nào.

Client đo lường cần chạy ở một container hoặc máy riêng. Nếu chạy `pgbench` bên trong container sắp bị kill, công cụ đo sẽ chết cùng đối tượng được đo.

### 5.1 Dòng thời gian failover (primary chết đột ngột)

| Giai đoạn | Sự kiện | Phản ứng của hệ thống |
|---|---|---|
| T+0 s | Primary bị kill | Kết nối ghi lỗi, PgCat trả lỗi cho client |
| T+3–9 s | HAProxy health-check thất bại 3 lần | Node bị đánh dấu DOWN trên `:5000`, cổng ghi không còn backend |
| T+≤30 s | Leader key trong etcd hết TTL | Các replica phát hiện không còn leader |
| T+~30 s | Replica so sánh WAL LSN | Node đủ điều kiện (Sync Standby nếu bật sync mode) giành lock và chạy `pg_ctl promote` |
| T+~31 s | `/primary` trả 200 trên node mới | HAProxy đưa node vào `:5000`, lệnh ghi qua PgCat thành công trở lại |
| T+~35 s | Replica còn lại nhận leader mới | Replica chuyển sang follow timeline mới |
| Khi node cũ khởi động lại | Patroni phát hiện timeline lệch | Chạy `pg_rewind` rồi rejoin làm replica |

### 5.2 Kết quả các kịch bản đã diễn tập

| # | Kịch bản | Kết quả đo được |
|---|---|---|
| T1 | `docker kill` leader | Ghi lại được sau **30,9 s** |
| T2 | `patronictl switchover` | Ghi lại được sau **6,7 s** |
| T3 | Leader cũ khởi động lại | `pg_rewind` exit code=0, rejoin làm replica |
| T4 | Kill 1/3 etcd | Không ảnh hưởng |
| T5 | Kill 2/3 etcd (mất quorum) | Leader tự hạ cấp sau ~12 s, cụm chuyển sang read-only. Khôi phục etcd thì cụm tự lành |
| T6 | Cô lập mạng primary | Primary cũ tự hạ cấp, node khác lên leader sau ~25 s. Tối đa 1 primary tại mọi thời điểm |
| T7 | Restart HAProxy / PgCat | Ghi lại được sau 10,6 s / 1,6 s |
| T8 | `synchronous_mode` + kill leader dưới tải | 905 giao dịch đã ACK → 905 dòng trên primary mới. **RPO = 0** |

Kịch bản **T5** thường gây bất ngờ cho đội ứng dụng: mất quorum etcd khiến toàn bộ cụm từ chối ghi, dù cả ba node PostgreSQL đều khỏe. Đây là hành vi đúng thiết kế, vì hệ thống chọn **nhất quán** thay vì **sẵn sàng**. Nếu cần giữ primary hoạt động khi chỉ DCS gặp sự cố, xem xét `failsafe_mode: true` của Patroni.

**Cách kiểm chứng RPO ở T8:** một client ghi tuần tự các số nguyên và chỉ ghi lại số nào đã được server xác nhận `COMMIT`. Sau failover, so sánh danh sách đó với dữ liệu trên primary mới. Số nào đã được xác nhận mà không tồn tại trong database chính là dữ liệu bị mất.

Hãy ghi lại thời gian thực tế trong môi trường của bạn, xác nhận không giao dịch đã ACK nào bị mất sau test, và lặp lại các bài diễn tập này sau mỗi lần nâng cấp Patroni, etcd hoặc thay đổi topology mạng.

---

## 6. Benchmark: Chi phí của từng tầng

Số liệu dưới đây đo bằng `pgbench`, scale 50 (~750 MB), 30 s mỗi lượt. Toàn bộ cụm và client chạy chung một máy 16 vCPU cùng các dịch vụ khác, nên chỉ nên dùng số liệu để **so sánh tương đối** (nhiễu khoảng ±30%). Hãy benchmark lại với workload của chính bạn trước khi chốt cấu hình.

| Kịch bản (32 client) | Trực tiếp | Qua PgCat |
|---|---|---|
| Ghi TPC-B, async | 1 480 TPS · 22 ms | 754 TPS · 42 ms |
| Ghi TPC-B, sync | 879 TPS · 36 ms | 589 TPS · 54 ms |
| Đọc select-only (primary) | 22 083 TPS · 1,45 ms | — |
| Đọc select-only (replica `:5001`) | 28 940 TPS · 1,11 ms | 18 895 TPS · 1,69 ms |
| Mở kết nối mới mỗi giao dịch (`-C`) | 301 TPS · 106 ms | 981 TPS · 33 ms |

Ba kết luận rút ra:

1. **PgCat có overhead khi kết nối sống lâu.** TPS giảm 15–50% do thêm một hop proxy, chi phí phân tích SQL và giới hạn `pool_size = 20` buộc 32 client phải xếp hàng.
2. **PgCat có lợi rõ rệt khi kết nối sống ngắn:** thông lượng tăng 3,3 lần, độ trễ giảm 3 lần. Đây là lý do chính để đặt pooler trong kiến trúc.
3. **Replication đồng bộ làm giảm ~40% thông lượng ghi** ở 32 client. Đó là cái giá của RPO = 0, và nên được quyết định theo yêu cầu nghiệp vụ.

---

## 7. Ma trận quyết định kiến trúc

| Tiêu chí | Script failover tự viết | Patroni + etcd + HAProxy | Patroni + etcd + HAProxy + PgCat |
|---|---|---|---|
| Chống split-brain | Không | Có (bầu leader qua DCS) | Có (bầu leader qua DCS) |
| Failover tự động | Có, nhưng không an toàn | Có, ~30 s | Có, ~30 s |
| RPO = 0 | Phải cấu hình thủ công | Có (`synchronous_mode`) | Có (`synchronous_mode`) |
| Tách đọc/ghi | Ứng dụng tự xử lý | Hai cổng 5000/5001, ứng dụng tự chọn | Tự động qua query parser |
| Chịu kết nối ngắn | Kém | Kém (301 TPS với `-C`) | Tốt (981 TPS với `-C`) |
| Độ trễ ghi thêm | 0 | ~0 | +~20 ms ở 32 client |
