---
title: Go 与 Python3 函数传参
date: 2020-11-29T14:31:07+08:00
draft: false
comments: false
images:
---

## Pass by what？

Go pass arguments by value. No primitve to pass it by reference.[^1]

也就是 Go 是传值，就像 C 一样，但是需要注意的是在 Go 里面，很多东西都是值，例如：Slice、Array.

Python pass argument by assignment.[^2]

看上去有些疑惑🤔，啥叫通过赋值传参数呢？其实和传值是一样的。

C++ can pass argument by reference.

C++ 可以选择引用做为参数传递，也就是在函数定义时使用 `&` 符号，这个传指针效果差不多，但是有本质区别，传指针还是传值，不过这个值是指针，但是传引用可以说是传了相同的对象。

### Go: Pass by value.

```go
package main

import (
  "fmt"
  "reflect"
  "unsafe"
)

func main() {
  m := make(map[int]int)
  m[0] = 0
  changeMap(m)
  fmt.Println(m[0])
  // what is m[0]?
  s := make([]int, 1)
  changeSlice(s)
  fmt.Println(s)
  // what is s?
  s = make([]int, 1, 2)
  appendSlice(s)
  fmt.Println(s)
  // what is s?
  hdr := (*reflect.SliceHeader)(unsafe.Pointer(&s))
  data := *(*[2]int)(unsafe.Pointer(hdr.Data))
  fmt.Println(data)
  // what is data
}

func changeMap(m map[int]int) {
  m[0] = 1
}

func changeSlice(s []int) {
  s[0] = 1
}

func appendSlice(s []int) {
  s = append(s, 1)
}
```

输出为：

```bash
1
[1]
[0]
[0 1]
```

1. 能改变 map 是因为 map 实际上是一个指向底层 structure 的指针，所以虽然是传值，但是 map 对应的值就是一个指针，所以表现的和引用差不多。

   > *A map value is a pointer to a* `runtime.hmap` *structure.*[^5]

2. 能改变 slice 中的某个值是因为，slice 底层是一个 structure, 有一个字段是一个指向 array 的指针（不一定是 array 的开头，可能是中间），所以虽然是传值，不过底层的字段是指针，改变的还是原来的数据。

3. 不能 append slice 是因为虽然可以改变底层的 array，但是 slice 有一个 len 字段，这个字段没有改变，slice 依旧没有改变。

   ⚠️ 这里使用的是 capacity 为 2 的 slice, 如果 capacity 小于 2 的话，`append()` 会创建一个新的底层 array. 这样外面的 slice 还是不会改变，并且两个 slice 指向了不同的值。

4. 如上面的⚠️所说，虽然改变了 array，但是没有改变 slice 的其他字段，slice 依旧没有扩充。而底层的 array 却已经变成了 `[0 1]`.

### Python: Pass by assignment.

需要明确在 Python 中，变量实际上是一名字，是一个对 object 的引用，也就是所谓的万物皆对象。

而对象有一个非常重要的性质：是 **mutable** 还是 **immutable**, 如果你改变一个不可变的对象 [^3] 例如 int, 实际上是创建了一个新的对象。

```python
>>> num = 1
>>> id(num)
4564023856
>>> num = 2
>>> id(num)
4564023888
```

> id(object)
>
> Return the “identity” of an object.[^4]

而 list 是一个可变的（mutable）对象。所以我们赋值获取了一个新的对象引用，改变的是一个对象。

而函数传参不过是给参数赋值，也就是创建了另一个对对象的引用。

```python
>>> def change_list(l):
...     l[0] = 1
...
>>> def change_num(n):
...     n = 100
...
>>> l = [0]
>>> change_list(l)
>>> l
[1]
>>> n = 88
>>> change_num(n)
>>> n
88
```

如上面的代码，list 可变，所以能够在函数内改变他的值，number 不可变，所以改变他的值是创建了另一个对象，原来的对象依旧没变。

## *Reference*

[^1]: https://golang.org/doc/faq#pass_by_value
[^2]: https://docs.python.org/3/faq/programming.html#how-do-i-write-a-function-with-output-parameters-call-by-reference

[^3]: https://docs.python.org/3/glossary.html#term-immutable
[^4]: https://docs.python.org/3/library/functions.html#id
[^5]: https://dave.cheney.net/2017/04/30/if-a-map-isnt-a-reference-variable-what-is-it#:~:text=Maps%2C%20like%20channels%2C%20but%20unlike,value%20in%20a%20Go%20program.
