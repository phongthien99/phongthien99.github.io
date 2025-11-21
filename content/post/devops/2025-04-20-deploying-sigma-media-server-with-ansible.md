---
title: Deploying Sigma Media Server with Ansible
author: phongthien99
date: 2025-04-20 12:40:00 +0800
categories: [Devops]
tags: [ansible]
math: true
media_subpath: '/posts/20240503'
---
# Deploying Sigma Media Server with Ansible
Trong bài viết này, chúng ta sẽ cùng nhau triển khai **Sigma Media Server** lên máy chủ từ xa thông qua **Ansible** — công cụ tự động hóa cấu hình và triển khai phần mềm phổ biến. Bằng cách sử dụng Ansible, bạn có thể tự động hóa quá trình cài đặt và cấu hình phần mềm trên nhiều máy chủ, giúp tiết kiệm thời gian và giảm thiểu lỗi.

---

## 1. Cấu Trúc Dự Án

Dự án được tổ chức theo chuẩn **Ansible Role**, giúp mã dễ bảo trì và mở rộng. Cấu trúc thư mục của dự án như sau:

```

devops
├── inventory.ini
├── playbook.yml
└── roles
    └── sigma/
        ├── handlers/
        │   └── main.yml
        └── tasks/
            └── main.yml

```

- **`inventory.ini`**: Định nghĩa các máy chủ mục tiêu và thông tin kết nối.
- **`playbook.yml`**: Playbook chính chứa các bước triển khai.
- **`roles/sigma/handlers/main.yml`**: Định nghĩa các hành động cần thực hiện sau khi có thay đổi.
- **`roles/sigma/tasks/main.yml`**: Các task thực thi như cài đặt phần mềm, cấu hình hệ thống.

---

## 2. File `inventory.ini`

Đây là nơi khai báo máy chủ đích (target host), nơi mà Ansible sẽ thực hiện các thao tác từ xa. Nội dung file `inventory.ini` như sau:

```

[sigma_servers]
172.16.60.123 ansible_user=dev ansible_password= ansible_become=true

```

- **ansible_user**: tài khoản đăng nhập từ xa.
- **ansible_password**: mật khẩu đăng nhập.
- **ansible_become**: cho phép nâng quyền `sudo` nếu cần.

---

## 3. File `playbook.yml`

Playbook chính mô tả quá trình triển khai, bao gồm các tác vụ từ việc cài đặt phần mềm cho đến khởi động dịch vụ. Nội dung file `playbook.yml` như sau:

```yaml

- name: Deploy Sigma Media Server
  hosts: sigma_servers
  become: yes
  roles:
    - sigma

```

> Playbook này sẽ áp dụng toàn bộ vai trò sigma lên nhóm máy chủ sigma_servers.
> 

---

## 4. Tác vụ triển khai (`roles/sigma/tasks/main.yml`)

Trong phần này, các tác vụ chính được định nghĩa để cài đặt **Sigma Media Server** và thực hiện các thao tác cần thiết như thêm repository, cài đặt gói phần mềm, và ghi nhận phiên bản.

```yaml
---
- name: Download Sigma software repository configuration file
  get_url:
    url: https://repo.sigma.video/debian/sigma-media-server.list
    dest: /etc/apt/sources.list.d/sigma-media-server.list
    mode: '0644'

- name: Update apt package list
  apt:
    update_cache: yes
    cache_valid_time: 3600

- name: Install Sigma Media Server
  apt:
    name: sigma-media-server
    state: present
  changed_when: true
  notify:
    - Start Sigma Media Server service
    - Query Sigma Media Server API
    - Show version running

- name: Create certificate directory if not exists
  file:
    path: /etc/pki/tls/certs
    state: directory
    mode: '0755'

- name: Copy CA certificates
  copy:
    src: /etc/ssl/certs/ca-certificates.crt
    dest: /etc/pki/tls/certs/ca-bundle.crt
    mode: '0644'
    backup: yes
    
- name: Get installed version of Sigma Media Server
  command: dpkg -s sigma-media-server
  register: sigma_version_info
  changed_when: false

- name: Log version
  debug:
    msg: >-
      sigma-media-server: {{
        sigma_version_info.stdout_lines
        | select('search', '^Version:')
        | map('regex_replace', '^Version \\s*', '')
        | first
      }}

```

Tóm lược các bước:

- **Thêm repo Sigma** vào hệ thống.
- **Cập nhật danh sách gói** (`apt update`).
- **Cài đặt `sigma-media-server`**.
- **Kiểm tra và copy CA** vào thư mục phục vụ cho hệ thống.
- **Ghi nhận phiên bản** của phần mềm Sigma đã cài đặt.

---

## 5. Handler khởi động và kiểm tra (`roles/sigma/handlers/main.yml`)

Sau khi các task có thay đổi, **handler** sẽ thực hiện các hành động như khởi động lại dịch vụ và kiểm tra trạng thái API của Sigma. Nội dung file `handlers/main.yml` như sau:

```yaml
- name: Start Sigma Media Server service
  systemd:
    name: sigma-media-server
    state: started
    enabled: true

- name: Query Sigma Media Server API
  uri:
    url: http://localhost:9999
    method: GET
    return_content: yes
    status_code: 200
  register: sigma_api_response
  retries: 5
  delay: 2
  until: sigma_api_response.status == 200

- name: Show version running
  debug:
    msg: "Sigma Media Server run version: {{ sigma_api_response.json.version }}"

```

### **Mối Quan Hệ Giữa `notify` và `handler`**

Trong Ansible, **`notify`** và **`handler`** có mối quan hệ mật thiết, hỗ trợ quá trình tự động hóa một cách hiệu quả.

- **`notify`** là thuộc tính trong các task, dùng để thông báo cho Ansible khi có thay đổi. Khi một task thay đổi trạng thái (ví dụ: cài đặt thành công), nó sẽ "thông báo" cho các handler đã được chỉ định trong phần `notify` của task.
- **`handler`** là các tác vụ đặc biệt được định nghĩa trong phần `handlers`. Chúng chỉ được thực thi khi có sự "thông báo" từ task. Handler thường được sử dụng để thực hiện các thao tác như khởi động lại dịch vụ, kiểm tra trạng thái hệ thống, hoặc các hành động bổ sung khác mà chỉ cần thực thi khi có thay đổi.

Ví dụ trong bài viết:

- Khi task **cài đặt `sigma-media-server`** thành công, các **handler** sẽ được thông báo và thực thi:
    - **Start Sigma Media Server service**: Khởi động dịch vụ Sigma.
    - **Query Sigma Media Server API**: Kiểm tra xem API của Sigma có đang hoạt động chính xác không.
    - **Show version running**: In ra phiên bản của Sigma Media Server hiện đang chạy.

Nhờ vào cơ chế **`notify`** và **`handler`**, các tác vụ bổ sung chỉ được thực thi khi có sự thay đổi thực sự, giúp tiết kiệm tài nguyên và giảm thiểu việc thực thi không cần thiết.

---

## 6. Triển Khai Thực Tế

Sau khi cấu hình xong, bạn có thể chạy lệnh dưới đây để triển khai:

```bash
ansible-playbook -i inventory.ini playbook.yml --ask-pass --ask-become-pass
```

- **- `i inventory.ini`**: Chỉ định tệp inventory chứa thông tin máy chủ đích.
- **`playbook.yml`**: Chỉ định tệp playbook chứa các task cần thực hiện.
- **`-ask-pass`**: Yêu cầu nhập mật khẩu để kết nối SSH đến máy chủ đích.
- **`-ask-become-pass`**: Yêu cầu nhập mật khẩu `sudo` để thực hiện các tác vụ yêu cầu quyền quản trị.

> Nếu thiết lập đúng, hệ thống sẽ cài đặt và khởi động Sigma Media Server, đồng thời in ra phiên bản đang chạy.
> 

---

## Kết Luận

Với việc đóng gói cấu hình Ansible thành **role** `sigma`, bạn có thể dễ dàng:

- Tự động hóa triển khai trên nhiều máy chủ.
- Kiểm tra trạng thái hoạt động dịch vụ.
- Ghi lại version nhằm phục vụ giám sát và audit.
