---
title: Fundamental Characteristics of OOP in Rust
author: phongthien99
date: 2024-12-12 01:17:00 +0800
categories: [Learn-rust]
tags: [rust]
math: true
media_subpath: '/posts/20180809'
---
# Fundamental Characteristics of OOP in Rust
Rust là một ngôn ngữ lập trình mạnh mẽ và an toàn, được thiết kế để mang lại hiệu suất cao và kiểm soát bộ nhớ mạnh mẽ mà không cần sự trợ giúp của garbage collector. Mặc dù Rust không phải là một ngôn ngữ hoàn toàn hướng đối tượng (OOP), nhưng nó vẫn hỗ trợ các tính năng của OOP thông qua các khái niệm như struct, trait và ownership. Trong bài viết này, chúng ta sẽ khám phá bốn tính chất cơ bản của lập trình hướng đối tượng trong Rust.

### 1. **Encapsulation (Đóng gói)**

Đóng gói là một trong những tính chất cốt lõi của lập trình hướng đối tượng, cho phép ẩn đi chi tiết nội bộ của đối tượng và chỉ cung cấp các phương thức hoặc giao diện công khai để tương tác với đối tượng đó. Trong Rust, điều này được thực hiện thông qua **visibility modifiers** và sự kết hợp giữa `struct` và `impl`.

- **Visibility modifiers**: Rust sử dụng các từ khóa như `pub` để kiểm soát phạm vi truy cập của các trường và phương thức. Mặc định, các trường trong `struct` là private và chỉ có thể được truy cập từ bên trong module nơi chúng được định nghĩa.
- **Impl blocks**: Cung cấp cách triển khai các phương thức để tương tác với dữ liệu được đóng gói.

```rust

struct Person {
    name: String,
    age: u32,
}

impl Person {
    // Phương thức tạo mới đối tượng Person
    pub fn new(name: String, age: u32) -> Self {
        Person { name, age }
    }

    // Phương thức công khai để lấy tên
    pub fn get_name(&self) -> &str {
        &self.name
    }

    // Phương thức công khai để tăng tuổi
    pub fn increment_age(&mut self) {
        self.age += 1;
    }

    // Phương thức riêng tư để kiểm tra nếu tuổi vượt quá giá trị cụ thể
    fn is_eligible_for_discount(&self) -> bool {
        self.age > 60
    }
}

fn main() {
    let mut person = Person::new("Alice".to_string(), 30);

    // Truy cập thông qua phương thức công khai
    println!("Name: {}", person.get_name());
    person.increment_age();

    // Không thể truy cập trực tiếp trường hoặc phương thức riêng tư
    // person.age = 40; // Lỗi: private field
    // println!("{}", person.is_eligible_for_discount()); // Lỗi: private method
}

```

Ở đây, các trường của `Person` được ẩn và chỉ có thể truy cập thông qua các phương thức được định nghĩa trong `impl`.

### 2. **Inheritance (Kế thừa)**

Mặc dù Rust không hỗ trợ kế thừa theo kiểu truyền thống của OOP (như trong Java hay C++), nhưng chúng ta vẫn có thể mô phỏng tính kế thừa bằng cách sử dụng `Deref`. `Deref` cho phép chúng ta chuyển hướng các phương thức và thuộc tính của một đối tượng sang đối tượng khác, tạo ra một cách tiếp cận linh hoạt để mô phỏng kế thừa.

Đầu tiên, chúng ta định nghĩa một cấu trúc `Person`, đại diện cho một người với trường `name` và một phương thức `greet` để trả về lời chào.

```rust
rust
Copy code
struct Person {
    name: String,
}

impl Person {
    fn greet(&self) -> String {
        format!("Hello, {}", self.name)
    }
}

```

Cấu trúc `Person` có một trường `name` và một phương thức `greet` để in ra lời chào với tên của người đó.

Tiếp theo, chúng ta tạo một cấu trúc `Employee`, trong đó bao gồm một trường `person` kiểu `Person`. Chúng ta sẽ sử dụng `Deref` để cho phép `Employee` sử dụng các phương thức của `Person`.

```rust

use std::ops::Deref;

struct Employee {
    person: Person,
    job_title: String,
}

impl Deref for Employee {
    type Target = Person;

    fn deref(&self) -> &Self::Target {
        &self.person
    }
}

fn main() {
    let employee = Employee {
        person: Person {
            name: "Alice".to_string(),
        },
        job_title: "Engineer".to_string(),
    };

    // Sử dụng phương thức greet từ Employee
    println!("{}", employee.greet());
}
```

Ở đây, chúng ta triển khai `Deref` cho `Employee` để chuyển hướng các lời gọi đến đối tượng `Person`. Điều này có nghĩa là, khi chúng ta gọi các phương thức như `greet()` từ `Employee`, Rust sẽ tự động gọi phương thức đó trên đối tượng `Person` bên tro

### 3. **Polymorphism (Đa hình)**

Polymorphism trong OOP cho phép các đối tượng khác nhau có thể sử dụng chung một giao diện mà không cần phải biết loại cụ thể của đối tượng. Trong Rust, polymorphism được thực hiện thông qua **trait objects** và **dynamic dispatch**.

Dưới đây là ví dụ về đa hình trong Rust:

```rust

trait Speak {
    fn speak(&self);
}

struct Dog;
struct Cat;

impl Speak for Dog {
    fn speak(&self) {
        println!("Woof!");
    }
}

impl Speak for Cat {
    fn speak(&self) {
        println!("Meow!");
    }
}

fn make_speak(speaker: &dyn Speak) {
    speaker.speak();
}

fn main() {
    let dog = Dog;
    let cat = Cat;

    make_speak(&dog); // Prints "Woof!"
    make_speak(&cat); // Prints "Meow!"
}

```

Ở đây, `make_speak` có thể nhận bất kỳ đối tượng nào thực hiện trait `Speak`, cho phép đa hình hoạt động mà không cần biết trước loại đối tượng.

### 4. **Abstraction (Trừu tượng hóa)**

Trừu tượng hóa giúp ẩn các chi tiết cài đặt và chỉ lộ ra giao diện cần thiết. Rust hỗ trợ trừu tượng hóa thông qua việc sử dụng trait và các kiểu dữ liệu như `Option` và `Result`, cho phép xử lý các tình huống mà không cần phải biết chi tiết cài đặt.

Ví dụ về trừu tượng hóa với `trait`:

```rust

trait Draw {
    fn draw(&self);
}

struct Circle;
struct Square;

impl Draw for Circle {
    fn draw(&self) {
        println!("Drawing a circle");
    }
}

impl Draw for Square {
    fn draw(&self) {
        println!("Drawing a square");
    }
}

fn draw_all(shapes: &Vec<Box<dyn Draw>>) {
    for shape in shapes {
        shape.draw();
    }
}

fn main() {
    let circle = Circle;
    let square = Square;

    // Bọc các đối tượng trong Box<dyn Draw> để sử dụng tính đa hình
    let shapes: Vec<Box<dyn Draw>> = vec![Box::new(circle), Box::new(square)];

    // Vòng lặp qua các đối tượng và gọi hàm `draw`
    draw_all(&shapes);
}

```

Trong ví dụ này, các chi tiết triển khai cụ thể như cách vẽ hình tròn hay hình vuông được ẩn đi. Thay vào đó, ta chỉ làm việc với giao diện `draw`, giúp mã nguồn trở nên rõ ràng và dễ bảo trì. Thông qua việc bọc đối tượng trong `Box<dyn Draw>`, chúng ta có thể lưu trữ và xử lý các kiểu dữ liệu khác nhau sử dụng cùng một giao diện chung mà không cần quan tâm đến kiểu cụ thể.

### Kết luận

Rust cung cấp đầy đủ các tính chất của lập trình hướng đối tượng thông qua các cấu trúc như `struct`, `trait` và các tính năng mạnh mẽ của hệ thống kiểu. Mặc dù Rust không hoàn toàn hướng đối tượng như một số ngôn ngữ khác, nhưng những khái niệm trên giúp chúng ta xây dựng các chương trình dễ bảo trì, tái sử dụng và mở rộng. Nếu bạn muốn làm việc với OOP trong Rust, các tính năng này cung cấp sự linh hoạt để giải quyết các vấn đề lập trình với phong cách OOP trong khi vẫn giữ được các lợi ích của Rust về hiệu suất và an toàn bộ nhớ.