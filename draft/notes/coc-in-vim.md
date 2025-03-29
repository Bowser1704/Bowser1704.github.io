---
title: "使用 coc.nvim"
summary: “在 vim 中像 vscode 一样 使用 LSP "
tags: [vim]
categories: [tools]
spotlight: false
date: 2020-06-26T11:12:15+08:00
---

## 在 vim 中像 vscode 一样 使用 LSP 

什么是 [LSP](https://nshipster.com/language-server-protocol/)，我们在 IDE 中使用的「自动补全」、「代码高亮」、「自动缩进」....，都依赖于 Language Server，这个时候需要使用 LSP，在 vim 中我们可以增加插件来实现一个 LSP Client。

> LSP 相当于 HTTP，Language Server 相当于 Web Server，LSP Client 相当于 Web Client。

### 在 vim 中使用 插件

使用 [vim-plug](https://github.com/junegunn/vim-plug)，一个 vim plugin manager。

**安装方式参考官网**

安装成功之后我们需要修改 vim configure, 一般为 ~/.vimrc。添加下面的配置

```sh
call plug#begin()
" 下面的命令是 安装插件。
" Plug 'tpope/vim-sensible' 
call plug#end()
```

之后就可以正常使用了。

### 安装 coc.nvim

[coc.nvim](https://github.com/neoclide/coc.nvim)，是一个提供完整 LSP 客户端支持的 vim 插件，并且可以安装插件，功能十分强大。

**安装方式参考官网**

安装成功之后 就可以使用很多 coc 插件。

