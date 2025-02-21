---
title: Unit Testing the Controller Layer in Web Applications
author: phongthien99
date: 2025-02-21 23:17:00 +0800
categories: [Test]
tags: [Test]
math: true
media_subpath: '/posts/20180809'
---
# *Unit Testing the Controller Layer in Web Applications*

Unit test là một phần quan trọng trong phát triển phần mềm, giúp đảm bảo từng thành phần nhỏ của hệ thống hoạt động chính xác. Trong ứng dụng web, controller xử lý request và response, gọi service để lấy dữ liệu, kiểm tra đầu vào, và trả về mã trạng thái HTTP phù hợp. Viết unit test cho controller giúp phát hiện lỗi sớm và đảm bảo logic hoạt động đúng.

## 1. Lợi ích của unit test cho controller

- Xác minh logic xử lý trong controller hoạt động đúng.
- Phát hiện lỗi sớm trước khi triển khai.
- Tăng độ tin cậy cho ứng dụng.
- Cô lập lỗi dễ dàng bằng cách sử dụng mock service.

## 2. Xác định phạm vi test hợp lý

- **Chỉ kiểm tra logic trong controller**, không test service hoặc database.
- **Kiểm tra các trường hợp quan trọng**, gồm cả thành công và thất bại.
- **Viết test tối giản nhưng đầy đủ**.
- **Dùng mock service để cô lập controller**.

### 2.1. Thiết lập môi trường

Ví dụ, một ứng dụng Go dùng `echo` framework có controller như sau:

```go
package example

import (
	"net/http"
	"strconv"

	"github.com/labstack/echo/v4"
)

type User struct {
	Id   int    `json:"id"`
	Name string `json:"name"`
}

type IUserService interface {
	GetUserId(id int) (*User, error)
}

type IExampleController interface {
	GetUser()
}

type exampleController struct {
	router  *echo.Group
	service IUserService
}

func NewExampleController(router *echo.Group, service IUserService) IExampleController {
	return &exampleController{
		router:  router.Group("/example"),
		service: service,
	}
}

func (e *exampleController) GetUser() {
	e.router.GET("/:id", func(c echo.Context) error {
		id := c.Param("id")
		idInt, err := strconv.Atoi(id)
		if err != nil {
			return c.JSON(http.StatusBadRequest, map[string]string{"error": "Invalid user ID"})
		}
		user, err := e.service.GetUserId(idInt)
		if err != nil {
			return c.JSON(http.StatusInternalServerError, map[string]string{"error": err.Error()})
		}
		return c.JSON(http.StatusOK, user)
	})
}
```

### 2.2. Viết unit test

```go
package example_test

import (
	"example/transport/http/example"
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/labstack/echo/v4"
	. "github.com/onsi/ginkgo/v2"
	. "github.com/onsi/gomega"
	"github.com/stretchr/testify/mock"
)

type MockUserService struct {
	mock.Mock
}

func (m *MockUserService) GetUserId(id int) (*example.User, error) {
	args := m.Called(id)
	if args.Error(1) != nil {
		return nil, args.Error(1)
	}
	return args.Get(0).(*example.User), nil
}

var _ = Describe("ExampleController", func() {
	var (
		e           *echo.Echo
		rec         *httptest.ResponseRecorder
		mockService *MockUserService
		controller  example.IExampleController
	)

	BeforeEach(func() {
		e = echo.New()
		rec = httptest.NewRecorder()
		mockService = new(MockUserService)
		controller = example.NewExampleController(e.Group(""), mockService)
		controller.GetUser()
	})

	Describe("GetUser", func() {
		Context("when the user ID is valid", func() {
			It("should return the user", func() {
				user := &example.User{Id: 1, Name: "John Doe"}
				mockService.On("GetUserId", 1).Return(user, nil)

				req := httptest.NewRequest(http.MethodGet, "/example/1", nil)
				e.ServeHTTP(rec, req)

				Expect(rec.Code).To(Equal(http.StatusOK))
				Expect(rec.Body.String()).To(ContainSubstring("John Doe"))
			})
		})

		Context("when the user ID is invalid", func() {
			It("should return a bad request error", func() {
				req := httptest.NewRequest(http.MethodGet, "/example/invalid", nil)
				e.ServeHTTP(rec, req)

				Expect(rec.Code).To(Equal(http.StatusBadRequest))
				Expect(rec.Body.String()).To(ContainSubstring("Invalid user ID"))
			})
		})

		Context("when the service returns an error", func() {
			It("should return an internal server error", func() {
				mockService.On("GetUserId", 1).Return(nil, echo.NewHTTPError(http.StatusInternalServerError, "Service error"))

				req := httptest.NewRequest(http.MethodGet, "/example/1", nil)
				e.ServeHTTP(rec, req)

				Expect(rec.Code).To(Equal(http.StatusInternalServerError))
				Expect(rec.Body.String()).To(ContainSubstring("Service error"))
			})
		})
	})
})

func TestService(t *testing.T) {
	RegisterFailHandler(Fail)
	RunSpecs(t, "Example Suite")
}
```

## 3. Kết luận

Viết unit test cho controller giúp đảm bảo logic xử lý đúng, phát hiện lỗi sớm và dễ dàng bảo trì. Bằng cách sử dụng mocking, ta có thể cô lập controller khỏi các thành phần khác. Việc tối ưu số lượng test case giúp tiết kiệm thời gian mà vẫn đảm bảo chất lượng. Sử dụng framework như `ginkgo` và `gomega` giúp test dễ đọc và duy trì hơn.

