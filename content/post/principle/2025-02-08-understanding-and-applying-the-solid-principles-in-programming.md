---
title: Understanding and Applying the SOLID Principles in Programming
author: phongthien99
date: 2025-02-08 01:17:00 +0800
categories: [Principle]
tags: [principle]
math: true
---
# Understanding and applying the SOLID Principles in Programming



Trong lập trình hướng đối tượng (OOP), SOLID là tập hợp năm nguyên lý giúp viết mã nguồn dễ bảo trì, mở rộng và tránh những vấn đề phổ biến như mã lặp (code duplication) hoặc liên kết chặt chẽ (tight coupling). SOLID là viết tắt của:

- **S**ingle Responsibility Principle (Nguyên lý trách nhiệm duy nhất)
- **O**pen/Closed Principle (Nguyên lý mở/đóng)
- **L**iskov Substitution Principle (Nguyên lý thay thế Liskov)
- **I**nterface Segregation Principle (Nguyên lý phân tách giao diện)
- **D**ependency Inversion Principle (Nguyên lý đảo ngược phụ thuộc)

Hãy cùng tìm hiểu chi tiết từng nguyên lý và cách áp dụng chúng trong thực tế.

## 1. Nguyên Lý Trách Nhiệm Duy Nhất (Single Responsibility Principle - SRP)

**Định nghĩa:** Mỗi lớp chỉ nên có một lý do để thay đổi, tức là chỉ đảm nhận một nhiệm vụ duy nhất.

**Ví dụ sai:** Một lớp xử lý cả logic nghiệp vụ và giao tiếp với cơ sở dữ liệu:

```go
type Report struct {}

func (r *Report) Generate() string {
    return "Báo cáo doanh thu tháng"
}

func (r *Report) SaveToFile() {
    fmt.Println("Lưu báo cáo vào file")
}
```

**Cách cải thiện:** Chia thành hai lớp, một lớp chịu trách nhiệm tạo báo cáo và một lớp khác lo lưu trữ:

```go
type Report struct {}

func (r *Report) Generate() string {
    return "Báo cáo doanh thu tháng"
}

type ReportStorage struct {}

func (s *ReportStorage) SaveToFile(report string) {
    fmt.Println("Lưu báo cáo vào file")
}
```

**Ứng dụng trong kiến trúc phần mềm:**

Khi xây dựng ứng dụng, chúng ta có thể áp dụng nguyên lý SRP bằng cách phân chia các tầng như **Controller, Service, Repository, Model**:

- **Controller**: Chỉ xử lý yêu cầu HTTP và gọi đến **Service**.
- **Service**: Chứa logic nghiệp vụ, không liên quan đến dữ liệu trực tiếp.
- **Repository**: Chịu trách nhiệm truy xuất dữ liệu từ cơ sở dữ liệu.
- **Model**: Chỉ đại diện cho dữ liệu, không chứa logic nghiệp vụ hoặc truy vấn.

Điều này giúp mã nguồn dễ bảo trì, thay đổi một tầng mà không ảnh hưởng đến các tầng khác.

## 2. Nguyên Lý Mở/Đóng (Open/Closed Principle - OCP)

**Định nghĩa:** Một module nên mở để mở rộng nhưng đóng để chỉnh sửa, tức là có thể thêm tính năng mới mà không thay đổi mã nguồn hiện có.

**Ví dụ sai:**

```go
func CalculateBonus(role string, salary float64) float64 {
  if role == "Manager" {
    return salary * 0.2
  } else if role == "Employee" {
    return salary * 0.1
  }
  return 0
}
```

Mỗi lần thêm loại nhân viên mới, chúng ta phải sửa đổi hàm `CalculateBonus`.

**Cách cải thiện:** Sử dụng Strategy Pattern để mở rộng:

```go
type BonusCalculator interface {
  CalculateBonus(salary float64) float64
}

type ManagerBonus struct {}
func (m ManagerBonus) CalculateBonus(salary float64) float64 {
  return salary * 0.2
}

type EmployeeBonus struct {}
func (e EmployeeBonus) CalculateBonus(salary float64) float64 {
  return salary * 0.1
}

type Context struct {
  bonusCalculator BonusCalculator
}

func (c *Context) SetStrategy(b BonusCalculator) {
  c.bonusCalculator = b
}

func (c *Context) ExecuteStrategy(salary float64) float64 {
  return c.bonusCalculator.CalculateBonus(salary)
}
```

Bây giờ, chúng ta có thể dễ dàng thêm loại nhân viên mới mà không cần thay đổi hàm `CalculateBonus`.


## 3. Nguyên Lý Thay Thế Liskov (Liskov Substitution Principle - LSP)

**Định nghĩa:** Các lớp con có thể thay thế lớp cha mà không làm thay đổi hành vi mong đợi của chương trình.

**Ví dụ sai:**

```go
type Bird struct {}
func (b Bird) Fly() {
  fmt.Println("Chim bay trên trời")
}

type Penguin struct {
  Bird
}
```

Chim cánh cụt không thể bay, nhưng kế thừa `Bird` khiến nó có phương thức `Fly()`, dẫn đến lỗi.

**Cách cải thiện:**

**Cách 1: Sử dụng Interface để nhóm hành vi thay vì kế thừa trực tiếp**

```go
type Bird interface {
  Move()
}

type FlyingBird struct{}
func (f FlyingBird) Move() {
  fmt.Println("Bay trên trời")
}

type Penguin struct{}
func (p Penguin) Move() {
  fmt.Println("Đi bộ trên băng")
}
```

Tách hành vi di chuyển thành một interface chung để các loài chim có thể triển khai theo đúng đặc điểm của chúng.





**Cách 2: Dùng Composition Thay Vì Kế Thừa**

```go
type Mover interface {
  Move()
}

type CanFly struct{}
func (f CanFly) Move() {
  fmt.Println("Bay trên trời")
}

type CanWalk struct{}
func (w CanWalk) Move() {
  fmt.Println("Đi bộ trên băng")
}

type Sparrow struct {
  Mover
}

type Penguin struct {
  Mover
}
```

Sử dụng composition giúp phân tách rõ hành vi `Fly` và `Walk`, tránh kế thừa sai và giúp mở rộng dễ dàng.
> **Note:** Khi thiết kế hệ thống, cần xác định rõ liệu lớp mới có thực sự là con của lớp cũ hay chỉ là một lớp mới hoàn toàn với các hành vi khác biệt. Điều này giúp tránh việc kế thừa sai và đảm bảo tuân thủ nguyên lý Liskov.

## 4. Nguyên Lý Phân Tách Giao Diện (Interface Segregation Principle - ISP)

**Định nghĩa:** Một interface lớn không nên ép các lớp triển khai những phương thức mà chúng không sử dụng.

**Ví dụ sai:**

```go
type Worker interface {
    Work()
    Eat()
}
```

Robot không cần phương thức `Eat()`, nhưng vẫn phải triển khai nó.

**Cách cải thiện:**

```go
type Workable interface {
    Work()
}

type Eatable interface {
    Eat()
}
```

Robot chỉ cần triển khai `Workable`, còn con người triển khai cả `Workable` và `Eatable`.

## 5. Nguyên Lý Đảo Ngược Phụ Thuộc (Dependency Inversion Principle - DIP)

**Định nghĩa:** Các module cấp cao không nên phụ thuộc vào module cấp thấp, cả hai nên phụ thuộc vào abstraction(interface).

**Ví dụ sai:**

```go
type MySQLDatabase struct {}
func (db MySQLDatabase) SaveData(data string) {
    fmt.Println("Lưu dữ liệu vào MySQL")
}

type Service struct {
    db MySQLDatabase
}

func (s Service) Store(data string) {
    s.db.SaveData(data)
}
```

Service bị phụ thuộc vào MySQL, gây khó khăn khi chuyển sang PostgreSQL.

**Cách cải thiện:**

```go
type Database interface {
    SaveData(data string)
}

type MySQLDatabase struct {}
func (db MySQLDatabase) SaveData(data string) {
    fmt.Println("Lưu dữ liệu vào MySQL")
}

type Service struct {
    db Database
}

func (s Service) Store(data string) {
    s.db.SaveData(data)
}
```

Bây giờ có thể dễ dàng thay thế MySQL bằng một cơ sở dữ liệu khác.

## Kết Luận

Việc áp dụng SOLID giúp mã nguồn dễ bảo trì, mở rộng và tái sử dụng hơn. Khi thiết kế hệ thống, hãy luôn nghĩ về các nguyên lý này để tránh những vấn đề về kiến trúc và đảm bảo code chất lượng cao.

