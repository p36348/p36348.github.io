---
layout:     post
title:      "Swift泛型应用, UserDefault"
date:       2018-12-13 12:00:00
author:     "P36348"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - iOS
    - Swift
    - Generic
    - UserDefault
    - 数据持久化
---

开发中经常会用到UserDefault来存储零碎的数据, 用普通的写法比较低效.

```swift
let uid = UserDefaults.standard.value(forKey: "userdefault.key.uid") as? Int
```

可以利用Swift的`extension`来给Int类型添加便捷的函数:

```swift
extension Int {
    static func valueFromUserDefault(forKey key: String) -> Int? {
        return UserDefaults.standard.value(forKey: key) as? Int
    }
}

let uid = Int.valueFromUserDefault(forKey: "userdefault.key.uid")
```

而实际上, UserDefault存储的数据类型还有好几种, 而实现代码其实都是大同小异, 要每一个都这样添加函数就太麻烦了. 
有一种更高效的方式给各种类型添加这个函数, 而且也方便统一修改. 就是用泛型和Protocol:

```swift
public protocol UserDefaultable {
    associatedtype E
    
    static func objectUserDefaults(forKey key: String) -> E?
}

extension UserDefaultable {
    public static func valueFromUserDefaults(forKey key: String) -> E? {
        return UserDefaults.standard.value(forKey: key) as? E
    }
}
```

这样只要声明了遵守UserDefaultable的类型都可以使用`valueFromUserDefaults`静态函数, 很自然, 这个`associatedtype E`就是绑定这个类型本身.

```swift
extension Bool: UserDefaultable {
    public typealias E = Bool
}

extension Int: UserDefaultable {
    public typealias E = Int
}

extension Int32: UserDefaultable {
    public typealias E = Int32
}

extension Int64: UserDefaultable {
    public typealias E = Int64
}

extension String: UserDefaultable {
    public typealias E = String
}

extension Double: UserDefaultable {
    public typealias E = Double
}

extension Data: UserDefaultable {
    public typealias E = Data
}

extension Array: UserDefaultable {
    public typealias E = Array
}
```