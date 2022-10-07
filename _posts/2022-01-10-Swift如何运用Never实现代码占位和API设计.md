---
layout:       post
title:        "Swift如何运用Never实现代码占位和API设计"
author:       "beforeold"
header-style: text
catalog:      true
tags:
    - Swift
    - Never
    - API Design
---

# 1 什么是 Never
Never 从字面意思理解就是“**永不**”。有以下应用场景
## 1.1 替代了早期 swift 版本中的特性 @nonreturn
例如，在一些方法暂时未来得及实现的情况下，可以使用 ```fatalError() ```函数 来实现临时占位：

这个函数的声明返回值就是 Never，该函数永远不会执行完后返回，会被编译器认为程序到此结束，因而可以临时代替函数的实现，其声明如下：
```Swift
func fatalError(_ message: @autoclosure () -> String = String(),
                     file: StaticString = #file,
                     line: UInt = #line) -> Never
```

注意：该函数在执行时使程序崩溃，因此仅用于特殊场景或占位，如下面的两个例子：
1、required init(from coder: NSCoder) 的实现，因为现在很少使用 xib/storyboard 但是作为 required 构造器必须提供实现
2、一些协议的实现需要逐个处理每个 api，可以先用 fatalErorr() 占位处理
```Swift
class MyVC: NSViewController {
    init(name: String) {
        super.init(nibName: nil, bundle: nil)
    }
    
    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
}
extension MyVC: NSCollectionViewDataSource {
    func collectionView(_ collectionView: NSCollectionView, numberOfItemsInSection section: Int) -> Int {
        fatalError()
    }
    
    func collectionView(_ collectionView: NSCollectionView, itemForRepresentedObjectAt indexPath: IndexPath) -> NSCollectionViewItem {
        fatalError()
    }
}

```
## 1.2 用于泛型参数表示不会发生的情况
比如在 Result 定义中，可以指定 Failure 为 Never。

```Swift
let ret: Result<String, Never> = .success("ok")
```

这样的 result 不可能失败，因为 Never 是无法实例化，也就无法实例化 failure 的 case。
此外，编译器也会判断出这个 Never 的 case，不需要去判断 success 即可，failure 可省略

```Swift
func foo() {
    switch ret {
//    case .failure:
//        print("never happen")
//
    case .success:
        print("always success")
    }
}
```

在 Combine 框架中也可以看到 Never 使用的身影，比如在一些只发送值的 Publisher 就可以指定其 Error 为 Never，同样是无法发送 Never 的 Error。

```Swift
let subject = PassthroughSubject<Int, Never>()
subject.send(666)
```

# 2 实现更好的占位效果和 API 设计
直接使用 fatalError() 即可代码函数的实现，还可以用于：
## 2.1 处理确实需要程序必须满足的情况
要求程序上下文必须满足特定条件才能执行，可触发 fatalError()，比如：

```Swift
func importantTask() {
    guard is_valid_user() else {
        fatalError("invalid user can not launch")
    }
    // go on 
}
```

##  2.2 用于参数的占位
对于一些参数较多，或者参数复杂的情况，无法一时间完成多个参数的输入，可以利用 fatalError() 的 Never 特性，定义一个占位函数，实现任意参数的占位：
```Swift
func undefined<T>(_ msg: String = "undefined implementation") -> T {
    fatalError(msg)
}
```
实现以下效果：
1、泛型参数返回值，可以在任何参数位置使用
2、fatalError() 作为占位代替了对应的参数传入
实际使用举例：
```Swift
let timer = Timer.init(timeInterval: undefined(),
                       target: undefined(),
                       selector: undefined(),
                       userInfo: undefined(),
                       repeats: undefined())

let timer2 = Timer(timeInterval: undefined(),
                   repeats: undefined(),
                   block: undefined())
                   
let empty = Empty(completeImmediately: false, outputType: String.self, failureType: Never.self)
_ = empty.sink(receiveCompletion: undefined(), receiveValue: undefined())
```


## 2.3 更清晰可靠的 API 定义
Never 的特性在 Combine 中还有不少 extension 的应用，在适当的场景中使用 Never 可以更好地传达 API 语义，Swift 是要将安全放在第一位。
- API 不会失败
- 相反，如果会失败的 API，在 API 设计上就要强制其必须处理 error

从 Publisher 的 extension 对比可见一斑：
- 1、通用方法，要求处理 completion 中潜在的 error
- 2、error 为 Never，可以不考虑 error

```Swift
// case 1
extension Publisher {
    public func sink(receiveCompletion: @escaping ((Subscribers.Completion<Self.Failure>) -> Void),
                     receiveValue: @escaping ((Self.Output) -> Void)) -> AnyCancellable
}

extension Publisher where Self.Failure == Never {
    // case 2
    public func sink(receiveValue: @escaping ((Self.Output) -> Void)) -> AnyCancellable
}
````

# 参考资料
1、[Apple 官方文档 - Never](https://developer.apple.com/documentation/swift/never)

2、[NSHipster - Never](https://nshipster.cn/never/)