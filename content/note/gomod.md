---
title: "Go Modules"
summary: ""
tags: [Go]
categories: [dependency-management]
spotlight: false
draft: true
date: 2020-11-12T19:19:16+08:00
---

# Go Modules

参考 Go blogpost [using-go-modules](https://blog.golang.org/using-go-modules).

## 两个 version

### [Pseudo-versions](https://golang.org/cmd/go/#hdr-Pseudo_versions) VS [semantic versions](https://research.swtch.com/vgo-import)

前者为依赖 repo 没有打 tag, 则按照一定的规则给一个 version。

后者为一种 version 规则。

```go
// 可以指定 version go get.
go get rsc.io/quote@v3.0.0
```

## 下载和校验 [Module downloading and verification ](https://golang.org/cmd/go/#hdr-Module_downloading_and_verification)

一个 GOPROXY, 一个 GOSUM.

GOPROXY 用来代理。

GOSUM 用来校验，确保开发的时候 fetch 的和运行或者构建的时候 fetch 的是同一个。

## major version 的升级

[semantic import versioning](https://research.swtch.com/vgo-import)

为了支持开发更新版本时可以一步步来，不同 major version 的依赖被看作不同的依赖，使用不同的 path.

例如

```go
import (
    "rsc.io/quote"
    quoteV3 "rsc.io/quote/v3"
)
```

## remove 不需要的依赖

`go mod tidy`