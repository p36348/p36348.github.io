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

## 需要持久化的数据

- 启动广告数据

- 首页广告数据

- 登录过的用户的数据

- 用户的操作记录(搜索记录, 聊天记录, 购物车数据, 编辑页面缓存等)

- 列表数据缓存

## 持久化策略

首先, 以上数据有一部分是和登录用户无关的, 而不同用户对应的数据应该分开存储. 所以在项目中的库会有N+1个(N是登录用户的个数).

跟登录用户无关的数据包括:

- 启动广告

- 首页广告

- 用户获取验证码的记录(用于倒计时)

- 部分列表数据(视项目而定)

然后有少部分数据通过UserDefault保存, 包括:

- 当前登录的用户标识(ID)

- 是否需要显示广告

- 需要显示的广告标识(ID)

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

其中`defaultDatabase`是固定的, 而`userDatabase`是根据登录用户切换的. 
根据Realm的文档, 我们可以通过Realm.Configuration来配置数据库所在路径. 
根据场景, `defaultDatabase`使用默认路径即可.

而`userDatabase`则根据用户的id来配置. 
这个需要通过`绑定`用户的登出与登入行为来切换.在Thinker.vc的项目中, 通过`RxSwift`来绑定`UserService`中对应的`Observable`. 先不讨论.

```swift
class DatabaseService {

    /// 单例
    static let shared: DatabaseService = DatabaseService()
    
    /// 基本数据库
    public let defaultDatabase: Realm = {
        var config = Realm.Configuration.defaultConfiguration
        // buildVersion是app对应的build版本, 只增不减. 目的是为了数据库migration*
        config.schemaVersion = UInt64(buildVersion as! String)!
        return try! Realm(configuration: config)
    }()
    
    /// 用户相关数据库
    public var userDatabase: Realm? = nil
    
    private init() {
        ...
        ...
    }
    
    /// 根据用户切换数据库
    func updateUserDatabase(for user: UserModel?) {
        guard let uid = user?.uid else {return}
        
        // https://realm.io/docs/swift/latest/#configuring-a-realm  根据uid命名数据库
        let fileUrl = Realm.Configuration.defaultConfiguration.fileURL!.deletingLastPathComponent().appendingPathComponent("user_\(uid).realm")
        let config = Realm.Configuration(fileURL: fileUrl, schemaVersion: UInt64(buildVersion as! String)!)
        
        self.userDatabase = try! Realm(configuration: config)
    }
}
```

`CogfigService`模块管理配置数据, 其中就包括了UserDefault要保存的数据: 
([泛型配合UserDefault便捷加载数据](https://p36348.github.io//2018/12/13/genericity-userdefault/))

```swift
// UserDefault关键字管理
private struct UserDefaultKeys {
    
    /// 首页广告id
    static let indexAdvertisementID: String = "UserDefaultKeys.indexAdvertisementID"
    
    /// 首页是否显示过
    static let indexAdvertisementShown: String = "UserDefaultKeys.indexAdvertisementShown"
    
    /// 是否显示过引导页
    static let guideShown: String = "UserDefaultKeys.guideShown"
    
    /// 登录用户id
    static let userID: String = "UserDefaultKeys.userID"
}
```

## Realm的Data Model

需要继承Realm的基本类`Object`, 使用非常方便.


---

---