---
title: "Go concurrency"
summary: ""
tags: [Go]
categories: [concurrency]
spotlight: false
draft: true
date: 2020-11-13T19:45:02+08:00
---

> 思考这样一个问题：我们有 100 个 Piece 要下载，有 10 个 Peer 可以连接。那么怎样并发地去请求这 10 个 Peer？怎样分配这 100 个 Piece？当某个 Peer 无法连接时，怎样将 Piece 转交给别的 Peer？下载好的结果怎样汇总？出现了错误怎样上报？怎样等待下载全部完成？

