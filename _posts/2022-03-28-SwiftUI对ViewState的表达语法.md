---
layout:       post
title:        "SwiftUI对ViewState的表达语法"
author:       "beforeold"
header-style: text
catalog:      true
tags:
    - SwiftUI
    - MVVM
---

# 背景
在 SwiftUI 中 View 可以理解为 State 的运算结果，View = f(State)，在处理映射关系中，比：在[一篇分析文章](https://juejin.cn/post/7034308534460383245)中定义了如下 ViewState 类型，并试图通过扩展的方式映射到 SwiftUI View。
```Swift

typealias BuilderWidget<T> = (T) -> some View

enum ViewState<T: Codable> {
    case loading
    case error
    case success(ViewSuccess)
    
    struct HttpRespone<T> {
        let data: T
    }
    
    enum ViewSuccess {
        case noData
        case content(BuilderWidget<T>, HttpRespone<T>)
    }
}

extension ViewState: View {
/// 这个some View 返回的是some View
/// 但是必须是一个唯一确定的类型,比如你在.error中返回EmptyView(),那么就会马上报错,一旦确定是返回是Text,那么必须都是Text, 这也导致了BuilderView这闭包无法使用    

    var body: some View {
        switch self {
        case .error:
            return Text("")
        case .loading:
            return Text("")
        case .success(let successState):
            switch successState {
            case .noData:
                return Text("")
            case .content(let builder, let resp):
                return builder(resp.data)
            }
        }
    }
}

```

此处由于  SwiftUi 对 body 的 some View 不透明返回值类型的设定，不同的 case 要求的返回值类型需要保持一致，从语法层面当前的扩展无法编译通过。

# 解决方案
## 方案一：AnyView 的使用
使用 AnyView 类型，这就满足了那篇文章中的 ContainerView 的需求，即每一个返回的地方都使用 AnyView 进行包裹。
```Swift
func createAnyView<T>(_ value: T) -> AnyView {
    return AnyView(Text("value"))
}

```
**注意：这个方式是不太可取的，AnyView 会擦除本身的 View 类型，失去了 SwiftUI 的明确的结构，不利于视图的刷新和动画，直接影响视图性能**

## 方案二：正确理解和使用 ViewBuilder
SwiftUI 的 some View 的使用，因为 ViewBuilder 这一  resultBuilder 的特性，使得 body 的构造有充分的灵活性和组合能力。
例如将 BuilderWidget 的声明新增一个 View 的泛型参数，同时也对 ViewState 新增 View 类型参数，上述的 body 代码就可以编译通过。
**注意，SwiftUI 对 switch 的支持要求是同类型，下面代码中使用的 if case 的形式，ViewBuilder 有更好的支持** 

```Swift

typealias BuilderWidget<T, V: View> = (T) -> V

enum ViewState<T: Codable, V: View> {
    case loading
    case error
    case success(ViewSuccess)
    
    struct HttpRespone<T> {
        let data: T
    }
    
    enum ViewSuccess {
        case noData
        case content(BuilderWidget<T, V>, HttpRespone<T>)
    }
}

extension ViewState: View {
    var body: some View {
        if case .success(let result) = self {
            if case .content(let builder, let resp) = result {
                builder(resp.data)
            } else {
                Text("no data")
            }
        }
        else if case .loading = self {
            ProgressView()
        }
        else {
            EmptyView()
        }
    }
}

```
而 body 的实际类型，不是某一个分支的 view，而是一个组合后的类型，通过打印得知如下：
```Swift
_ConditionalContent<_ConditionalContent<_ConditionalContent<Button<Text>, Text>, ProgressView<EmptyView, EmptyView>>, EmptyView>
```

不过这样以来， VIewState 就包含了 View 的信息，不符合架构上的职责隔离，ViewState 不负责 BuilderWidget 更合适，可以抽象一个新的结构进行 view 的构造，例如：
```Swift

typealias BuilderWidget<T: Codable, V: View> = (T) -> V

enum ViewState<T: Codable> {
    case loading
    case error
    case success(ViewSuccess)
    
    struct HttpRespone<T> {
        let data: T
    }
    
    enum ViewSuccess {
        case noData
        case content(HttpRespone<T>)
    }
}

struct ViewMaker<T: Codable, V: View>: View {
    let viewState: ViewState<T>
    let builder: BuilderWidget<T, V>
    
    var body: some View {
        if case .success(let result) = viewState {
            if case .content(let resp) = result {
                builder(resp.data)
            } else {
                Text("no data")
            }
        }
        else if case .loading = viewState {
            ProgressView()
        }
        else {
            EmptyView()
        }
    }
}

```

# 小结
对于 SwiftUI 的语法特性，对 ViewBuilder 和泛型有很好的应用，可以从其声明中进一步学习，先从 [SwiftUI 完善的官方教程](https://developer.apple.com/tutorials/swiftui/)开始。

# 参考文章

- [UI = f(State)，在Swift中的一点思考](https://juejin.cn/post/7034308534460383245)

- [ViewBuilder](https://developer.apple.com/documentation/swiftui/viewbuilder)

- [resultBuilder](https://github.com/apple/swift-evolution/blob/main/proposals/0289-result-builders.md)