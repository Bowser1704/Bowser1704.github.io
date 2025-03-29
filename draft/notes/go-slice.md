---
title: "Go Slice"
date: 2020-11-23T18:28:28+08:00
draft: true
comments: false
images:

---

# Something about Go slice

## Array

First, array.

> Go's arrays are values. An array variable denotes the entire array; it is not a pointer to the first array element (as would be the case in C).
>
> — [slices-intro](https://blog.golang.org/slices-intro)[^0]

In Go:

```go
arr1 := [2]int{1, 2}
arr2 := [...]int{1, 2}
// arr1 == arr2
// true or false
```

> Can you name four places where three dots (...) are used in Go?
>
> — [3 dots in 4 places](https://yourbasic.org/golang/three-dots-ellipsis/)[^1]

## Slice

A slice that refers to an array.

- Garbage collections.

  When the gc will release the underlying array. [A possible "gotcha”](https://blog.golang.org/slices-intro)[^0]

  using copy function to create a new slice which refers to a new array.

### Slice expressions

> Slice expressions construct a substring or slice from a string, array, pointer to array, or slice. 
>
> — [Slice expressions ](https://golang.org/ref/spec#Slice_expressions)[^2]

```go
a := [5]int{1, 2, 3, 4, 5}	// array
t := a[1:4:4]			// change max value from 4 to 5?
t = append(t, 99) // resize the slice t, create a new underlying array.
t[1] = 88
fmt.Println(t)	// [2 88 4 99]
fmt.Println(a)	// [1 2 3 4 5]
```

### Making slices, maps and channels

> The built-in function `make` takes a type `T`, which must be a slice, map or channel type, optionally followed by a type-specific list of expressions. It returns a value of type `T` (not `*T`).
>
> — [Making slices, maps and channels](https://golang.org/ref/spec#Making_slices_maps_and_channels)[^3]

```go
s := make([]int, 1<<63)         // illegal: len(s) is not representable by a value of type int
```

### Pass by value

>  but for instance to change the length of a slice in a method the receiver must still be a pointer

## *Reference*

[^0]: https://blog.golang.org/slices-intro

[^1]: https://yourbasic.org/golang/three-dots-ellipsis/

[^2]: https://golang.org/ref/spec#Slice_expressions

[^3]: https://golang.org/ref/spec#Making_slices_maps_and_channels

