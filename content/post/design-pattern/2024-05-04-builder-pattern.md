---
title: Builder Pattern
author: phongthien99
date: 2024-05-04 00:40:00 +0800
categories: [Design-pattern]
tags: [interview]
math: true
media_subpath: '/posts/20180809'
---
# Builder pattern

## Intent

- **Builder** is a creational design pattern that lets you construct complex objects step by step. The pattern allows you to produce different types and representations of an object using the same construction code

## **Problem**

- When an object has too many properties or optional properties, creating a constructor with all the parameters or using multiple constructors can become complex and difficult to maintain..

## Solution

The Builder pattern suggests that you extract the object construction code out of its own class and move it to separate objects called *builders*.

- Director: Responsible for building an object using Builder.
- Builder: Interface or abstract class defines the steps necessary to construct an object.
- ConcreteBuilder: Concretizes the Builder, implements specific steps to build the object and provides a method to retrieve the final object.
- Product: Represents the built object.

## Example

```go
package main

import "log"

// Product
type User struct {
	Name string
	Age  int
}

// ConcreteBuilder
type UserBuilder struct {
	Name string
	Age  int
}

// Director
type IDiretor[T any] interface {
	Builder() T
}

// Builder
type IUserBuilder interface {
	SetName(name string) IUserBuilder
	SetAge(age int) IUserBuilder
	IDiretor[*User]
}

func (u *UserBuilder) SetName(name string) IUserBuilder {
	u.Name = name
	return u

}

func (u *UserBuilder) SetAge(age int) IUserBuilder {
	u.Age = age
	return u

}

func (u *UserBuilder) Builder() *User {
	return &User{
		Name: u.Name,
		Age:  u.Age,
	}

}
func NewUserBuilder() IUserBuilder {
	return &UserBuilder{}
}

func main() {
	user := NewUserBuilder().SetName("test").SetAge(12).Builder()

	log.Print(user)

}

```

## **Pros and Cons**

 ✅ You can construct objects step-by-step, defer construction steps or run steps recursively.

 ✅ You can reuse the same construction code when building various representations of products.

 ✅ *Single Responsibility Principle*. You can isolate complex construction code from the business logic of the product.

❌ The overall complexity of the code increases since the pattern requires creating multiple new classes.

## Conclusion

Although the Option Pattern offers many benefits in terms of flexibility and readability, it also requires careful consideration when used, especially for cases with a large number of options.