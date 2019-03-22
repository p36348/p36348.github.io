---
layout:     post
title:      "RxSwift, MVVM(待完成)"
subtitle:   " 工作期间项目中RxSwift的应用"
date:       2018-12-01 12:00:00
author:     "P36348"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - iOS
    - RxSwift
    - Swift
    - ReactiveX
    - 响应式编程
    - MVVM
    - MVC
---

## 前言

以目前主流项目的功能需求,MVC模式下的C都很难避免臃肿的情况.
在稍复杂的页面上,如果不想办法把C瘦身,会让C的文件非常难维护.
MVVM模式可以减轻C的负担,`但是MVVM模式的总代码量也会相应增加`,在一些简单的页面上没必要生搬硬套地用上.
所以一个项目中的页面是有必要在MVC和MVVM两种模式中做切换的.

## 独立开的数据流中心 Service

我在项目中构建了一个`Service`模块, 作为数据流的中心, 这个模块的作用有以下几点:

1.存储Model数据

2.调用异步函数例如 Network API, Location API, 第三方SDK API, Database API等操作

3.`结合RxSwift框架提供一系列可监听的Observable供外部使用`

## RxSwift & Stream

ReactiveX中的Observable

## MVC & RxSwift




---
