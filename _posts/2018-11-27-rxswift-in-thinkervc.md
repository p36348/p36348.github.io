---
layout:     post
title:      "RxSwift的实战记录"
subtitle:   " \"主要记录一下Thinker.vc工作期间项目中RxSwift的应用\""
date:       2018-11-27 12:00:00
author:     "P36348"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - iOS
    - RxSwift
    - Swift
    - 响应式编程
---

> “Yeah It's on. ”

## 响应式编程&链式编程

接手thinker.vc的几个iOS共享经济项目, 有较多**后台定时的网络请求,定位和蓝牙操作的组合**. 
原实现方案是直接把不同操作通过**闭包**嵌套起来, 如此一来有些比较头疼的问题:
 
- 异步操作组合出现"回调地狱", 每个组合操作的业务上有变动需要做大修.
 
- 任务管理不便,无法获取或取消上一次的请求/操作.

- 异步响应不及时可能造成之前的请求后至, 让数据出错.

网上较推荐的解决方案,就是使用**响应式编程框架**.
大体分为[RxSwift](https://github.com/ReactiveX/RxSwift)和[ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa)/
[ReactiveSwift](https://github.com/ReactiveCocoa/ReactiveSwift).

**ReactiveCocoa**是从OC时代就开始有的, 在Swift时代迁移过来, 开始的时候只是单纯的调用OC源码混编, 到后来衍生出了**ReactiveSwift**.
而**RxSwift**则是"原生"Swift诞生的.


刚好公司的项目里面本来已经使用了[Swagger](https://github.com/swagger-api/swagger-codegen)来自动生成网络请求业务的代码, 
自带[Moya](https://github.com/Moya/Moya)框架.
于是就理所当然用了**RxSwift**.


<p id = "build"></p>
---

## 串行异步请求处理

举个例子需要更新坐标然后请求附近的设备列表, 需要考虑定位超时或者定位出错的情况(例如没有开启权限), 以及提示用户当前正在更新数据.

分解之后需要实现以下函数:

```swift
// 更新坐标
func updateLocation(success: (CLLocation) -> Void, failure: (Error) -> Void)
// 请求设备
func fetchDevices(near location: CLLocation, success: ([Device]) -> Void, failure: (Error) -> Void)
// 根据数据更新界面
func updateUI(with devices: [Device])
// 处理错误
func handlError(_ error: Error)
```

原来的传统闭包实现方式

```swift
self.updateLocation(success: { [weak self] location in
         self?.fetchDevices(near: location, 
                            success: { res in 
                               self?.updateUI(with: res)
                            }
                            failure: { err in 
                               self?.handlError(err)
                            })}, 
                     failure: { [weak self] err in 
                         self?.handlError(err)
                     })
```

类似这样的业务在共享系列项目经常出现, 如:

1.首页地图上轮询用户坐标附近的设备

2.设备列表界面

先讨论第2种情况, 这个时候上面的代码应该在下拉刷新中调用, 就是实际上项目写出的是这种嵌套的代码:

```swift
self.tableView.pullToRefresh(action: { [unowned self] in
        self.updateLocation(success: { [weak self] location in
                 self?.fetchDevices(near: location, 
                                    success: { res in 
                                       self?.updateUI(with: res)
                                    }
                                    failure: { err in 
                                       self?.handlError(err)
                                    })}, 
                             failure: { [weak self] err in 
                                 self?.handlError(err)
                             })
})
```

一开始我尝试把更新定位和数据这一段操作封装为一个函数**func updateData()**, 然后在tableView的下拉刷新闭包里面调用. 

如此类推也可以把**func updateData()**里面的操作继续拆分, 视觉上就不会面对这么难看的代码块. 

**可是即使这样, 下拉刷新之后一系列的请求动作还是没有很直观的呈现出来, 代码也会因此变得相对零散, 最后甚至更加难看出原来的业务逻辑.**

这个时候搬出**RxSwift**. 在接入之后, 代码就可以改成以下这个样子:

```swift
self.tableView.rx_pullToRefresh
    .flatMap({[unowned self] in self.rx_updateLocation()})
    .flatMap({[unowned self] in self.rx_fetchDevices(near: $0)})
    .subscribe(onNext:  {[weak self] in self?.updateUI(with: $0)},
               onError: {[weak self] in self?.handleError($0)})
    .disposed(by: self.bag)
```

然后逐步解析这段代码, 关键点主要有:

- 链式

  把原来通过闭包回调的函数都改造成了返回**Observable**的函数, 改造后回调的处理就通过链式调用**Observable**的**Operator**结合函数式编程来传递.

- 函数式
  
  把处理回调的函数作为参数传递给**flatMap**和**subscribe**函数.
  
  **但是跟函数式编程的定义有点不一样, 以上几个函数是有副作用的.**

- Error聚合处理

  原来每个异步操作都有一个Error的返回, 用闭包回调的方式需要在每次调用函数的时候传入. 
  而使用RxSwift的话, 其实所有Observable是被聚合成了1个, 就是在**subscribe**调用的那个, 而之前聚合起来的Observable的Error都可以在subscribe的**onError**参数里传递处理.
  
 这段代码仍不够完善, 优化方式后面会提到. 但是显而易见, 这一系列异步操作的条理顺序已经出来了, 而如果中间某一环需要修改, 也并不难操作.

---

## 并行异步请求处理

照样举例, 下载多张图片然后更新数据库:

分解之后有以下几个函数:

```swift
// 下载图片
func downloadImage(withURL: URL) -> (imageData: Data?, error: Error?)
// 处理下载好的图片
func handleDownloadImage(imageData: Data)

func handleError(error: Error)
// 写入数据库
func updateDatabase()
```

原来的实现:
```swift
let group = DispatchGroup()

DispatchQueue.global().async(group: group, execute: { [unowned self] in
    let res = self.downloadImage(withURL: url1)
    if let data = res.imageData {
        self.handleDownloadImage(imageData: data)
    }else if let error = res.error {
        self.handleError(error: error)
    }
})
DispatchQueue.global().async(group: group, execute: { [unowned self] in
    let res = self.downloadImage(withURL: url2)
    if let data = res.imageData {
        self.handleDownloadImage(imageData: data)
    }else if let error = res.error {
             self.handleError(error: error)
    }
})
DispatchQueue.global().async(group: group, execute: { [unowned self] in
    let res = self.downloadImage(withURL: url2)
    if let data = res.imageData {
        self.handleDownloadImage(imageData: data)
    }else if let error = res.error {
        self.handleError(error: error)
    }
})

group.notify(queue: DispatchQueue.main) { [unowned self] in
    self.updateDatabase()
}
```

**RxSwift**的做法:

```swift
Observable.zip(self.rx_downloadImage(withURL: url1), self.rx_downloadImage(withURL: url2), self.rx_downloadImage(withURL: url3))
          .subscribe(onNext: {[unowned self] res in self.updateDatabase()}
                     onError: {[unowned self] err in self.handleError(error: error)})
```

## 旧函数改造

ReactiveX框架确实很适合用来处理异步场景,只要使用者习惯了用return Observable来代替block, 并且要在Observable中传递处理结果.

RxSwift提供了一个比较简单的方式创建Observable, 就是Observable的静态函数**create**:

```swift
/**
Creates an observable sequence from a specified subscribe method implementation.
    
- seealso: [create operator on reactivex.io](http://reactivex.io/documentation/operators/create.html)
    
- parameter subscribe: Implementation of the resulting observable sequence's `subscribe` method.
- returns: The observable sequence with the specified implementation for the `subscribe` method.
*/
public static func create(_ subscribe: @escaping (RxSwift.AnyObserver<Self.E>) -> Disposable) -> RxSwift.Observable<Self.E>
```

这个函数只有一个闭包参数需要传入, 而这个闭包会给我们提供一个**Observer**的实例, 我们的操作都会在这个闭包里面执行, 而结果就通过**Observer**来派发, 比方上面的下载图片函数:

```swift
func rx_downloadImage(withURL url: URL) -> Observable<Data> {
    return Observable.create({ [unowned self] observer -> Disposable in 
        DispatchQueue.global().async({
        let res = self.downloadImage(withURL: url)
                if let data = res.data {
                    observer.onNext(data)
                    observer.onCompleted()
                }else if let error = res.error {
                    observer.onError(error)
                }
        })
        return Disposables.create()
    })
}
```

这样就封装了一个在内部子线程调用旧函数**downloadImage**的函数了. 对于子线程调用这个需求, 这里比较简单粗暴的用了GCD.

## 队列的切换

其实对于还可以通过RxSwift提供的办法写得更好看.
那就是通过:

```swift
/**
 Wraps the source sequence in order to run its observer callbacks on the specified scheduler.

 This only invokes observer callbacks on a `scheduler`. In case the subscription and/or unsubscription
 actions have side-effects that require to be run on a scheduler, use `subscribeOn`.

- seealso: [observeOn operator on reactivex.io](http://reactivex.io/documentation/operators/observeon.html)

- parameter scheduler: Scheduler to notify observers on.
- returns: The source sequence whose observations happen on the specified scheduler.
*/
public func observeOn(_ scheduler: ImmediateSchedulerType) -> PrimitiveSequence<Trait, Element> 
    
/**
 Wraps the source sequence in order to run its subscription and unsubscription logic on the specified 
 scheduler. 
    
 This operation is not commonly used.
    
 This only performs the side-effects of subscription and unsubscription on the specified scheduler. 
    
In order to invoke observer callbacks on a `scheduler`, use `observeOn`.

- seealso: [subscribeOn operator on reactivex.io](http://reactivex.io/documentation/operators/subscribeon.html)
    
- parameter scheduler: Scheduler to perform subscription and unsubscription actions on.
- returns: The source sequence whose subscriptions and unsubscriptions happen on the specified scheduler.
*/
public func subscribeOn(_ scheduler: ImmediateSchedulerType)
```

其中**subscribeOn**是指定了Observable是在什么队列被订阅的, 这个队列同时也会是被订阅的Observable任务执行所在的队列.
 
而**observeOn**则是指定了处理结果所在的队列, 就是我们最后调用**subscribe(onNext, onError,...)**的时候, 闭包里的任务的执行所在队列.

这两个函数接收的参数``ImmediateSchedulerType``是RxSwift定义的协议, 实际上已经帮我们实现了几个结构体可以直接使用, 比较常用的是``SerialDispatchQueueScheduler``, ``ConcurrentDispatchQueueScheduler``, ``ConcurrentMainScheduler``和``MainScheduler``.

我们可以在``SerialDispatchQueueScheduler``和``ConcurrentDispatchQueueScheduler``的init函数中传入对应的队列.
或者直接指定qos, 类似DispatchQueue的global函数.

通过这2个函数, 可以便捷地把原来同步执行的函数放到子队列里面执行, 然后回到主线程处理结果, 比方:

```swift
func rx_downloadImage(withURL url: URL) -> Observable<Data> {
    return Observable.create({ [unowned self] observer -> Disposable in 
        let res = self.downloadImage(withURL: url)
            if let data = res.data {
                observer.onNext(data)
                observer.onCompleted()
            }else if let error = res.error {
                observer.onError(error)
            }
        return Disposables.create()
    })
}

Observable.zip(self.rx_downloadImage(withURL: url1), self.rx_downloadImage(withURL: url2), self.rx_downloadImage(withURL: url3))
          .subscribeOn(ConcurrentDispatchQueueScheduler(qos: DispatchQoS.default))
          .observeOn(ConcurrentMainScheduler.asyncInstance)
          .subscribe(onNext: {[unowned self] res in self.updateDatabase()}
                     onError: {[unowned self] err in self.handleError(error: error)})

```

---
