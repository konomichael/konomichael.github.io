+++
title = "Understanding Pointers and the Unsafe Package in Go"
date = "2019-02-16T15:17:28+08:00"
author = "Michael Chan"
authorTwitter = "konomichael_" #do not include @
cover = ""
tags = ["golang"]
keywords = ["golang"]
description = ""
showFullContent = false
readingTime = true
hideComments = false

+++

Go is a popular programming language known for its simplicity, concurrency, and fast compile times. It features built-in support for pointers, which are variables that store the memory address of another variable. Pointers can be used to manipulate data more efficiently and can enable more advanced programming techniques, such as passing data by reference.

In addition to its built-in support for pointers, Go also provides an `unsafe` package that allows developers to bypass some of the safety checks that the language provides. This package can be used to perform low-level memory operations that are not possible using the safe language features.

In this blog post, we'll explore how pointers and the `unsafe` package work in Go by examining a code snippet that uses both features.

<!--more-->

```go
func main() {
	bs := make([]byte, 0, 10)
	bs = append(bs, 'h')
	bs = append(bs, 'e')
	bs = append(bs, 'l')
	bs = append(bs, 'l')
	bs = append(bs, 'o')

	p := &bs

	n := uintptr(8)
	lp := (*int)(unsafe.Pointer(uintptr(unsafe.Pointer(p)) + n))
	cp := (*int)(unsafe.Pointer(uintptr(unsafe.Pointer(p)) + 2*n))
	fmt.Printf("the length of slice is %d\n", *lp)
	fmt.Printf("the capacity of slice is %d\n", *cp)

	firstp := (*byte)(unsafe.Pointer(*(*uintptr)(unsafe.Pointer(p))))
	fmt.Printf("the first byte of slice is %c\n", *firstp)
}

```

Let's break down what this code does, line by line:

```go
bs := make([]byte, 0, 10)
```

This line creates a byte slice with length 0 and capacity 10. The `make` function is used to allocate memory for the slice, and the slice is assigned to the `bs` variable.

```go
bs = append(bs, 'h')
bs = append(bs, 'e')
bs = append(bs, 'l')
bs = append(bs, 'l')
bs = append(bs, 'o')
```

These lines append the bytes 'h', 'e', 'l', 'l', and 'o' to the `bs` slice, using the `append` function. The result is a slice with length 5 and capacity 10.

```go
p := &bs
```

`p := &bs` creates a pointer `p` that points to the memory address of the `bs` slice, then `n := uintptr(8)` creates a `uintptr` variable `n` with a value of 8.

```go
lp := (*int)(unsafe.Pointer(uintptr(unsafe.Pointer(p)) + n))
cp := (*int)(unsafe.Pointer(uintptr(unsafe.Pointer(p)) + 2*n))
```

These lines create two pointers, `lp` and `cp`, that point to the memory addresses of the slice length and capacity values, respectively. This is done using the `unsafe.Pointer` function to convert the `p` pointer to a `uintptr` value, adding an offset to the pointer using the `+` operator, and then converting the result back to a pointer of type `*int`.

```text
           p         p+8       p+16
           ┌─────────┬─────────┬─────────┐
           │(uintptr │  (int)  │  (int)  │
SliceHeader│  Data   │  Len    │  Cap    │
           └─┬───────┴─────────┴─────────┘
             │
             │   ┌───┬───┬───┬───┬───┬───┐
             └───► h │ e │ l │ l │ o │...│
                 └───┴───┴───┴───┴───┴───┘
```

The first `unsafe.Pointer` call converts the `p` pointer to a `uintptr` value, and the second `unsafe.Pointer` call converts it back to a pointer of type `unsafe.Pointer`. This is necessary because the `uintptr` value cannot be directly converted to a pointer of type `*int`.

The first pointer, `lp`, has an offset of 8 from the start of the `bs` slice, which points to the length value held by `sliceHeader`. The second pointer, `cp`, has an offset of 16, which points to the capacity value held by `sliceHeader`.

```go
fmt.Printf("the length of slice is %d\n", *lp)
fmt.Printf("the capacity of slice is %d\n", *cp)
```

These lines print the length and capacity values of the `bs` slice, which are accessed through the `lp` and `cp` pointers.

```go
firstp := (*byte)(unsafe.Pointer(*(*uintptr)(unsafe.Pointer(p))))
fmt.Printf("the first byte of slice is %c\n", *firstp)
```

This line creates a pointer `firstp` that points to the first byte of the `bs` slice. This is done by first converting the `p` pointer to a `uintptr` value(which is the address of `data` held by `sliceHeader`), then dereferencing it to access the address, and finally converting the result to a `uintptr` value and dereferencing it again to access the first byte of the slice.

The first byte is printed using the `%c` format specifier, which formats the value as a character.

<br/>

In summary, the code we examined demonstrates how to use pointers and the `unsafe` package in Go to perform low-level memory operations. While the `unsafe` package can be useful in certain situations, it should be used with caution, as it bypasses some of the safety checks that Go provides. When working with pointers and the `unsafe` package, it is important to understand how memory is allocated and managed in Go, and to follow best practices for memory management and data manipulation.
