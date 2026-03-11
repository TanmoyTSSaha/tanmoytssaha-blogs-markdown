---
title: Getting started with Go!
slug: getting-started-with-go
date: 2026-03-11
author: Tanmoy Saha
tags:
  - series
  - golang
  - learnings
  - programming
  - language
description: Kickoff to my Golang learning series—how I’m learning Go and a practical walk-through of core data types and values.
reading_time: 10
draft: false
series: Golang Learnings
series_slug: golang-learnings
series_order: 1
---

## What is this series is about and what will we cover in this series?

This series is all about my learnings journey about Golang. How I dive deep to learn go. How I'm approaching to solve a problem using go. During learning what mistakes I'm making and what you can learn from these journey.

## What am I following to learn Golang?

I'm following Joseph Abah's Udemy course to learn the Golang - [Course link](https://www.udemy.com/share/10erQb3@SP0T2PAYT_-cD9BLdmyDz3pupf7fTKNQPlfUFbAGzBFgpzVDRKHjgI-QnVMuVkRQ/). And I'll be following other sources too which I'll mention inside the blog.

*So let's start with the blog!*

## Core language Fundamentals

### Section 1: Understanding values in Go.
We have many types of values inside Go like string, integer, float, boolean, array, complex.
Let's try defining each variable here - 
`var stringType string = "Hey I am string"`
`var intType int = 7` %% Thala for a reason %%
`var floatType float = 3.14`
`var booleanType bool = true`
`var arrayType [7]int = [7]int{1,2,3,4,5,6,7}`
`var complexType complex64 = complex(6,9)` or `var complexType complex64 = complex64(6+9i)`

Now let's try to compile these variables in one program -

```
package main

import (
	"fmt"
)

func main() {
	// DECLARING VARIABLES
	var stringType string = "Hey I am string"
	var intType int = 7    // Thala for a reason
	var floatType float32 = 3.14
	var booleanType bool = true
	var arrayType [7]int = [7]int{1,2,3,4,5,6,7}
	var complexType complex64 = complex(6,9)
	
	fmt.Printf("%s\n", stringType)
	fmt.Printf("%d%\n", intType)
	fmt.Printf("%.2f\n", floatType)
	fmt.Printf("%t\n", booleanType)
	fmt.Printf("%+v\n", arrayType)
	fmt.Printf("%v\n", complexType)
}
```

Terminal output -
```
Hey I am string
7
3.14
true
[1 2 3 4 5 6 7]
(6+9i)
```

Let's dive deep into each variables. Crafted this part using [Geeksforgeeks blog](https://www.geeksforgeeks.org/go-language/data-types-in-go/) & Gemini.

##### 1. Integer - 
In Golang we have 3 types of sub-categories of numbers, integer is one of these 3 types. Followings are the types of integer - 

| **Data type**   | **Range**                                                                | **Description**                                                                                                       |
| --------------- | ------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------- |
| int8            | -128 to 127                                                              | 8-bit signed integer                                                                                                  |
| int16           | -32768 to 32767                                                          | 16-bit signed integer                                                                                                 |
| int32 (or rune) | -2147483648 to 2147483647                                                | 32-bit signed integer                                                                                                 |
| int64           | -9223372036854775808 to 9223372036854775807                              | 64-bit signed integer                                                                                                 |
| uint8 (or byte) | 0 to 255                                                                 | 8-bit unsigned integer                                                                                                |
| uint16          | 0 to 65535                                                               | 16-bit unsigned integer                                                                                               |
| uint32          | 0 to 4294967295                                                          | 32-bit unsigned integer                                                                                               |
| uint64          | 0 to 18446744073709551615                                                | 64-bit unsigned integer                                                                                               |
| int             | -2147483648 to 2147483647 or -9223372036854775808 to 9223372036854775807 | Depending operating system it can be 32 bit or 64 bit                                                                 |
| uint            | 0 to 4294967295 or 0 to 18446744073709551615                             | Depending operating system it can be 32 bit or 64 bit                                                                 |
| uintptr         |                                                                          | Large enough to hold the bit pattern of any pointer. Usually 32 bits on 32-bit systems and 64 bits on 64-bit systems. |
Let's do a fun experiment here (*Found after searching about it on Gemini*)
The fun fact is we do not need to memorize this huge numbers. We can fetch these number using a Go package called `math`. Trying it programmatically - 
```
package main

import (
	"fmt"
	"math"
)

func main() {
	fmt.Println("Max int8: ", math.MinInt8, " to ", math.MaxInt8) // This is how we do string concatenation
	fmt.Println("Max uint16: ", 0, " to ", math.MaxUint16)
}
```

Terminal output - 
```
Max int8:  -128  to  127
Max uint16:  0  to  65535
```

##### 2. Float -
Float is second type out of 3 sub-categories. We have two types of floating-point numbers in Golang. 

| **Data type** | **Bits** | **Min Value (approx)**  | **Max Value (approx)**      | Decimal Precision |
| ------------- | -------- | ----------------------- | --------------------------- | ----------------- |
| float32       | 32       | 1.4 X 10<sup>-45</sup>  | 3.4028235 X 10<sup>38</sup> | ~7 digits         |
| float64       | 64       | 4.9 X 10<sup>-324</sup> | 1.8 X 10<sup>308</sup>      | ~15 digits        |
There are 3 different ways we can write out floating-point numbers in Golang. Here are these - 
**1. Decimal Literals** - This is the standard way of writing numbers like 3.14, .5, 10. (Go stores these as float64)
**2. Exponential Literals -** This is "Scientific notation" used for very large or very small number so we do not have to deal with dozens of zeros. e.g. `12e18`: 12 X 10<sup>18</sup>  (*The `e` can be lowercase or uppercase*) and `1e-3` is 0.0001 (*negative*).
**3. Mixed literals -** The combination of decimal & exponential. e.g. `13.16e12`: 12.15 X 1000000000000.

##### 3. Complex - 
There are 2 types of complex number we have in Golang. Complex numbers are built using float32 or float64 respectively.

| **Data type** | **Components** | **Description**                                            |
| ------------- | -------------- | ---------------------------------------------------------- |
| complex64     | Two float32    | 32 bits for the real part, 32 bits for the imaginary part. |
| complex128    | Two float64    | 64 bits for the real part, 64 bits for the imaginary part. |
There are some built-in-functions in Golang for complex numbers - 
- complex - It constructs a complex number. Both arguments must be of same floating-point type.
- real(c) - It extracts the real component as float from the complex number.
- imag(c) - It extracts the imaginary component as float from the complex number.

This the end of this section. I will extend this with Section 2: Variables next.

*Till then enjoy your life and happy engineering!*