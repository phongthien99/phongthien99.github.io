---
title: "Optimizing the Development Workflow with Taskfile in Spring Boot"
date: "2026-01-29T00:00:00Z"
draft: false
tags:
  - Spring Boot
  - JAVA
categories:
  - Devops
author: "phongthien"
---

# Optimizing the Development Workflow with Taskfile in Spring Boot

## 1. Đặt vấn đề

### Nỗi đau của Java Developer

Mỗi ngày, một Java developer phải gõ đi gõ lại những câu lệnh dài dòng:

```bash
mvn clean package -DskipTests
mvn spring-boot:run -Dspring-boot.run.profiles=dev -Dspring-boot.run.jvmArguments="-Dserver.port=8081"
mvn test -Dtest="UserServiceTest"
```

Vấn đề không chỉ dừng ở việc gõ nhiều. Còn có:

**1. Thiếu nhất quán trong team**
- Dev A chạy port 8080, Dev B chạy port 8081
- Người dùng profile `dev`, người dùng `local`
- Config database mỗi người một kiểu

**2. Secret bị commit lên Git**
```properties
# Đừng làm thế này!
spring.datasource.password=my_super_secret_password
```

**3. Onboarding chậm**
- Dev mới vào team mất 1-2 ngày chỉ để setup môi trường
- Đọc README dài 3 trang vẫn không chạy được

**4. CI/CD phức tạp**
- Script build khác với local
- "Works on my machine" syndrome

### Makefile không phải giải pháp tối ưu

Nhiều team chuyển sang dùng Makefile:

```makefile
.PHONY: run
run:
    mvn spring-boot:run
```

Nhưng Makefile có hạn chế:
- Syntax khó đọc với người mới
- Không hỗ trợ cross-platform tốt (Windows)
- Không có dependency management giữa các task
- Không đọc được file `.env`


## 2. Giải pháp: Taskfile

### Taskfile là gì?

[Taskfile](https://taskfile.dev) là một task runner hiện đại, viết bằng Go:
- **Cross-platform**: Windows, macOS, Linux
- **Single binary**: Không cần runtime
- **YAML syntax**: Dễ đọc, dễ viết
- **Dotenv support**: Tự động load `.env`
- **Task dependencies**: Chạy task theo thứ tự

### Triết lý thiết kế

```
┌─────────────────────────────────────────────────────────┐
│                      Developer                          │
│                          │                              │
│                    task dev                             │
│                          │                              │
│                          ▼                              │
│  ┌─────────────────────────────────────────────────┐   │
│  │                 Taskfile.yml                     │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────────────┐  │   │
│  │  │  .env   │  │ Scripts │  │ Maven/Gradle    │  │   │
│  │  └─────────┘  └─────────┘  └─────────────────┘  │   │
│  └─────────────────────────────────────────────────┘   │
│                          │                              │
│                          ▼                              │
│                   Spring Boot App                       │
└─────────────────────────────────────────────────────────┘
```



## 3. Thực hiện

### Bước 1: Cài đặt Taskfile

Bạn có thể cài đặt Taskfile theo hướng dẫn chính thức tại [https://taskfile.dev/docs/installation](https://taskfile.dev/docs/installation).


### Bước 2: Cấu trúc project

```
project/
├── src/main/resources/
│   ├── application.properties        # Config chung (commit)
│   ├── application-local.properties  # Local dev profile
│   └── application-staging.properties # Staging (dùng ENV)
├── Taskfile.yml                      # Task definitions
├── .env.example                      # Template (commit)
├── .env                              # Local secrets (ignored)
└── .gitignore
```

### Bước 3: Tạo .env.example

```bash
# .env.example - Template cho dev mới
SPRING_PROFILES_ACTIVE=local
SERVER_PORT=8081

# Database (cho staging/prod)
DB_HOST=localhost
DB_PORT=5432
DB_NAME=demo
DB_USER=dev
DB_PASSWORD=
```

### Bước 4: Tạo Taskfile.yml

```yaml
# https://taskfile.dev
version: '3'

dotenv: ['.env']

vars:
  MVN: mvn

tasks:
  # ===== DEVELOPMENT =====
  default:
    desc: "Show available tasks"
    cmds:
      - task --list

  dev:
    desc: "Run app in dev mode (with .env)"
    cmds:
      - "{{.MVN}} spring-boot:run -Dspring-boot.run.jvmArguments='-Dserver.port={{.SERVER_PORT}}'"
    env:
      SPRING_PROFILES_ACTIVE: "{{.SPRING_PROFILES_ACTIVE | default \"local\"}}"

  run:
    desc: "Run app (alias for dev)"
    cmds:
      - task dev

  # ===== BUILD =====
  build:
    desc: "Build the project"
    cmds:
      - "{{.MVN}} clean package -DskipTests"

  build:test:
    desc: "Build with tests"
    cmds:
      - "{{.MVN}} clean package"

  compile:
    desc: "Compile only (fast)"
    cmds:
      - "{{.MVN}} compile"

  # ===== TESTING =====
  test:
    desc: "Run all tests"
    cmds:
      - "{{.MVN}} test"

  test:unit:
    desc: "Run unit tests only"
    cmds:
      - "{{.MVN}} test -Dtest='!*IntegrationTest'"

  test:integration:
    desc: "Run integration tests only"
    cmds:
      - "{{.MVN}} test -Dtest='*IntegrationTest'"

  # ===== DATABASE =====
  db:console:
    desc: "Open H2 console in browser"
    cmds:
      - 'echo "H2 Console at http://localhost:{{.SERVER_PORT}}/h2-console"'
      - 'echo "JDBC URL: jdbc:h2:mem:demodb"'

  # ===== UTILITIES =====
  clean:
    desc: "Clean build artifacts"
    cmds:
      - "{{.MVN}} clean"

  deps:
    desc: "Show dependency tree"
    cmds:
      - "{{.MVN}} dependency:tree"

  deps:update:
    desc: "Check for dependency updates"
    cmds:
      - "{{.MVN}} versions:display-dependency-updates"

  # ===== PRODUCTION BUILD =====
  prod:build:
    desc: "Build for production"
    cmds:
      - "{{.MVN}} clean package -Pprod -DskipTests"

  # ===== SETUP =====
  setup:
    desc: "Initial project setup"
    cmds:
      - cp -n .env.example .env || true
      - echo "env file ready"
      - "{{.MVN}} clean compile"
      - echo "Project compiled. Run 'task dev' to start"

```

### Bước 5: Config Spring Boot đọc ENV

**application-local.properties:**
```properties
# Server port từ ENV, default 8081
server.port=${SERVER_PORT:8081}

# H2 in-memory cho local dev
spring.datasource.url=jdbc:h2:mem:demodb
spring.h2.console.enabled=true
```

**application-staging.properties:**
```properties
# PostgreSQL với ENV variables
spring.datasource.url=jdbc:postgresql://${DB_HOST}:${DB_PORT}/${DB_NAME}
spring.datasource.username=${DB_USER}
spring.datasource.password=${DB_PASSWORD}
```


### Sử dụng hàng ngày

```bash
# Dev mới join team
git clone <repo>
task setup          # Tự động copy .env và compile

# Chạy app
task dev            # 1 lệnh duy nhất!

# Build & Test
task build          # Build nhanh
task test           # Chạy test
task test:unit      # Chỉ unit test

# Xem tất cả tasks
task --list
```

**Output của `task --list`:**
```
task: Available tasks for this project:
* build:          Build without tests
* build:test:     Build with tests
* clean:          Clean build artifacts
* default:        Show available tasks
* dev:            Run app in dev mode
* setup:          Initial project setup
* test:           Run all tests
* test:integration: Run integration tests
* test:unit:      Run unit tests only
```

### Advanced: Task Dependencies

```yaml
tasks:
  # Chạy lint trước khi test
  test:
    deps: [lint]
    cmds:
      - "{{.MVN}} test"

  lint:
    cmds:
      - "{{.MVN}} checkstyle:check"

  # Build chỉ khi test pass
  release:
    deps: [test]
    cmds:
      - "{{.MVN}} clean package -Pprod"
```

### Advanced: Parallel Tasks

```yaml
tasks:
  ci:
    desc: "Run CI pipeline"
    deps:
      - task: lint
      - task: test:unit
      - task: test:integration
    # Tất cả deps chạy song song!
```

---

## 4. Kết luận

### Lợi ích đạt được

| Trước | Sau |
|-------|-----|
| Gõ lệnh dài | `task dev` |
| Secret trong code | `.env` (gitignored) |
| Onboarding 1-2 ngày | `task setup` - 5 phút |
| Script CI khác local | Cùng Taskfile |
| Mỗi người config khác | Team dùng chung `.env.example` |

### Nguyên tắc quan trọng

1. **Commit `.env.example`, ignore `.env`**
   - Template là documentation sống
   - Secret không bao giờ lên Git

2. **CI/CD không phụ thuộc Taskfile**
   - Taskfile là convenience cho dev
   - CI chạy Maven/Gradle trực tiếp với ENV từ secrets

3. **Profile phân tách rõ ràng**
   - `local`: H2, không cần setup
   - `staging/prod`: Real DB, config từ ENV

4. **Một lệnh để chạy**
   - Dev mới: `task setup && task dev`
   - Không cần đọc README dài


*"Automation is not about replacing humans. It's about freeing humans to do more meaningful work."*

Happy coding!
