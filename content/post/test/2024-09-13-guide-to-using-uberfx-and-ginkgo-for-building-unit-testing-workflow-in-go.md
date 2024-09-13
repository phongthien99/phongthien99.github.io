---
title:  Guide to Using Uberfx and Ginkgo for Building Unit Testing Workflow in Go
author: phongthien99
date: 2024-09-13 21:17:00 +0800
categories: [test]
tags: [go,test]
math: true
media_subpath: '/posts/20180809'
---
# Guide to Using Uberfx and Ginkgo for Building Unit Testing Workflow in Go

## Đặt vấn đề

      Để đảm bảo chất lượng mã nguồn trong phát triển phần mềm, unit testing là một yếu tố không thể thiếu. Việc viết và tổ chức các bài kiểm tra đơn vị một cách hiệu quả là điều quan trọng, đặc biệt khi làm việc với các ứng dụng phức tạp. Tuy nhiên, việc này thường gặp khó khăn trong việc quản lý các phụ thuộc và duy trì cấu trúc kiểm tra rõ ràng..

## Giải pháp

     Khi làm việc với Go, việc kết hợp [Uberfx](https://github.com/uber-go/fx) (framework DI) và [Ginkgo](https://github.com/onsi/ginkgo) (thư viện BDD) cung cấp một phương pháp hiệu quả để giải quyết các vấn đề trên. Uberfx giúp quản lý các phụ thuộc của ứng dụng, trong khi Ginkgo hỗ trợ viết các bài kiểm tra đơn vị theo cách dễ đọc và rõ ràng.

Bài viết này sẽ hướng dẫn bạn cách sử dụng Uberfx để quản lý phụ thuộc và Ginkgo để viết các bài kiểm tra, nhằm xây dựng một quy trình phát triển mạnh mẽ và dễ bảo trì.

## Thực hiện

### Bước 1: Tạo module với Uberfx

```yaml
package example

import (
	"errors"
	"sync"

	"go.uber.org/fx"
)

type User struct {
	ID    int    `json:"id"`
	Name  string `json:"name"`
	Email string `json:"email"`
}

type UserService interface {
	Create(user *User) (*User, error)
	Get(id int) (*User, error)
	Update(id int, user *User) (*User, error)
	Delete(id int) error
}

type userService struct {
	users map[int]*User
	lock  sync.RWMutex
	idSeq int
}

func NewUserService() UserService {
	return &userService{
		users: make(map[int]*User),
		idSeq: 1,
	}
}

func (s *userService) Create(user *User) (*User, error) {
	s.lock.Lock()
	defer s.lock.Unlock()
	user.ID = s.idSeq
	s.idSeq++
	s.users[user.ID] = user
	return user, nil
}

func (s *userService) Get(id int) (*User, error) {
	s.lock.RLock()
	defer s.lock.RUnlock()
	user, ok := s.users[id]
	if !ok {
		return nil, errors.New("user not found")
	}
	return user, nil
}

func (s *userService) Update(id int, updatedUser *User) (*User, error) {
	s.lock.Lock()
	defer s.lock.Unlock()
	user, ok := s.users[id]
	if !ok {
		return nil, errors.New("user not found")
	}
	user.Name = updatedUser.Name
	user.Email = updatedUser.Email
	return user, nil
}

func (s *userService) Delete(id int) error {
	s.lock.Lock()
	defer s.lock.Unlock()
	if _, ok := s.users[id]; !ok {
		return errors.New("user not found")
	}
	delete(s.users, id)
	return nil
}

func Module() fx.Option {
	var Module = fx.Provide(NewUserService)

	return Module
}
```

### 

### Giải thích

- `Module() fx.Option`: Hàm này sử dụng `fx.Provide` của Uberfx để cung cấp một thể hiện của `userService` như là một dịch vụ cho các phần khác của ứng dụng. Đây là cách để Uberfx quản lý và cung cấp dịch vụ thông qua DI.

### Cách Hoạt Động

- **Khởi tạo:** Khi ứng dụng khởi động, `Module()` sẽ được gọi để cung cấp `userService` qua Uberfx. Điều này cho phép các phần khác của ứng dụng có thể yêu cầu dịch vụ này mà không cần phải khởi tạo nó trực tiếp.
- **Quản lý Người Dùng:** Các phương thức trong `userService` cho phép tạo, lấy, cập nhật và xóa người dùng trong một bản đồ, với sự đồng bộ hóa cần thiết để xử lý nhiều yêu cầu đồng thời.
- **Dependency Injection:** Uberfx sẽ quản lý việc cung cấp và chia sẻ `userService` trong toàn bộ ứng dụng, giúp cải thiện cấu trúc mã và quản lý các phụ thuộc.

### Bước 2: Viết testcase  với ginkgo

```yaml
package example_test

import (
	"context"
	"testing"
	"unit-test-example/module/example"
	model "unit-test-example/module/example"
	service "unit-test-example/module/example"

	. "github.com/onsi/ginkgo/v2"
	. "github.com/onsi/gomega"
	"go.uber.org/fx"
)

var _ = Describe("UserService with Fx", func() {
	var (
		userService service.UserService
	)

	BeforeEach(func() {
		// Create a new Fx application to initialize dependencies
		app := fx.New(
			example.Module(),          // Register the UserService module
			fx.Populate(&userService), // Automatically inject UserService into the variable
		)

		// Start the Fx application and ensure it starts without errors

		Expect(app.Start(context.Background())).To(Succeed())
	})

	AfterEach(func() {
		// Stop the Fx application after each test to release resources
		app := fx.New()
		Expect(app.Stop(context.TODO())).To(Succeed())
	})
	Describe("UserService", func() {
		It("should not be nil", func() {
			Expect(userService).NotTo(BeNil())
		})
	})
	Describe("Create User", func() {
		It("should create a new user successfully", func() {
			// Define a new user
			user := &model.User{Name: "John Doe", Email: "john@example.com"}

			// Call the Create method from UserService
			createdUser, err := userService.Create(user)

			// Verify that there are no errors
			Expect(err).To(BeNil())

			// Verify that the created user has the expected values
			Expect(createdUser.ID).To(Equal(1))
			Expect(createdUser.Name).To(Equal("John Doe"))
			Expect(createdUser.Email).To(Equal("john@example.com"))
		})
	})

	// Additional test cases for Get, Update, and Delete can be added similarly
})

func TestService(t *testing.T) {
	RegisterFailHandler(Fail)
	RunSpecs(t, "UserService Suite")
}

```

**Giải thích:**

- **`func TestService(t *testing.T)`:** Đây là hàm điểm vào (entry point) cho các bài kiểm tra trong Go. Khi bạn chạy lệnh `go test`, Go sẽ tìm các hàm có tên bắt đầu bằng `Test` và thực thi chúng. Hàm `TestService` là một hàm đặc biệt được sử dụng để khởi động Ginkgo và chạy các bài kiểm tra đã được định nghĩa bằng Ginkgo.
- **`RegisterFailHandler(Fail)`:** Đăng ký một hàm xử lý lỗi. Trong trường hợp có lỗi trong các bài kiểm tra, hàm `Fail` của Ginkgo sẽ được gọi, dẫn đến việc báo cáo lỗi và dừng quá trình kiểm tra.
- **`RunSpecs(t, "UserService Suite")`:** Khởi động Ginkgo và thực hiện tất cả các bài kiểm tra đã được định nghĩa. Tham số đầu tiên là đối tượng `testing.T` để Ginkgo có thể tích hợp với hệ thống kiểm tra của Go. Tham số thứ hai là tên của suite kiểm tra, dùng để tổ chức và báo cáo các bài kiểm tra.
- `fx.Populate(&userService)` yêu cầu Uberfx cung cấp một instance của `UserService` và gán nó cho biến `userService`.
    
     **Làm thế nào `fx.Populate` hoạt động:** Khi ứng dụng Uberfx khởi động, nó tìm các dịch vụ hoặc phụ thuộc đã được đăng ký (trong ví dụ này là `UserService` thông qua `example.Module()`) và tự động cung cấp chúng cho các biến được chỉ định thông qua `fx.Populate`. Điều này giúp tách biệt việc khởi tạo các đối tượng khỏi logic của bài kiểm tra, làm cho mã nguồn trở nên sạch sẽ và dễ bảo trì hơn
    

### Bước 3: Chạy test

  Chạy test bằng go test

```yaml
go run test -v

```

  Kết quả:

```yaml
=== RUN   TestService
Running Suite: UserService Suite - /home/phongthien/Desktop/start-up/unit-test/module/example
=============================================================================================
Random Seed: 1726234713

Will run 2 of 2 specs
[Fx] PROVIDE    fx.Lifecycle <= go.uber.org/fx.New.func1()
[Fx] PROVIDE    fx.Shutdowner <= go.uber.org/fx.(*App).shutdowner-fm()
[Fx] PROVIDE    fx.DotGraph <= go.uber.org/fx.(*App).dotGraph-fm()
[Fx] PROVIDE    example.UserService <= unit-test-example/module/example.NewUserService()
[Fx] INVOKE             reflect.makeFuncStub()
[Fx] RUN        provide: unit-test-example/module/example.NewUserService()
[Fx] RUNNING
[Fx] PROVIDE    fx.Lifecycle <= go.uber.org/fx.New.func1()
[Fx] PROVIDE    fx.Shutdowner <= go.uber.org/fx.(*App).shutdowner-fm()
[Fx] PROVIDE    fx.DotGraph <= go.uber.org/fx.(*App).dotGraph-fm()
•[Fx] PROVIDE   fx.Lifecycle <= go.uber.org/fx.New.func1()
[Fx] PROVIDE    fx.Shutdowner <= go.uber.org/fx.(*App).shutdowner-fm()
[Fx] PROVIDE    fx.DotGraph <= go.uber.org/fx.(*App).dotGraph-fm()
[Fx] PROVIDE    example.UserService <= unit-test-example/module/example.NewUserService()
[Fx] INVOKE             reflect.makeFuncStub()
[Fx] RUN        provide: unit-test-example/module/example.NewUserService()
[Fx] RUNNING
[Fx] PROVIDE    fx.Lifecycle <= go.uber.org/fx.New.func1()
[Fx] PROVIDE    fx.Shutdowner <= go.uber.org/fx.(*App).shutdowner-fm()
[Fx] PROVIDE    fx.DotGraph <= go.uber.org/fx.(*App).dotGraph-fm()
•

Ran 2 of 2 Specs in 0.003 seconds
SUCCESS! -- 2 Passed | 0 Failed | 0 Pending | 0 Skipped
--- PASS: TestService (0.00s)
PASS
ok      unit-test-example/module/example        0.012s
```

## Kết Luận

Bằng cách sử dụng Uberfx để quản lý phụ thuộc và Ginkgo để viết các bài kiểm tra, bạn có thể xây dựng một quy trình phát triển mã nguồn mạnh mẽ và dễ bảo trì hơn. Việc tổ chức mã nguồn và kiểm tra theo cách này giúp cải thiện chất lượng mã và dễ dàng phát hiện lỗi hơn trong các ứng dụng phức tạp.

## Tài liệu tham khảo

- [Uberfx Documentation](https://uber-go.github.io/fx/index.html)
- [Ginkgo Documentation](https://github.com/onsi/ginkgo)