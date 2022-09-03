[Demo中的 XSignal+Funtional category](https://github.com/beforeold/XSignal)
#### 基于信号的响应式
在前文[《[iOS] [OC] 一步一步实现响应式(附 demo)》](http://www.jianshu.com/p/9dd3c1c7d8e3) 中实现了基础的响应式结构，响应式信号的设计具备高度的抽象能力：
- 信号的产生
- 信号的消费（订阅 next/ fail/ completed）

利用这种抽象，可以实现函数式的扩展。

#### 函数式扩展
> 在我咨询 [Casa 老师](https://casatwy.com/) 关于响应式和函数式的问题时，他给出了非常形象的比方：
> - 响应式：像是 NSNotification，是监听后等待回调的关系
> - 函数式： 像是 NSInvocation，是传递一段可执行代码的关系

在函数式中，常用的 map / filter 等方法，都是携带一个可执行代码（即 block） 后实现特定功能的设计意图，不论会有多个复杂的函数设计，结合实际的应用场景去分析，往往更容易理解。


#### 常用的函数式方法实现

- **map** 的定义是将信号的数据进行一次加工后，即想接收到 next 数据时，对 next 数据进行再一次的处理，map 函数需要一个返回新的数据的的 block 参数：
```id (^f)(id)```。
结合之前的响应式基础上，map 的定义和实现如下：
```Objective-C

@interface XSignal (XFunctional)

/// 通过 map 后返回一个新的信号
- (XSignal *)map:(id(^)(id))f;

@end

@implementation XSignal (XFunctional)

- (XSignal *)map:(id (^)(id))f {
    return [[XSignal alloc] initWithGenerator:^XDisposable(XSubscriber *subscriber) {
        return [self subscribeWithNextHandler:^(id next) { [subscriber gotNext:f(next)]; }
                                 errorHandler:^(id error) { [subscriber gotError:error]; }
                            completionHandler:^{ [subscriber completed]; }];
    }];
}

@end


```

从实现可见，订阅者既然可以获取到原信号的 error/ completion 情况，而得到 调用 参数为 f 的 block 后 map 加工后的新数据 ```f(next)```。

- **filter** 是实现数据过滤的功能，同样针对信号的 next 数据进行一次筛选，filter 函数需要一个返回布尔值的 block 参数：
```bool (^f)(id)```。
只有当 filter 的 block 条件满足时，才会将数据发送给订阅者，对应地，其声明和实现如下：
```Objective-C
/// 生成一个过滤条件的信号
- (XSignal *)filter:(BOOL(^)(id))f;

- (XSignal *)filter:(BOOL (^)(id))f {
    return [[XSignal alloc] initWithGenerator:^XDisposable(XSubscriber *subscriber) {
        return [self subscribeWithNextHandler:^(id next){ if(f(next)) [subscriber gotNext:next];}
                                 errorHandler:^(id error) { [subscriber gotError:error]; }
                            completionHandler:^{ [subscriber completed]; }];
    }];
}
```

类似的函数式特点还有很多，很值得去进一步探索学习。

#### 小结
函数式的应用中，重点是理解一下思路：
- 将 Signal 理解成信号产生源
- 而后续的函数式处理是接受一个函数（block）参数
- 再对整个信号源进行订阅后自身成为一个新的信号源。


#### 参考资料：
> - [SSignal](https://github.com/peter-iakovlev/signals)
> - [ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa)
