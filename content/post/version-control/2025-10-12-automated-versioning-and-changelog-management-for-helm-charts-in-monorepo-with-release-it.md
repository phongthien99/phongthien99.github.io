---
title: "Automated Versioning and Changelog Management for Helm Charts in a Monorepo with Release It"
author: phongthien99
date: 2025-10-12 00:40:00 +0800
categories: [version-control]
tags: [version-control]
math: true
---

# Automated Versioning and Changelog Management for Helm Charts in a Monorepo with Release It

## **1. Đặt vấn đề**

Trong các dự án **monorepo**, nơi chứa nhiều **Helm chart** triển khai cho từng service hoặc module khác nhau (ví dụ: `demo-umbrella-chart`, `whoami`, `api-gateway`...), việc quản lý **version và release** cho từng chart riêng lẻ thường rất phức tạp.

Nếu thực hiện thủ công, bạn sẽ cần:

- Sửa `version` trong `Chart.yaml`.
- Sinh changelog.
- Tạo Git tag tương ứng.
- Đảm bảo chart hợp lệ (`helm lint`).

Với hàng chục chart trong cùng repo, việc này dễ sai sót, tốn thời gian, và khó tích hợp vào pipeline CI/CD.



## **2. Giải pháp**

Sử dụng công cụ [**Release It**](https://github.com/release-it/release-it) để tự động hóa quy trình release cho từng chart độc lập.

Mỗi Helm chart sẽ có:

- Một file cấu hình riêng `.release-it.json` (hoặc dùng script sinh động).
- Cấu hình plugin để:
    - Tự động cập nhật `version` trong `Chart.yaml`.
    - Sinh changelog tự động (`CHANGELOG.md`).
    - Tạo tag Git riêng biệt theo tên chart.
- Các hook để lint chart, commit file, và in log kết quả.

Cách này giúp:

- Mỗi chart có **chu kỳ release riêng biệt**.
- **Dễ mở rộng** khi thêm chart mới.
- **Dễ tích hợp CI/CD** (GitHub Actions, GitLab CI, Jenkins,…).



## **3. Thực hiện**

### 3.1 Cấu trúc thư mục monorepo

```
├── demo-umbrella-chart
│   ├── CHANGELOG.md
│   ├── Chart.lock
│   ├── charts
│   │   ├── mariadb-18.0.3.tgz
│   │   └── redis-19.0.2.tgz
│   ├── Chart.yaml
│   ├── templates
│   └── values.yaml
├── whoami
│   ├── charts
│   ├── Chart.yaml
│   ├── templates
│   └── values.yaml
├── RELEASE.md
├── package.json
├── pnpm-lock.yaml

```

> Mỗi thư mục con như demo-umbrella-chart hay whoami đại diện cho một Helm chart độc lập.
> 



### 3.2 Cấu hình `release-it` tổng quát cho từng chart

Tạo file:

**`<chart-name>/.release-it.json`**

Ví dụ:

`demo-umbrella-chart/.release-it.json` hoặc `whoami/.release-it.json`

```json
{
  "npm": {
    "publish": false},
  "git": {
    "tagName": "<chart-name>/v${version}",
    "tagMatch": "<chart-name>/v*",
    "commitMessage": "chore(<chart-name>): release v${version}",
    "tagAnnotation": "Release <chart-name> v${version}",
    "getLatestTagFromAllRefs": true,
    "requireCleanWorkingDir": false,
    "preset": "angular"
  },
  "github": {
    "release": false,
    "releaseName": "<chart-name> v${version}",
    "releaseNotes": "echo 'See CHANGELOG.md for details'"
  },
  "hooks": {
    "before:init": [
      "echo '🚀 Starting release for <chart-name>'",
      "helm lint ./<chart-name>"
    ],
    "after:bump": [
      "git add ./<chart-name>/Chart.yaml"
    ],
    "after:git:release": "echo '✅ <chart-name> v${version} released successfully'"
  },
  "plugins": {
    "@release-it/bumper": {
      "in": {
        "file": "./<chart-name>/Chart.yaml",
        "path": "version"
      },
      "out": {
        "file": "./<chart-name>/Chart.yaml",
        "path": "version"
      }
    },
    "@release-it/conventional-changelog": {
      "preset": "angular",
      "infile": "./<chart-name>/CHANGELOG.md",
      "header": "# Changelog\n\nAll notable changes to <chart-name> will be documented in this file.",
      "gitRawCommitsOpts": {
        "path": "./<chart-name>"
      }
    }
  }
}

```

> 🔧 Ghi chú:
> 
> - Thay `<chart-name>` bằng tên thật của chart (vd: `demo-umbrella-chart`, `whoami`).
> - Mỗi chart có thể copy cấu hình này và chỉ cần đổi tên là dùng được.
> - `@release-it/bumper` sẽ tự động sửa `version` trong `Chart.yaml`.
> - `@release-it/conventional-changelog` tạo `CHANGELOG.md` dựa trên commit Angular style.

---

### 3.3 Quy trình release

Chạy lệnh sau cho chart cụ thể:

```bash

npx release-it --config ./<chart-name>/.release-it.json

```

Ví dụ:

```bash
npx release-it --config ./demo-umbrella-chart/.release-it.json

```

Release It sẽ:

1. Lint chart (`helm lint`).
2. Sinh `CHANGELOG.md` dựa trên commit.
3. Tăng version trong `Chart.yaml`.
4. Commit và tạo tag `demo-umbrella-chart/vX.Y.Z`.
5. Hiển thị log hoàn tất release.

---

## **4. Kết luận**

Việc áp dụng **Release It** cho từng Helm chart trong monorepo giúp:

✅ **Chuẩn hóa quy trình release** — mọi chart đều có cấu hình thống nhất.

✅ **Tách biệt version control** — mỗi chart có tag riêng (`<chart-name>/vX.Y.Z`).

✅ **Tự động hóa 100%** — từ lint, version bump, đến changelog và tag Git.

✅ **Dễ mở rộng** — chỉ cần copy file cấu hình, đổi `<chart-name>` là xong.

✅ **Dễ tích hợp CI/CD**, đảm bảo release luôn ổn định và lặp lại được.