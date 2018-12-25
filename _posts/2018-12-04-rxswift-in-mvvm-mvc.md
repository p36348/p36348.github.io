---
layout:     post
title:      "RxSwift, MVVM, MVC"
subtitle:   " \"主要记录工作期间项目中RxSwift的应用\""
date:       2018-12-01 12:00:00
author:     "P36348"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - iOS
    - RxSwift
    - Swift
    - 响应式编程
    - MVVM
    - MVC
---

以目前主流项目的功能需求,MVC模式下的C都很难避免臃肿的情况.
在稍复杂的页面上,如果不想办法把C瘦身,会让C的文件非常难维护.
MVVM模式可以在这个时候大展拳脚,但是MVVM模式下代码量相对会比较大,没必要在一些简单的页面上生搬硬套上.
所以一个项目中的页面是有必要在MVC和MVVM两种模式中做切换的.



---