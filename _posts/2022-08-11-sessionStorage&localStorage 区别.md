---
layout: post
title: "sessionStorage & localStorage 的区别"
subtitle: '介绍 sessionStorage 和 localStorage 的异同'
author: "Vascent"
header-img: "//w.wallhaven.cc/full/kx/wallhaven-kxkjk6.jpg"
tags:
  - JavaScript
  - 基础
---

# sessionStorage & localStorage

`sessionStorage`和`localStorage`都是HTML5中提供的Web存储API，它们可以用于在浏览器中存储数据。

它们的主要区别在于：

- `localStorage`中的数据不会过期，而`sessionStorage`中的数据在页面会话结束时被清除。
- `sessionStorage`存储的数据只在当前会话（当前窗口或标签页）中有效，而`localStorage`存储的数据在所有同源的窗口和标签页中都有效。

### `sessionStorage`

- 每当在浏览器的某个选项卡中加载文档时，都会创建**`一个唯一`**的页面会话并分配给该特定选项卡。该页面会话仅对该特定选项卡有效。
- 页面会话持续的时间与选项卡或浏览器打开的时间一样长，并且在页面重新加载和恢复期间保持。
- 在新选项卡或窗口中打开页面会话会创建具有顶级浏览上下文值的 `new session`，这与会话cookie的工作方式不同。
- 使用相同URL打开多个`选项卡/窗口`，会为每个`选项卡/窗口创建sessionStorage`。
- `复制`选项卡会将选项卡的`sessionStorage复制`到新选项卡中。
- 关闭`选项卡/窗口`会结束会话并清除sessionStorage中的对象。
