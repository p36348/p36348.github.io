---
layout:     post
title:      "RxCocoa如何把UIKit的Delegate桥接"
subtitle:   "记一次RxSwift/RxCocoa源码阅读"
date:       2019-02-19 12:00:00
author:     "P36348"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - iOS
    - RxSwift
    - RxCocoa
    - UIKit
    - Swift
    - 响应式编程
---

## RxCocoa的调用(`scrollView.rx.didScroll`)

得益于**RxCocoa**对UIKit做了extension，我们使用UI组件的Rx封装时只需要调用`rx`属性，就可以访问到Rx框架的内容。比如需要订阅***UIScrollView***的滚动事件：

```swift
scrollView
	.rx.didScroll
	.subscribe(onNext: { () in 
		print("scroll view did scroll")
	})
```

***`But, why?？？`***

在原生的Cocoa框架中，要监听UIScrollView的滑动事件，需要通过实现***UIScrollViewDelegate***协议的`scrollViewDidScroll`函数。但是RxCocoa为何可以让用户通过简单的调用就得到了回调？***UIScrollViewDelegate***是如何被隐藏的？

## `rx`属性是怎么来的

通过查看，`rx`属性是一个`Reactive`的类型。在*Reactive.swift*文件，可以看到它是一个泛型**struct** 。而`rx`是在一个`ReactiveCompatible`的**protocol**里面定义的。

```swift
public struct Reactive<Base> {
    /// Base object to extend.
    public let base: Base

    /// Creates extensions with base object.
    ///
    /// - parameter base: Base object.
    public init(_ base: Base) {
        self.base = base
    }
}

/// A type that has reactive extensions.
public protocol ReactiveCompatible {
    /// Extended type
    associatedtype CompatibleType

    /// Reactive extensions.
    static var rx: Reactive<CompatibleType>.Type { get set }

    /// Reactive extensions.
    var rx: Reactive<CompatibleType> { get set }
}

extension ReactiveCompatible {
    /// Reactive extensions.
    public static var rx: Reactive<Self>.Type {
        get {
            return Reactive<Self>.self
        }
        set {
            // this enables using Reactive to "mutate" base type
        }
    }

    /// Reactive extensions.
    public var rx: Reactive<Self> {
        get {
            return Reactive(self)
        }
        set {
            // this enables using Reactive to "mutate" base object
        }
    }
}


import class Foundation.NSObject

/// Extend NSObject with `rx` proxy.
extension NSObject: ReactiveCompatible { }
```

可以看到`Reactive`的定义非常简单，只有一个`base`属性，而这个属性是一个**泛型**。

在文件的最后一行，***NSObject***被声明了遵守`ReactiveCompatible`，因此***UIScrollView***也遵守了该协议。

查看`ReactiveCompatible`的定义以及其下方的extension就可以看出，`rx`属性的泛型是该类型本身，也就是说***UISCrollView***的`rx`属性就是一个***UISCrollView***泛型的`Reactive`结构体，**其`base`属性就是这个*UIScrollView*本身的实例**。

##  `didScroll`是怎么来的

顺着往下看，可以在*UIScrollView+Rx.swift*文件里找到，是通过**extension**的**where**语法对*UIScrollView*泛型的`Reactive`添加的一个*计算型属性*。

```swift
extension Reactive where Base: UIScrollView {
	...
    	...
    	...
        /// Reactive wrapper for delegate method `scrollViewDidScroll`
        public var didScroll: ControlEvent<Void> {
            let source = RxScrollViewDelegateProxy.proxy(for: base).contentOffsetPublishSubject
            return ControlEvent(events: source)
        }
    	...
    	...
    	...
    }
```

从命名可以看出，`RxScrollViewDelegateProxy`是一个实现`UIScrollViewDelegate`的代理类型。

暂时不探究`RxScrollViewDelegateProxy`的`proxy(for:)`函数，比较好理解，`source`是通过一个对应***UISrollView***实例获得的`RxScrollViewDelegateProxy`的一个**UIScrollView的contentOffset事件广播**，函数最后再把这个广播封装成一个`ControlEvent`类型实例，也就是被订阅的那个事件。

而关键就在于`RxScrollViewDelegateProxy`这个类型做了什么事情。

## RxScrollViewDelegateProxy

继续查看*RxScrollViewDelegateProxy.swift*文件

```swift
/// For more information take a look at `DelegateProxyType`.
open class RxScrollViewDelegateProxy
    : DelegateProxy<UIScrollView, UIScrollViewDelegate>
    , DelegateProxyType 
, UIScrollViewDelegate {
    ...
    ...
    ...
    fileprivate var _contentOffsetPublishSubject: PublishSubject<()>?
    ...
    /// Optimized version used for observing content offset changes.
    internal var contentOffsetPublishSubject: PublishSubject<()> {
        if let subject = _contentOffsetPublishSubject {
            return subject
        }

        let subject = PublishSubject<()>()
        _contentOffsetPublishSubject = subject

        return subject
    }
    ...
    ...
    ...
}
```

`RxScrollViewDelegateProxy`继承了`DelegateProxy`，并实现了两个协议：

- DelegateProxyType
- ***UIScrollViewDelegate***

显然这里就是***UIScrollViewDelegate***的藏身点，下面继续看在这里它是如何被黑盒化的。

在这个文件里可以找到原始回调的函数：

```swift
/// For more information take a look at `DelegateProxyType`.
    public func scrollViewDidScroll(_ scrollView: UIScrollView) {
        if let subject = _contentOffsetBehaviorSubject {
            subject.on(.next(scrollView.contentOffset))
        }
        if let subject = _contentOffsetPublishSubject {
            subject.on(.next(()))
        }
        self._forwardToDelegate?.scrollViewDidScroll?(scrollView)
    }
```

在回调的处理上，做了3件事：

- 发送contentOffset的数值变化广播
- 发送contentOffset变化的事件广播
- 调用了`_forwardToDelegate`属性的另一个***scrollViewDidScroll***函数

可以看到两个广播事件应该是按需发送的，如果在没有订阅者的情况下，应该不会产生对应的属性。而在上面被订阅的的`contentOffsetPublishSubject`就是这个`_contentOffsetPublishSubject`。

而`_forwardToDelegate`应该是开发者在外部设置的delegate，也就是说：

**即使开发者已经为一个*UIScrollView*设置了*delegate*（或者没有设置），也不会影响通过RxCocoa框架去订阅这个UIScrollView的事件。 而且可以有多个订阅者通过RxCocoa去订阅UIScrollView的回调事件，因为这里的`Observable`是广播类型。**

可见在***RxCocoa***中，每一个有事件订阅者的***UIScrollView***都有与之对应的`RxScrollViewDelegateProxy`实例。所以下一个问题就是：

代码`RxScrollViewDelegateProxy.proxy(for: base)`当中，`proxy(for:)`函数是如何把一个`RxScrollViewDelegateProxy`绑定到一个***UIScrollView***上的？

`To be continued...`