---
layout:     post
title:      "RxSwift, 异步操作组合处理"
subtitle:   " \"工作期间项目中RxSwift的应用\""
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

## 响应式编程&链式编程

接手thinker.vc的几个iOS共享经济项目, 有较多**后台定时的网络请求,定位和蓝牙操作的组合**. 
原实现方案是直接把不同操作通过**闭包**嵌套起来, 如此一来有些比较头疼的问题:
 
- 异步操作组合出现"回调地狱", 每个组合操作的业务上有变动需要做大修.
 
- 任务管理不便,无法获取或取消上一次的请求/操作.

- 异步响应不及时可能造成之前的请求后至, 让数据出错. 或在页面退出之后仍然在进行未完成的请求.

网上较推荐的解决方案,就是使用**响应式编程框架**.
大体分为[RxSwift](https://github.com/ReactiveX/RxSwift)和[ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa)/
[ReactiveSwift](https://github.com/ReactiveCocoa/ReactiveSwift).

**ReactiveCocoa**是从OC时代就开始有的, 在Swift时代迁移过来, 开始的时候只是单纯的调用OC源码混编, 到后来衍生出了**ReactiveSwift**.
而**RxSwift**则是"原生"Swift诞生的.


刚好公司的项目里面本来已经使用了[Swagger](https://github.com/swagger-api/swagger-codegen)来自动生成网络请求业务的代码, 
自带[Moya](https://github.com/Moya/Moya)框架并选择的是**RxSwift**.
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
// 更新数据以及界面
func update(with devices: [Device])
// 处理错误
func handlError(_ error: Error)
```

原来的传统闭包实现方式

```swift
self.updateLocation(success: { [weak self] location in
         self?.fetchDevices(near: location, 
                            success: { res in 
                               self?.update(with: res)
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
                                       self?.update(with: res)
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
    .subscribe(onNext:  {[weak self] in self?.update(with: $0)},
               onError: {[weak self] in self?.handleError($0)})
    .disposed(by: self.disposeBag)
```

然后逐步解析这段代码, 关键点主要有:

- 链式

  把原来通过闭包回调的函数都改造成了返回**Observable**的函数, 改造后回调的处理就通过链式调用**Observable**的**Operator**结合函数式编程来传递.

- 函数式
  
  把处理回调的函数作为参数传递给**flatMap**和**subscribe**函数.

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

**RxSwift**的做法, 利用Observable的``zip``函数把多个下载任务聚合:

```swift
Observable.zip(self.rx_downloadImage(withURL: url1), self.rx_downloadImage(withURL: url2), self.rx_downloadImage(withURL: url3))
          .subscribe(onNext: {[unowned self] res in self.updateDatabase()}
                     onError: {[unowned self] err in self.handleError(error: error)})
          .disposed(by: self.disposeBag)
```

## 改造耗时的同步函数

ReactiveX框架确实很适合用来处理异步场景,只要使用者习惯了用return Observable来代替block, 并且要在Observable中传递处理结果.

RxSwift提供了一个比较简单的方式创建Observable, 就是Observable的静态函数`create`:

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

这样就封装了一个在内部子线程调用旧函数`downloadImage`的函数. 对于子线程调用这个需求, 简单地使用了`global`队列来异步执行下载.

## 队列的切换

以上的做法实现了在函数内异步下载并回调的需求, 但是这个做法会让调用者无法控制下载任务在哪个队列执行. 

如果我们有必要手动去控制这类函数执行的队列, 可以通过Rx提供的解决方案实现, 就是以下两个函数`observeOn`和`subscribeOn`:

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

## Observable的"缺陷"

在串行异步请求处理里面提到了这样的处理方式并不完善, 因为Observable有一个特点就是**抛出Error之后就会自动销毁**. 
而这段代码里面, 下拉刷新的Observable是与另外两个异步操作``聚合``在一起的, 就是说如果网络请求或者定位的操作抛出Error, 那么用户下一次下拉刷新也是**不会被处理**的.

所以, 为了避免这个问题, 解决方式有两种:

1.不要把下拉刷新的Observable与另外两个异步操作的Observable聚合;

2.``拦截``另外两个Observable可能抛出的Error

第1个方案的解决代码:

```swift
self.tableView.rx_pullToRefresh
    .subscribe(onNext: { [unowned self] in
        _ = self.rx_updateLocation()
                .flatMap({self.rx_fetchDevices(near: $0)})
                .subscribe(onNext:  {self.update(with: $0)},
                           onError: {self.handleError($0)})
    })    
    .disposed(by: self.disposeBag)
```

这样就可以保证``rx_pullToRefresh``会一直被observe, 但是这个处理方式会让代码可读性又下降了.

所幸RxSwift还给我们提供了另外的选择, 利用Observable的``catchError``函数或者``catchErrorJustReturn``函数, 我们就可以把以上2个异步操作的错误拦截.

```swift
/**
 Continues an observable sequence that is terminated by an error with the observable sequence produced by the handler.
 
 - seealso: [catch operator on reactivex.io](http://reactivex.io/documentation/operators/catch.html)
 
 - parameter handler: Error handler function, producing another observable sequence.
 - returns: An observable sequence containing the source sequence's elements, followed by the elements produced by the handler's resulting observable sequence in case an error occurred.
 */
public func catchError(_ handler: @escaping (Error) throws -> RxSwift.Observable<Self.E>) -> RxSwift.Observable<Self.E>

/**
 Continues an observable sequence that is terminated by an error with a single element.
 
 - seealso: [catch operator on reactivex.io](http://reactivex.io/documentation/operators/catch.html)
 
 - parameter element: Last element in an observable sequence in case error occurs.
 - returns: An observable sequence containing the source sequence's elements, followed by the `element` in case an error occurred.
 */
public func catchErrorJustReturn(_ element: Self.E) -> RxSwift.Observable<Self.E>
```

其中``catchErrorJustReturn``函数可以让我们产生一个默认值来继续事件链, 
而``catchError``则接收一个闭包参数, 可以通过我们产生的Error来制定一个值给后续事件链或者直接在闭包里面处理Error.
显然这个场景我们是需要处理Error的, 所以选用``catchError``. 而这样的话, ``subscribe``函数的``onError``就永远都不会执行了, 等于是把Error的处理提前到了``catchError``:

```swift
// 处理Error并且返回默认值
func rx_handleError(_ error: Error) -> Observable<[Device]> {
    // 处理Error, 生成默认参数
    ...
    return Observable.of(__defaultValue__)
}
// 请求列表的数据(聚合2个异步操作的Observable)
func rx_fetchTableViewData() -> Observable<[Device]> {
    return self.rx_updateLocation().flatMap({self.rx_fetchDevices(near: $0)})
}

self.tableView.rx_pullToRefresh
    .flatMap({ [unowned self] in 
        self.rx_fetchTableViewData()
            .catchError({self.rx_handleError($0)})
    })
    .subscribe(onNext: {[weak self] in self?.update(with: $0)})
    .disposed(by: self.disposeBag)
```

以上这样操作算是把Error处理了, 但是还是存在问题, 我们还需要再改进一下, 不要让handleResponseValue和handleError的代码分散.

## 配合enum以及泛型更舒服地处理Error

Swift里面的枚举类型是可以带参数的, 我们可以把我们要的结果抽象出来, 定义一个枚举:

```swift
enum Result {
    case value([Device])
    case error(Error)
}
```

然后改动一下我们的fetch以及handle方式:

```swift
// 改进后的fetch函数把[Device]和Error转换成Result
func rx_fetchTableViewData() -> Observable<Result> {
    return self.rx_updateLocation()
               .flatMap({self.rx_fetchDevices(near: $0)})
               .map({Result.value($0)})
               .catchError({Observable.of(Result.error($0))})
}
// 统一处理Result
func handleResult(_ result: Result) {
    switch result {
        case .value(let value):
            self.update(with: value)
        case .error(let error):
            self.handleError(error)
    }
}
```

最后我们这个``下拉刷新->更新定位坐标->获取附近设备->更新table``的代码就可以变成:

```swift
self.tableView.rx_pullToRefresh
    .flatMap({[unowned self] in self.rx_fetchTableViewData()})
    .subscribe(onNext: {[weak self] in self?.handleResult($0)})
    .disposed(by: self.disposeBag)
```

但是, 实际上有这种场景的不止是设备列表, 还有比如附近的设备中心(Station), 这个``Result``显然可以承担更多的任务.

利用Swift的泛型改进, 让``Result``适用于通用的场景:

```swift
enum Result<T> {
    case value(T)
    case error(Error)
}
func rx_fetchDeviceTableViewData() -> Observable<Result<[Device]>> {
    return ...
}
func handleDeviceResult(_ result: Result<[Device]>) {
    ...
}
func rx_fetchStationTableViewData() -> Observable<Result<[Station]>> {
    return ...
}
func handleStationResult(_ result: Result<[Station]>) {
    ...
}
```

## 改造原来的GCD异步函数

如果在在项目中途接入Rx, 原来项目中已经存在大量通过GCD回调的函数了.
 
这个时候把全部函数都改造是很高成本的, 而且部分函数可能在项目中被调用了很多次, 涉及的模块可能比较多, 但是不一定每个调用了这个函数的模块都有必要接入Rx. 
在这种情况下通用可以使用Observable的`create`函数去封装原来的函数.

比如有一个加载本地数据的函数:

```swift
function loadDataFromLocal(filePath: URL, success: (Data)->Void, failure: (Error)->Void) {
    ...
    ...
    ...
}
```

在不改动原函数的情况下, 增加一个新的函数:

```swift
function rx_loadDataFromLocal(filePath: URL) -> Observable<Result<Data>> {
    return Observable.create({ observer in
    
        loadDataFromLocal(filePath: filePath,
                          success: {data in observer.onNext(.value(data))}, 
                          failure: {error in observer.onNext(.error(error))})
                          
        return Disposables.create()
    })
}
```

这样的函数改造和同步函数的改造一样, 有一个类似的缺陷, 就是**不可以再改变`loadDataFromLocal`函数在哪个队列执行**.

相对的, 这个改造方式可以比较简便地复用现有的函数.

## Delegate回调

除了GCD, delegate也是异步回调的一种常用方式, 而delegate回调其实也是有可能和异步函数嵌套组合.

## 取消异步任务

这个比较容易操作, 使用Observable的``disposed(by:)``函数, 在想要取消这个任务的时候把传入的``DisposeBag``实例销毁.

比方说让ViewController持有一个DisposeBag, 在ViewController退出调用deinit的时候, 绑定在这个DisposeBag上的Observable就都不会继续处理了.

## 小结

以上是Thinker.vc项目中接入RxSwift对异步/并行操作的优化改进经历.概括来说, 就是:

1.把多层的闭包嵌套降维成链式单层闭包

2.把散落的Error处理整合起来

3.更灵活方便地切换队列

4.更明确展现异步操作之间的关系

5.更方便顺应业务改变异步操作组合

6.可以取消任务响应

---
