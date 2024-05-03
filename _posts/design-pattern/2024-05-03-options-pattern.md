---
title: Options Pattern
author: phongthien99
date: 2024-05-03 00:40:00 +0800
categories: [Design-pattern]
tags: [interview]
math: true
media_subpath: '/posts/20240503'
---

# Options Pattern

## Intent

- helps solve the problem when a function or constructor has too many parameters, making calling the function or constructor complicated and confusing.
- instead of passing a series of parameters, we create an object containing optional parameters (options object), from which we can set the values for those parameters dynamically.

## Problem

- calling a function or constructor with too many parameters can result in code that is difficult to read, verbose, and confusing.
- additionally, when we need to change parameters, we must modify each function or constructor call.

## Solution

- Step 1:  create an object containing options parameters (options object).
- Step 2: define methods in the options object to set the values for those parameters
- Step 3:  When calling a function or constructor, instead of passing a series of parameters, we pass in the options object that was previously set.
- Step 4: In the function or constructor, we use the value set in the options object to perform the necessary tasks.
- Step 5: When calling a function or constructor, instead of passing a series of parameters, we pass in the options object that was previously set.

## Example

```go
package main

// step 1 create an object containing options parameters (options object).
type Config struct {
	URL                  string
	DocExpansion         string
}

// Step 2: define methods in the options object to set the values for those parameters
func URL(url string) func(*Config) {
	return func(c *Config) {
		c.URL = url
	}
}

func DocExpansion(docExpansion string) func(*Config) {
	return func(c *Config) {
		c.DocExpansion = docExpansion
	}
}

// Step 3:  When calling a function or constructor, instead of passing a series of parameters, we pass in the options object that was previously set.
func newConfig(configFns ...func(*Config)) *Config {
	config := Config{
	}
 // Step 4: In the function or constructor, we use the value set in the options object to perform the necessary tasks.
	for _, fn := range configFns {
		fn(&config)
	}

	return &config
}
// Step 5: When calling a function or constructor, instead of passing a series of parameters, we pass in the options object that was previously set.
func main() {
	r := newConfig(URL("http://test.com"),DocExpansion("test"))

}

```

## **Pros and Cons**

| Pros |  Cons |
| --- | --- |
| Flexible and extensible: Option Pattern allows adding or changing optional parameters without changing the structure of the function or constructor. This makes the code more extensible and flexible.
 | Complexity with large number of options: When there are too many options, managing and using the Option Pattern can become complicated. This can lead to defining too many constructors and optional setters, making the code complex and confusing. |
| Ease of reading and maintenance: When using the Option Pattern, function or constructor calls become more concise and easier to read. Instead of having a bunch of parameters, we just need to pass an options object, making the code easy to maintain. | Unclear with required options: If there are parameters that are optional but necessary for use, using the Option Pattern can make the code unclear. In this case, using the Option Pattern may not be appropriate and one should consider other methods such as using default parameters or structured data types. |
| Minimize errors: Option Pattern helps reduce the number of errors caused by confusion about the order or data type of parameters. Instead of having to remember the order or data type of each parameter, we just need to set the values for the fields in the options object. |  |

## Conclusion

Although the Option Pattern offers many benefits in terms of flexibility and readability, it also requires careful consideration when used, especially for cases with a large number of options.