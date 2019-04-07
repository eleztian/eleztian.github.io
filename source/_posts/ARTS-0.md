---
title: ARTS -0-
date: 2019-03-17 21:38:34
tags: ARTS
categories: "ARTS"
---
| ARTS       | Content             |
| ---------- | ------------------- |
| Algorithm  | Stack、Heap、Queue  |
| Review     | 如何使用好超时控制? |
| Tip/Techni | hystrix-go          |
| Share      | 熔断器理解实现      |
<!-- more -->

> ARTS是什么？
- Algorithm：每周至少做一个leetcode的算法题；
- Review：阅读并点评至少一篇英文技术文章；
- Tip/Techni：学习至少一个技术技巧；
- Share：分享一篇有观点和思考的技术文章;

---

## Algorithm
[Stack、Heap、Queue](http://eleztian.top/2019/03/18/Abstract-Stack-And-Queue/)

## Review

[Context Deadlines and How to Set Them](https://engineering.grab.com/context-deadlines-and-how-to-set-them)

### 文章要点
- 什么是Timeout
- Timeout的作用，以及在微服务中的分析
- 如何设置一个合适的Timeout
- 在Go中的使用及代码示例

### 个人理解
调用超时后，服务一般有以下行为：
- Return Error
- Return fallback
- Retry

如何设置一个合理的超时时间：
![timeout](https://engineering.grab.com/img/context-deadlines-and-how-to-set-them/image3.jpg)
如图，一个不合理的超时时间会导致计算资源泄露，由于从B到A的调用超时，A（或A的客户端）可能会重试，导致B上的负载增加。这反过来导致C上的负载增加，最终所有服务都将停止响应。这就是常见的级联故障。
一个合格的超时时间应该根据服务提供方所提供的SLA(service-level agreements)来设定。

## Tip/Techni
一个优秀的熔断器实现(go)： [hystrix-go](https://github.com/afex/hystrix-go)

hystrix-go旨在帮助Gopher轻松构建一个具有基于Java的Hystrix库的类似执行语义的应用程序。

## Share

[Circuit Breakers](/2019/03/24/Circuit-Breakers/)