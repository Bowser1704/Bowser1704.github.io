---
title: "异常与错误"
date: 2019-11-09T22:28:25+08:00
draft: true
categories:
- cs
tags:
- language
- error
description: 不同语言对于异常与错误的处理
---

## 概念

在Go语言中，错误被认为是一种可以预期的结果；而异常则是一种非预期的结果

异常：不可拯救的，一出问题，应该马上崩溃，是不可以预计的错误，例如内存不够了，内存溢出。

错误：可以挽救的，出现错误后应该是进行检查，而不是直接返回。让程序继续运行。因为返回的是可以预期的错误中的。

## go语言的机制 与C语言类似。

基本上都是错误，但是你可以用recover捕获异常，并且处理，但是一般都是不断地error判断，十分ugly。并且丢失了真正的错误，和错误的一定信息。

利用recover可以捕获panic机制，但是必须要隔==一个==栈帧。

```go
func MyRecover() interface{} {
    return recover()
}

func main() {
    // 可以正常捕获异常
    defer MyRecover()
    panic(1)
}

//相邻的栈帧，隔0个
func main() {
    // 无法捕获异常
    defer recover()
    panic(1)
}

//隔多个栈帧
func main() {
    defer func() {
        // 无法捕获异常
        if r := MyRecover(); r != nil {
            fmt.Println(r)
        }
    }()
    panic(1)
}

func MyRecover() interface{} {
    log.Println("trace...")
    return recover()
}
```

## python3的机制

错误：语法或者逻辑上的错误

异常：该处理的，在正常流之外的处理机制。