---
title: "Go Error Handle"
summary: ""
tags: [Go]
categories: [error-handle]
spotlight: false
draft: true
date: 2020-11-13T16:46:30+08:00
---

# Go Error Handling

Remember: errors are values!

Go 官方 blog 上的一个文章 [errors-are-values](https://blog.golang.org/errors-are-values)

[Go 1.13 Error](https://blog.golang.org/go1.13-errors)

关于新的几个特性，`As()`、`Is()`、`Unwrap()`, 其实就是我们常常使用的一个 struct 来封装一层 error，实现一个 error 链表。

## whether you wrap or not

