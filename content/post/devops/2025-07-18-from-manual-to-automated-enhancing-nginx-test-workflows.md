---
title: "From Manual to Automated: Enhancing Nginx Test Workflows"
date: "2025-07-18T00:00:00Z"
draft: false
tags:
  - nginx
  - testcontainers
  - ginkgo
  - devops
categories:
  - devops
  - testing
author: "phongthien"
---

# From Manual to Automated: Enhancing Nginx Test Workflows

## Problem Statement

Trong quá trình phát triển và vận hành hệ thống sử dụng **Nginx làm reverse proxy**, việc thêm, sửa hoặc xóa các cấu hình `location`, `rewrite`, hoặc `proxy_pass` có thể:

- **Ảnh hưởng không mong muốn đến các routing khác** (ví dụ: route `/admin/` bị chặn khi thêm `/admin/api`)
- **Gây lỗi ẩn** khó phát hiện do trùng khớp thứ tự `location` hoặc `rewrite` không đúng như mong đợi
- Thiếu công cụ kiểm thử tự động nên việc đảm bảo tính đúng đắn phụ thuộc nhiều vào manual test hoặc sản phẩm thật

Hệ quả:

- **Regression**: thay đổi một route ảnh hưởng đến các route khác

## Solution

Để giải quyết những vấn đề trên, chúng ta cần một **bộ test tự động mạnh mẽ để kiểm thử routing của Nginx**. Mục tiêu là đảm bảo các routing mới **không ảnh hưởng** đến các route hiện có, đồng thời có thể **diễn giải và in ra cây routing logic** giúp kiểm tra độ ưu tiên và khớp `location` một cách chính xác. Bộ test này cũng phải **tái sử dụng** được cho nhiều phiên bản cấu hình Nginx khác nhau trong các quy trình CI/CD, staging, hoặc kiểm tra pull request.

**Đề xuất kỹ thuật:**

- **Sử dụng `testcontainers` kết hợp `Ginkgo/Gomega`**: Đây là sự kết hợp lý tưởng để mô phỏng hệ thống Nginx với cấu hình cụ thể. Mỗi test case sẽ khởi tạo một Nginx container với file `nginx.conf` tương ứng, gửi các HTTP request và kiểm tra response có đúng như kỳ vọng.
- **Xây dựng bộ test matrix (routing path → backend mong muốn)**: Cách tiếp cận này giúp dễ dàng mở rộng khi có thêm route mới, đồng thời kiểm tra hiệu quả các trường hợp xung đột, rewrite sai, hoặc khớp sai thứ tự.

## Implementation

### Định nghĩa routing và Test Case

**Routing cần test:**

```nginx
server {
    listen 80;

    location /web {
        proxy_pass http://whoami.com;
    }

    location /api/ {
        rewrite ^/api/(.*)$ /$1 break;
        proxy_pass http://whoami.com;
    }
}
```

**Bảng Test Case:**

| Test Case ID | Input URL | Expected URI nhận tại backend | Ghi chú |
| --- | --- | --- | --- |
| TC001 | `/web` | `GET /web` | `proxy_pass` giữ nguyên URI |
| TC002 | `/api/demo` | `GET /demo` | `rewrite` + `proxy_pass` |
| TC003 | `/api/test/x` | `GET /test/x` | `rewrite` sâu hơn |
| TC004 | `/api/` | `GET /` | `rewrite /api/` → `/` |

### Mã nguồn Ginkgo test: `nginx_test.go`

```go
package nginx_test

import (
	"context"
	"fmt"
	"io"
	"log"
	"net/http"
	"testing"
	"time"

	. "github.com/onsi/ginkgo/v2"
	. "github.com/onsi/gomega"
	"github.com/testcontainers/testcontainers-go"
	"github.com/testcontainers/testcontainers-go/network"
	"github.com/testcontainers/testcontainers-go/wait"
)

var (
	ctx     context.Context
	net     *testcontainers.DockerNetwork
	whoamiC testcontainers.Container
	nginxC  testcontainers.Container
)

var _ = BeforeSuite(func() {
	log.Print("Setting up test environment...")
	ctx = context.Background()

	// Tạo Docker network dùng chung
	var err error
	net, err = network.New(ctx)
	Expect(err).NotTo(HaveOccurred())

	// Tạo container whoami
	whoamiReq := testcontainers.ContainerRequest{
		Image:        "traefik/whoami:latest",
		Name:         "whoami",
		Hostname:     "whoami.com",
		ExposedPorts: []string{"80/tcp"},
		Networks:     []string{net.Name},
		WaitingFor:   wait.ForHTTP("/").WithPort("80/tcp").WithStartupTimeout(10 * time.Second),
	}

	whoamiC, err = testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
		ContainerRequest: whoamiReq,
		Started:          true,
	})
	Expect(err).NotTo(HaveOccurred())

	// Tạo container nginx
	nginxReq := testcontainers.ContainerRequest{
		Image:        "nginx:latest",
		ExposedPorts: []string{"80/tcp"},
		Networks:     []string{net.Name},
		WaitingFor:   wait.ForHTTP("/").WithPort("80/tcp").WithStartupTimeout(10 * time.Second),
		Files: []testcontainers.ContainerFile{
			{
				HostFilePath:      "nginx.conf", // Đường dẫn đến file cấu hình nginx trên máy local
				ContainerFilePath: "/etc/nginx/nginx.conf",
				FileMode:          0644,
			},
		},
	}

	nginxC, err = testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
		ContainerRequest: nginxReq,
		Started:          true,
	})
	Expect(err).NotTo(HaveOccurred())
})

var _ = Describe("Nginx Reverse Proxy to Whoami", func() {
	testCases := []struct {
		name          string
		path          string
		expectedURI   string
		expectSuccess bool
	}{
		{"proxy_pass without rewrite", "/web", "GET /web", true},
		{"rewrite + proxy_pass 1", "/api/demo", "GET /demo", true},
		{"rewrite + proxy_pass 2", "/api/test/x", "GET /test/x", true},
		{"rewrite + proxy_pass root", "/api/", "GET /", true},
	}

	for _, tc := range testCases {
		tc := tc // capture range variable
		It(fmt.Sprintf("should handle route %s → %s", tc.path, tc.expectedURI), func() {
			host, _ := nginxC.Host(ctx)
			port, _ := nginxC.MappedPort(ctx, "80")
			url := fmt.Sprintf("http://%s:%s%s", host, port.Port(), tc.path)

			resp, err := http.Get(url)
			Expect(err).NotTo(HaveOccurred())
			defer resp.Body.Close()

			body, _ := io.ReadAll(resp.Body)
			log.Printf("Response for %s: %s", tc.path, string(body))

			if tc.expectSuccess {
				Expect(resp.StatusCode).To(Equal(200))
				Expect(string(body)).To(ContainSubstring(tc.expectedURI))
				Expect(string(body)).To(ContainSubstring("Hostname: whoami.com"))
			} else {
				Expect(string(body)).NotTo(ContainSubstring("Hostname: whoami.com"))
			}
		})
	}

})

var _ = AfterSuite(func() {
	log.Print("Tearing down test environment...")
	if whoamiC != nil {
		whoamiC.Terminate(ctx)
	}
	if nginxC != nil {
		nginxC.Terminate(ctx)
	}
	if net != nil {
		net.Remove(ctx)
	}
})

func TestNginx(t *testing.T) {
	RegisterFailHandler(Fail)
	RunSpecs(t, "Nginx Suite")
}
```

Để chạy bộ test này, bạn chỉ cần lưu cấu hình Nginx vào file `nginx.conf` cùng cấp với file `nginx_test.go` và chạy lệnh trong terminal:

```bash
go test -v
```

## Conclusion

Việc kiểm thử cấu hình Nginx một cách tự động là một bước tiến quan trọng trong việc đảm bảo sự ổn định và chính xác của hệ thống. Bằng cách kết hợp **Testcontainers** và **Ginkgo/Gomega**, chúng ta đã xây dựng được một bộ khung mạnh mẽ, cho phép:

- **Tự động hóa kiểm thử**: Giảm thiểu đáng kể rủi ro gây ra bởi các thay đổi cấu hình thủ công.
- **Phát hiện sớm lỗi**: Các vấn đề về routing, rewrite, hoặc xung đột cấu hình sẽ được nhận diện ngay trong giai đoạn phát triển hoặc CI/CD.
- **Kiểm soát phức tạp**: Giúp quản lý hiệu quả hơn các cấu hình Nginx ngày càng phức tạp, đặc biệt trong các hệ thống microservices.
