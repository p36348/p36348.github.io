---
layout:     post
title:      "RxSwift应用MVVM"
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

# 前言

目前高移动端应用的要求已经越来越高, 主要体现在:

1. 越来越复杂的用户可操作页面;
2. 多个页面一起承载繁琐的业务;
3. 多个状态需要实时反映到应用界面.

这种背景下正是MVVM模式施展的时候, 而MVVM模式下的好搭档无疑要提到响应式框架.以下把个人在工作中对RxSwift&MVVM实践的经验记录一下.

## 数据流中心模块: Service

我在项目中构建了一个`Service`模块, 作为数据流的中心, 这个模块的作用有以下几点:

- 单例模式, 项目共享数据与状态.

- 提供业务/功能相关的函数, 内部调用下一层异步函数, 例如 Network API, Location API, 第三方SDK API, Database API等操作.

- 以属性的形式提供全局可观察事件, 例如 User登录状态变更, Location 更新.

### Service模块之Observable

在Service模块中, 大部分函数的返回值和属性的类型都是Observable. 

Observable是Rx框架中的最基本可观察类型, 其中有一个需要用到的概念——广播型.

广播类型和非广播类型的区别在于, **广播类型可以同时多次subsribe**, 这个特性正好契合全局可观察事件. 因此Service模块中提供的Observable属性都应该是广播类型的. 而Rx框架里面的广播类型Observable, 对应的就是**`PublishSubject`**和**`ReplaySubject`**.

而相对地, 函数返回的对象则只需要用非广播类型, 因为实际情况调用函数的回调都只是调用者自己需要subscribe. **`如果这个回调的结果有必要转化成广播类型, 则应该通过调用者自己转换`**.

### Service模块之属性

既然已经可以确定, Service属性中的Observable需要用广播类型, 那如何选择**`PublishSubject`**和**`ReplaySubject`**?

先看两个类型的描述:

```swift
/// Represents an object that is both an observable sequence as well as an observer.
///
/// Each notification is broadcasted to all subscribed observers.
public final class PublishSubject<Element>
    : Observable<Element>
    , SubjectType
    , Cancelable
    , ObserverType
	, SynchronizedUnsubscribeType {
    ...
    ...
}

/// Represents an object that is both an observable sequence as well as an observer.
///
/// Each notification is broadcasted to all subscribed and future observers, subject to buffer trimming policies.
public class ReplaySubject<Element>
    : Observable<Element>
    , SubjectType
    , ObserverType
	, Disposable {
    ...
    ...
}
```

从注释上面可以了解到, `ReplaySubject`的不同在于, 它的数据会发送给将来订阅它的监听者, 也就是说, **`ReplaySubject`的数据可以追溯**.

可以得出**`ReplaySubject`**的使用策略: 

**当数据发送的时机早于数据被正式使用的时候, 我们用`ReplaySubject`, 这样可以避免我们丢失已经获得的数据.**

### Service模块之函数

在设想中, 我希望`Service`是MVVM中的`ViewModel`的辅助, `ViewModel`调用`Service`提供的函数, 而`Service`应该帮`ViewModel`**处理好与UI无关的业务逻辑**. 所以`Service`的函数应该有大量异步串行/并行的操作, 并且返回`Observable`类型.

剩下需要注意的就是, **函数的副作用**. 这一点非常值得注意, 因为稍不注意副作用的管理, 就会导致`ViewModel`收到一些莫名的数据更新. 对此, 人为约束:

- 有副作用的函数, 会导致`Service`持有的数据改变, 或者导致`Service`的可观察对象发送数据的, 使用以下命名:
  - 会返回数组数据的函数, `reloadxxx`表示重新刷新, `loadMorexxx`表示加载更多.
  - 返回`hash`数据/`Model`的函数, 使用`updatexxx`表示更新.
- 没有副作用的函数:
  - 请求数据的, 无论返回什么类型, 使用`fetchxxx`表示请求.
  - 执行某类操作, 使用`performxxx`. 例如登录, 命名应该是`performSignin`.

## MVVM的核心模块: ViewModel和View

MVVM有别于MVC主要在于`ViewModel`模块, 设想中它的功能有以下:

- 存储`View/ViewController`层要**直接显示**的数据, 注意不只是持有`Model`.
- 针对对应的`View/ViewController`提供UI交互的输入口(Tableview, TextField, Button, Segmented等操作).
- 内部处理UI的input数据, 调用对应`Service`的业务函数, 并提供输出口绑定到`View/ViewController`上.
- 映射`Service`模块的一些可观察事件, 输出给`View/ViewController`subscribe.

```swift
class SignViewControllerViewModel {
    // input
    var usernameInput: BehaviorRelay<String> = BehaviorRelay(value: "")
    var passwordInput: BehaviorRelay<String> = BehaviorRelay(value: "")
    // output
    var usernameInputEnable: Observable<Bool>
    var passwordInputEnable: Observable<Bool>
    var submitEnable: BehaviorRelay<Bool> = BehaviorRelay(value: true)
    var submitTitle: BehaviorRelay<String> = BehaviorRelay(value: "Submit")
    var indicatorHidden: Observable<Bool>
    var stateHidden: Observable<Bool>
    var state: Observable<String> {
        return _internalState.asObservable()
    }
    ...
    ...
    
    init(disposeBag: DisposeBag!) {
        ...
    }
    
    func handleClickSubmit() {
        ...
    }
}

```



MVVM中的**V**是视图层, 在项目中视图层主要其实是`ViewController`和各种`View`的子类, 并不是单独指`View`.

而`View/ViewController`要做的就是做布局, 以及调用`ViewModel`:

```swift
class SignViewController: UIViewController {
    // 懒加载, 传入disposeBag
   	var viewModel: SignViewControllerViewModel
    // UI
    @IBOutlet weak var usernameTf: UITextField!
    @IBOutlet weak var passwordTf: UITextField!
    @IBOutlet weak var submitButton: UIButton!
    @IBOutlet weak var stateLabel: UILabel!
    @IBOutlet weak var indicatorView: UIActivityIndicatorView!
    
    // deinit之后释放subscribtions
    let disposeBag: DisposeBag = DisposeBag()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        ...
        ...
        self.bindObservables()
    }
    
    func bindObservables() {
        // 数据输入 viewModel
        self.usernameTf.rx.text.orEmpty
            .bind(to: self.viewModel.usernameInput)
            .disposed(by: self.disposeBag)
        
        self.passwordTf.rx.text.orEmpty
            .bind(to: self.viewModel.passwordInput)
            .disposed(by: self.disposeBag)
        
        self.submitButton.rx.tap
            .subscribe(onNext: {_ in self.viewModel.handleClickSubmit()})
            .disposed(by: self.disposeBag)
        
        
        // viewModel 数据输出
        self.viewModel.submitEnable
            .bind(to: self.submitButton.rx.isEnabled)
            .disposed(by: self.disposeBag)
        
        self.viewModel.usernameInputEnable
            .bind(to: self.usernameTf.rx.isEnabled)
            .disposed(by: self.disposeBag)
        
        self.viewModel.passwordInputEnable
            .bind(to: self.passwordTf.rx.isEnabled)
            .disposed(by: self.disposeBag)
        
        self.viewModel.submitTitle
            .bind(to: self.submitButton.rx.title())
            .disposed(by: self.disposeBag)
        
        self.viewModel.state
            .bind(to: self.stateLabel.rx.text)
            .disposed(by: self.disposeBag)
        
        self.viewModel.indicatorHidden
            .bind(to: self.indicatorView.rx.isHidden)
            .disposed(by: self.disposeBag)
        
        self.viewModel.indicatorHidden
            .map({!$0})
            .bind(to: self.indicatorView.rx.isAnimating)
            .disposed(by: self.disposeBag)
        
        self.viewModel.stateHidden
            .bind(to: self.stateLabel.rx.isHidden)
            .disposed(by: self.disposeBag)
    }
}
```

在理想情况下, `View/ViewController`只面对`ViewModel`, 而`ViewModel`面对的就是`Service`, `Model`, `View/ViewController`

注意: 因为`ViewController`中必定存在若干个`View`, 而`View`是可以复用的, 甚至`ViewController`也是可以作为`childViewController`被复用, 所以ViewController的应该有一个对应的ViewModel, 而ViewController的ViewModel**有可能需要持有若干个**View的ViewModel(有点绕, 出图表达).

![](/img/in-post/post-rx-swift-mvvm/view_model_structure.png)

## 数据: Model

`Model`在MVVM中是最轻的一个模块, 除了保存数据的值, 它什么都不需要做, 所见即所有.

# 最终的MVVM结构

![](/img/in-post/post-rx-swift-mvvm/mvvm_structure.png)

# [示例代码](https://github.com/p36348/p36348.github.io/tree/master/_sources/rx_swift_in_mvvm/RxSwiftMVVMDemo)

---

`持续更新...`