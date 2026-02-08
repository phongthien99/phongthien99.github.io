---
title: "Object-Oriented Programming in Golang"
author: phongthien99
date: 2026-02-08 00:00:00 +0700
categories: ["Principle", "Programming"]
tags: [Principle]
math: true

---
# Object-Oriented Programming in Golang

## 1. Đặt vấn đề

Khi học lập trình, đa số chúng ta đều được tiếp cận với **lập trình hướng đối tượng (OOP)** qua các ngôn ngữ như Java, C++, C#, hoặc Python. Những khái niệm như **class, kế thừa, đa hình** trở nên quen thuộc và gần như là tiêu chuẩn.

Tuy nhiên, khi chuyển sang **Golang**, nhiều lập trình viên cảm thấy bối rối:

- Go không có `class`
- Không có `extends` hay `inheritance` truyền thống
- Không có constructor chuẩn như OOP cổ điển

Vậy câu hỏi đặt ra là:

> Golang có hỗ trợ OOP không? Nếu có thì được thiết kế như thế nào?
> 

Bài viết này sẽ giúp bạn hiểu cách Go tiếp cận OOP theo một triết lý đơn giản và linh hoạt hơn.

---

## 2. Lý thuyết: 4 tính chất của OOP

Lập trình hướng đối tượng được xây dựng trên 4 tính chất cốt lõi. Những tính chất này giúp hệ thống phần mềm:

- Dễ bảo trì
- Dễ mở rộng
- Giảm phụ thuộc giữa các thành phần
- Tăng khả năng tái sử dụng code

---

### 2.1 Encapsulation – Đóng gói

**Encapsulation** là nguyên tắc **gói dữ liệu và hành vi vào cùng một đối tượng**, đồng thời **ẩn các chi tiết nội bộ** và chỉ cho phép truy cập thông qua các phương thức được kiểm soát.

Nói cách khác:

> Đối tượng không cho phép truy cập trực tiếp vào trạng thái bên trong nếu điều đó có thể gây ra lỗi hoặc phá vỡ logic nghiệp vụ.
> 

---

### 2.2 Abstraction – Trừu tượng

**Abstraction** là quá trình **ẩn đi chi tiết cài đặt**, chỉ cung cấp **giao diện hành vi** cho người sử dụng.

Nguyên tắc:

> Người dùng chỉ cần biết đối tượng làm được gì, không cần biết nó hoạt động như thế nào.
> 

### Ví dụ đời thực

Khi sử dụng máy ATM:

- Bạn chỉ thấy: rút tiền, chuyển tiền, kiểm tra số dư.
- Không cần biết:
    - Hệ thống core banking xử lý ra sao
    - Database được thiết kế như thế nào

Đó là trừu tượng.

---

### 2.3 Inheritance – Kế thừa

**Inheritance** là cơ chế cho phép **class con tái sử dụng thuộc tính và hành vi của class cha**.

Mục đích chính:

- Tái sử dụng code
- Xây dựng hệ thống theo dạng phân cấp
- Giảm lặp lại logic

---

### 2.4 Polymorphism – Đa hình

**Polymorphism** nghĩa là:

> Cùng một interface hoặc hành vi, nhưng có nhiều cách thực hiện khác nhau.
> 

Mục tiêu:

- Viết code linh hoạt
- Giảm phụ thuộc vào implementation cụ thể
- Dễ mở rộng hệ thống

### Compile-time polymorphism (đa hình tại thời điểm biên dịch)

Đây là loại đa hình được **quyết định khi compile**, thường xuất hiện dưới dạng:

- Function overloading
- Operator overloading
- Template (C++)

Compiler sẽ chọn hàm phù hợp dựa trên kiểu dữ liệu.

➡ Quyết định được đưa ra ở **compile time**.

---

### Runtime polymorphism (đa hình tại thời điểm chạy)

Đây là loại đa hình xảy ra khi:

- Hành vi thực sự chỉ được xác định khi chương trình chạy.
- Thường thông qua:
    - Virtual method
    - Interface
    - Method overriding

---

## 3. OOP trong Golang

Go **không phải là ngôn ngữ OOP thuần túy**, nhưng vẫn hỗ trợ đầy đủ 4 tính chất trên theo cách riêng.

Triết lý của Go:

> “Composition over inheritance”
> 
> 
> (Ưu tiên ghép thành phần thay vì kế thừa)
> 

---

### 3.1 Encapsulation trong Go

Go sử dụng **quy tắc chữ hoa – chữ thường** để đóng gói dữ liệu.

```go
type User struct {
    Name string // public
    age  int    // private
}

func (u *User) SetAge(age int) {
    if age > 0 {
        u.age = age
    }
}

func (u User) GetAge() int {
    return u.age
}
```

- `Name` có thể truy cập từ package khác.
- `age` bị ẩn, chỉ truy cập qua method.

Đây chính là **đóng gói** trong Go.

---

### 3.2 Abstraction bằng interface

Go dùng **interface** để định nghĩa hành vi.

```go
type Animal interface {
    Speak() string
}
```

Interface chỉ quan tâm:

- Đối tượng có thể **làm gì**
- Không quan tâm **nó làm thế nào**

---

### 3.3 Inheritance bằng Composition (Embedding)

Go không có kế thừa truyền thống, thay vào đó dùng **struct embedding**.

```go
type Animal struct {
    Name string
}

func (a Animal) Speak() string {
    return "Some sound"
}

type Dog struct {
    Animal // embedding
}
```

`Dog` không kế thừa `Animal`, mà **nhúng** `Animal` vào bên trong.

Nhờ vậy:

```go
d := Dog{
    Animal: Animal{Name: "Buddy"},
}

fmt.Println(d.Speak())
```

`Dog` vẫn có method `Speak()`.

---

### 3.4 Polymorphism bằng interface

```go
type Animal interface {
    Speak() string
}

type Dog struct{}
func (Dog) Speak() string {
    return "Woof"
}

type Cat struct{}
func (Cat) Speak() string {
    return "Meow"
}

func MakeSound(a Animal) {
    fmt.Println(a.Speak())
}
```

Sử dụng:

```go
MakeSound(Dog{})
MakeSound(Cat{})
```

Cùng một interface:

- Nhưng mỗi struct có hành vi khác nhau.

Đây là **đa hình** trong Go.

---

## 4. Kết hợp Abstraction + Embedding

Một pattern phổ biến trong Go:

```go
type Logger interface {
    Log(message string)
}

type BaseLogger struct {
    prefix string
}

func (b BaseLogger) Log(message string) {
    fmt.Println(b.prefix + ": " + message)
}

type FileLogger struct {
    BaseLogger
    filePath string
}
```

Sử dụng:

```go
func PrintLog(l Logger) {
    l.Log("Hello")
}

f := FileLogger{
    BaseLogger: BaseLogger{prefix: "FILE"},
    filePath:   "/tmp/log.txt",
}

PrintLog(f)
```

Ở đây:

- `Logger` → abstraction
- `BaseLogger` → logic chung
- `FileLogger` → embedding để tái sử dụng code

---

## 5. Kết hợp OOP với Generic (từ Go 1.18)

Từ **Go 1.18**, ngôn ngữ này bổ sung **Generics**, giúp viết code tổng quát, tái sử dụng tốt hơn, đặc biệt khi kết hợp với các nguyên lý OOP.

Generics cho phép:

- Viết struct và function dùng cho nhiều kiểu dữ liệu
- Giảm lặp code
- Giữ được type safety

---

### 5.1 Generic struct

Ví dụ một repository tổng quát:

```go
type Repository[T any] struct {
    data []T
}

func (r *Repository[T]) Save(item T) {
    r.data = append(r.data, item)
}

func (r *Repository[T]) All() []T {
    return r.data
}
```

Sử dụng:

```go
type User struct {
    Name string
}

userRepo := Repository[User]{}
userRepo.Save(User{Name: "Giang"})
```

---

### 5.2 Generic kết hợp với abstraction

Ta có thể kết hợp **interface + generic**:

```go
type Saver[T any] interface {
    Save(T)
}

type MemoryRepository[T any] struct {
    data []T
}

func (m *MemoryRepository[T]) Save(item T) {
    m.data = append(m.data, item)
}
```

Sử dụng đa hình:

```go
func Process[T any](s Saver[T], item T) {
    s.Save(item)
}

repo := &MemoryRepository[string]{}
Process(repo, "hello")
```

Ở đây:

- `Saver[T]` → abstraction
- `MemoryRepository[T]` → implementation

---

## 6. Kết luận

Golang không đi theo OOP truyền thống, mà chọn một cách tiếp cận **đơn giản và linh hoạt hơn**:

| Tính chất OOP | Cách Go thực hiện |
| --- | --- |
| Encapsulation | Exported / unexported fields |
| Abstraction | Interface |
| Inheritance | Composition (embedding) |
| Polymorphism | Interface + dynamic dispatch |

Từ **Go 1.18**, generics giúp:

- Tăng khả năng tái sử dụng
- Giảm lặp code
- Kết hợp tốt với interface để tạo abstraction mạnh hơn