---
layout: post
title: 从HelloWorld出发到理解微服务框架
subtitle: 程序是如何执行的
date: 2021-06-12
author: HuK
header-img: img/post-bg-universe.jpg
catalog: true
tags:
  - 程序执行
  - 抽象分层
---

## 前言

从 HelloWorld 出发迈向微服务架构，意味着从简单的单一功能过度到复杂的分布式系统。本文将从执行流、技术栈、日志和监控三个角度出发简单的梳理一下二者的异同

## 正文

### 执行流

#### HelloWorld 执行顺序

顺序执行：

线性执行：HelloWorld 程序从上到下顺序执行，没有分支或并行处理。
确定性：每次执行的步骤和结果是确定的，输入和执行顺序不变。
简单流程：例如，Python 代码：

def main():
print("Hello, World!")
if _name_ == "_main_":
main()
