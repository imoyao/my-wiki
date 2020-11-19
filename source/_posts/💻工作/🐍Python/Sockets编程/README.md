---
title: Python 中的 Socket 编程
toc: true
tags:
  - sockets
categories:
  - "\U0001F4BB 工作"
  - "\U0001F40DPython"
  - 网络编程
date: 2020-10-01 10:16:53
---

## 说明

本书翻译自 [realpython](https://realpython.com/) 网站上的文章教程 [Socket Programming in Python \(Guide\)](https://realpython.com/python-sockets/)，由于原文很长，所以整理成了 Gitbook 方便阅读。你可以去 [首页](https://legacy.gitbook.com/book/keelii/socket-programming-in-python-cn/details) 下载 [PDF](https://legacy.gitbook.com/download/pdf/book/keelii/socket-programming-in-python-cn)/[Mobi](https://legacy.gitbook.com/download/mobi/book/keelii/socket-programming-in-python-cn)/[ePub](https://legacy.gitbook.com/download/epub/book/keelii/socket-programming-in-python-cn) 格式文件或者 [在线阅读](https://keelii.gitbooks.io/socket-programming-in-python-cn/content/)

## 原作者

> Nathan Jennings 是 Real Python 教程团队的一员，他在很早之前就使用 C 语言开始了自己的编程生涯，但是最终发现了 Python，从 Web 应用和网络数据收集到网络安全，他喜欢任何 Pythonic 的东西
> —— realpython

## 译者注

[译者](https://keelii.com/) 是一名前端工程师，平常会写很多的 JavaScript。但是当我使用 JavaScript 很长一段时间后，会对一些 _语言无关_ 的编程概念感兴趣，比如：网络 /socket 编程、异步 / 并发、线 / 进程通信等。然而恰好这些内容在 JavasScript 领域很少见

因为一直从事 Web 开发，所以我认为理解了网络通信及其 socket 编程就理解了 Web 开发的某些本质。过程中我发现 Python 社区有很多我喜欢的内容，并且很多都是高质量的公开发布且开源的

最近我发现了这篇文章，系统地从底层网络通信讲到了应用层协议及其 C/S 架构的应用程序，由浅入深。虽然代码、API 使用了 Python 语言，但是底层原理相通。非常值得一读，推荐给大家

另外，由于本人水平所限，翻译的内容难免出现偏差，如果你在阅读的过程中发现问题，请毫不犹豫的提醒我或者开新 [PR](https://github.com/keelii/socket-programming-in-python-cn/pulls)。或者有什么不理解的地方也可以开 [issue](https://github.com/keelii/socket-programming-in-python-cn/issues) 讨论。当然 star 或者 [支持](https://user-images.githubusercontent.com/458894/32358969-179e7a28-c085-11e7-882a-485164168f74.png) 也是欢迎的

## 授权

本文（翻译版）通过了 realpython 官方授权，原文版权归其所有，任何转载请联系他们。翻译版遵循本站 [许可证协议](https://keelii.com/about/#license)

## 原文链接
[[译]Python 中的 Socket 编程（指南）](https://keelii.com/2018/09/24/socket-programming-in-python/#%E5%BC%95%E7%94%A8)

