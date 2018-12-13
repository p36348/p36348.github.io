---
layout:     post
title:      "iOS数据持久化解决方案Realm"
date:       2018-12-04 12:00:00
author:     "P36348"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - iOS
    - Swift
    - 数据持久化
    - 数据库
    - Realm
---

## 前言

iOS数据持久化数据库方式解决方案, 主流的有FMDB(SQLite), CoreData, Realm.

三者中CoreData的使用比较不方便, FMDM需要关注线程安全的问题. 而Realm除了API友好, 在线程处理上也相对方便, 而且可以使用桌面应用[Realm Browser](https://itunes.apple.com/us/app/realm-browser/id1007457278?mt=12)查看iOS模拟器的数据库文件.

## 需要持久化的有哪些数据

- 启动广告数据

- 首页广告数据

- 登录过的用户的数据

- 用户的操作记录(搜索记录, 聊天记录, 购物车数据, 编辑页面缓存等)

- 列表数据缓存

## 保存策略

首先, 以上数据有一部分是和登录用户无关的, 而不同用户对应的数据应该分开存储. 所以在项目中的库会有N+1个(N是登录用户的个数).

跟用户无关的数据包括:

- 启动广告

- 首页广告

- 部分列表数据(视项目而定)

然后有少部分数据通过UserDefault保存, 包括:

- 当前登录的用户id(或者其他标识)

- 是否需要显示广告

在Thinker.vc项目中会在一个`DatabaseService`模块里面管理数据库:

```swift
class DatabaseService {

    /// 单例
    static let shared: DatabaseService
    
    /// 基本数据库
    public let defaultDatabase: Realm
    
    /// 用户相关数据库
    public var userDatabase: Realm?
}
```

`CogfigService`模块管理配置数据, 其中就包括了UserDefault要保存的数据: ([泛型配合UserDefault便捷加载数据](https://p36348.github.io//2018/12/13/genericity-userdefault/))

```swift
/// 关键字管理
private struct UserDefaultKeys {
    static let indexAdvertisementID: String = "UserDefaultKeys.indexAdvertisementID"
    static let indexAdvertisementShown: String = "UserDefaultKeys.indexAdvertisementShown"
    static let guideShown: String = "UserDefaultKeys.guideShown"
}
```

---

---