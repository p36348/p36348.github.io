---
layout:     post
title:      "RxSwift应用MVVM(待完成)"
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
---

## 前言

MVC模式对于

## 数据流中心模块: Service

我在项目中构建了一个`Service`模块, 作为数据流的中心, 这个模块的作用有以下几点:

1.单例模式, 项目共享数据与状态.

2.调用异步函数(不产生副作用), 例如 Network API, Location API, 第三方SDK API, Database API等操作.

3.以属性的形式提供全局可观察事件, 例如 User登录状态变更, Location 更新.

## Service模块中套用Observable

在Service模块中, 大部分函数的返回值和属性的类型都是Observable. 

Observable是Rx框架中的最基本可观察类型, 其中有一个需要用到的概念——广播型.

广播类型和非广播类型的区别在于, **广播类型可以同时多次subsribe**, 这个特性正好契合全局可观察事件. 因此Service模块中提供的Observable属性都应该是广播类型的. 而Rx框架里面的广播类型Observable, 对应的就是**`PublishSubject`**和**`ReplaySubject`**.

*那什么时候用非广播类型?*



### 副作用



### 最终的MVVM结构

![](/img/in-post/rx-swift-mvvm/mvvm_structure.png)

---

`To be continued...`
