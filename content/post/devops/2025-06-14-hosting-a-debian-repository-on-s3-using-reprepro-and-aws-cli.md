---
title: "Hosting a Debian Repository on S3 using Reprepro and AWS CLI"
date: 2025-06-14T00:00:00Z
draft: false
tags:
  - debian
  - repository
  - aws
  - s3
  - reprepro
  - devops
categories:
  - DevOps
  - Cloud
author: "Your Name"
---

# Hosting a Debian Repository on S3 using Reprepro and AWS CLI

## I. Đặt vấn đề

Trong quá trình phát triển phần mềm, đặc biệt với hệ thống backend hoặc hạ tầng đóng gói nội bộ, việc phát hành các gói `.deb` nhanh chóng và có thể truy cập qua `apt` là một lợi thế rất lớn. Tuy nhiên, việc dựng hạ tầng riêng (như NGINX, APT server...) có thể phức tạp, tốn chi phí và không dễ mở rộng.

Một giải pháp hiệu quả, đơn giản hơn là: **host repository Debian trực tiếp trên Amazon S3** và sử dụng **`reprepro`** để quản lý metadata.

---

## II. Cách giải quyết

Chúng ta sẽ xây dựng một quy trình:

- Tạo repo Debian chuẩn bằng `reprepro`.
- Cập nhật các gói `.deb` vào repo.
- Đồng bộ repo lên Amazon S3.
- Sinh ra file `.list` để client dễ cấu hình `apt`.

### Công cụ sử dụng:

| Công cụ | Vai trò |
| --- | --- |
| `reprepro` | Tạo & quản lý cấu trúc Debian repo (`conf/`, `pool/`, `dists/`) |
| `aws s3 sync` | Đồng bộ thư mục repo lên Amazon S3 |
| Shell Script | Tự động hóa các bước thêm gói, build metadata, và đồng bộ hóa |

---

## III. Thực hiệ

### 1. `update-repo.sh` – **Thêm `.deb` vào repository**

### ✅ Nhiệm vụ:

- Kiểm tra và tạo cấu hình `conf/distributions` nếu chưa có.
- Thêm gói `.deb` vào đúng `codename` bằng `reprepro`.

### 🧩 Nội dung chính:

```bash

#!/bin/bash
set -e

REPO_DIR="${REPO_DIR:-./repo}"      # Thư mục chứa repo
DEB_FILE="$1"                        # File .deb đầu vào
DIST="$2"                            # Codename (vd: v1.0)
ARCH="${ARCH:-amd64}"
COMPONENT="${COMPONENT:-main}"

if [[ -z "$DEB_FILE" || -z "$DIST" ]]; then
    echo "Usage: $0 path/to/package.deb <codename>"
    exit 1
fi

if [[ ! -f "$DEB_FILE" ]]; then
    echo "❌ File not exist: $DEB_FILE"
    exit 2
fi

mkdir -p "$REPO_DIR/conf"
DIST_FILE="$REPO_DIR/conf/distributions"
if ! grep -q "Codename: $DIST" "$DIST_FILE" 2>/dev/null; then
    echo "➕ Adding new codename: $DIST"
    cat >> "$DIST_FILE" <<EOF

Origin: LocalCompany
Label: LocalRepo
Codename: $DIST
Architectures: $ARCH
Components: $COMPONENT
Description: Auto-generated distribution $DIST
EOF
fi

echo "📦 Adding package: $DEB_FILE → codename: $DIST"
reprepro -b "$REPO_DIR" includedeb "$DIST" "$DEB_FILE"

echo "✅ Completed!"

```

### 🛠️ Cách chạy:

```bash

./update-repo.sh ./incoming/myapp_1.0.0.deb v1.0

```

---

### 2. `generate-sources-list.sh` – **Tạo file `sources.list` cho client**

### ✅ Nhiệm vụ:

- Duyệt file `conf/distributions` để lấy danh sách các `Codename` và `Component`.
- Sinh ra dòng cấu hình repo chuẩn cho `apt`.

### 🧩 Nội dung chính:

```bash

#!/bin/bash

REPO_URL=${1:-"http://localhost/repo"}     # URL repo
OUTPUT=${2:-"localrepo.list"}              # Tên file output
DISTRIBUTIONS_FILE="repo/conf/distributions"

> "$OUTPUT"  # Clear file

grep -E '^Codename:' "$DISTRIBUTIONS_FILE" | awk '{print $2}' | while read -r codename; do
  component=$(grep -A 10 "Codename: $codename" "$DISTRIBUTIONS_FILE" | grep -E '^Components:' | awk '{$1=""; print $0}' | xargs -n1 | sort -u | xargs)
  if [[ -n "$component" ]]; then
    echo "deb [trusted=yes,arch=amd64] $REPO_URL $codename $component" >> "$OUTPUT"
  else
    echo "⚠️ Warning: No components found for Codename: $codename"
  fi
done

echo "✅ File $OUTPUT created:"
cat "$OUTPUT"

```

### 🛠️ Cách chạy:

```bash

./generate-sources-list.sh https://my-s3-bucket.s3.amazonaws.com/repo my-repo.list

```

File output `my-repo.list` có dạng:

```

deb [trusted=yes,arch=amd64] https://my-s3-bucket.s3.amazonaws.com/repo v1.0 main

```

---

### 3. `publish.sh` – **Đồng bộ hóa toàn bộ repo lên S3**

### ✅ Nhiệm vụ:

- Lấy cấu hình từ S3 về local.
- Duyệt và thêm tất cả file `.deb` hiện có vào repo.
- Xóa `repo/db` để tránh sync dữ liệu cache không cần thiết.
- Sync toàn bộ repo lên S3.
- Tạo file `.list` và upload nó lên S3.

### 🧩 Nội dung chính:

```bash

#!/bin/bash
set -e

S3_BUCKET="$1"
REPO_NAME="$2"
VERSION="$3"
PUBLIC_HOST_URL="$4"

if [[ -z "$S3_BUCKET" || -z "$REPO_NAME" || -z "$VERSION" || -z "$PUBLIC_HOST_URL" ]]; then
  echo "Usage: $0 <S3_BUCKET> <REPO_NAME> <VERSION> <PUBLIC_HOST_URL>"
  exit 1
fi

aws s3 sync s3://$S3_BUCKET/repo/conf ./repo/conf --exact-timestamps

for file in ./*.deb; do
  ./update-repo.sh "$file" "$VERSION"
done

rm -rf ./repo/db
aws s3 sync ./repo s3://$S3_BUCKET/repo

./generate-sources-list.sh "$PUBLIC_HOST_URL" "$REPO_NAME.list"
aws s3 cp ./$REPO_NAME.list s3://$S3_BUCKET/source/$REPO_NAME.list

```

### 🛠️ Cách chạy:

```bash

./publish.sh my-s3-bucket my-repo v1.0 https://my-s3-bucket.s3.amazonaws.com/repo

```

---

## 4.Thử nghiêm

Client có thể cấu hình APT như sau:

```bash

sudo curl -o /etc/apt/sources.list.d/my-repo.list https://my-s3-bucket.s3.amazonaws.com/source/my-repo.list
sudo apt update

```

---

## V. Kết luận

Việc sử dụng `reprepro` kết hợp với S3 là một giải pháp:

- **Nhẹ, đơn giản**
- **Không cần web server**
- **Dễ tích hợp CI/CD**