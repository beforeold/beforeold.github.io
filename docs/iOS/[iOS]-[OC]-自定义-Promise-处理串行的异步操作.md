#### 背景
```iOS``` 应用中很多操作是异步的，比如：
- 网络请求的回调
- ```UIAlertController``` 等待用户点击事件的回调
- 定位的回调

等等，当这些操作需要逐个串行去执行时，由于不同条件下的回调结果以及不同操作的回调顺序，如果连续嵌套会导致业务严重耦合，多个操作回调加上可能的错误处理就会形成常说的**“回调地狱”**（举例如下面的伪代码），造成逻辑可读性差，且影响以后的维护。

```Objective-C
/// 回调地狱
- (void)callbackHell {
    [TestManager doSomethingWithCompletion:^{
        NSLog(@"No 1 done");
        [TestManager doSomethingWithCompletion:^{
            NSLog(@"No 2 done");
            [TestManager doSomethingWithCompletion:^{
                NSLog(@"No 3 done");
            }];
        }];
    }];
}
```

针对串行情况下的异步回调的处理，可以将异步回调进行一次封装处理，这种对一个操作执行的预期，通常称为 ```Promise```。

将每一个异步的回调进行封装后放入到 PromiseManager 处理，效果举例如下：

```Objective-C
/// 使用 Promise
- (void)testPromise {
    GSPromise *a = [GSPromise promiseWithHandler:^(dispatch_block_t then){
        [TestManager doSomethingWithCompletion:^{
            NSLog(@"done a"); then();
        }];
    }];
    
    GSPromise *b = [GSPromise promiseWithHandler:^(dispatch_block_t then){
        [TestManager doSomethingWithCompletion:^{
            NSLog(@"done b"); then();
        }];
    }];
    
    GSPromise *c = [GSPromise promiseWithHandler:^(dispatch_block_t then){
        [TestManager doSomethingWithCompletion:^{
            NSLog(@"done c"); then();
        }];
    }];
    
    GSPromiseManager *manager = [GSPromiseManager manangerWithPromises:@[a, b]];
    [manager addPromise:c];
}
```

#### 解决方案
串行操作的顺序是，一个操作结束后执行下一个操作，因此可将“执行下一个操作”这一事件，封装入一个 block 中 （称为 ```then```），并要求上一个操作执行完成后调用。

此外，将操作需要执行的方法也封装入一个 block 中 （称为 ```handler```），要求每一个操作都具备 ```handler``` 这一属性，因此将该属性声明为协议 GSPromisable（如下），要求操作去遵循。

```Objective-C
/**
 what a promise will execute

 @param then should call this block after promise done
 */
typedef void(^GSPromiseHandler)(dispatch_block_t then);

@protocol GSPromisable <NSObject>
/**
 the handler which a promise should retain
 */
@property (nonatomic, copy) GSPromiseHandler handler;

@end
```

其中，遵循 ```GSPromisable``` 协议的对象即 ```promise```， 都添加到 ```PromiseManager``` 所管理的容器数组中，每个 promise 的```handler``` 属性，将由外部的 ```PromiseManager``` 在合适的时机调用。为实现 ```PromiseManager``` 对串行的管理，```manager``` 在执行执行操作的 ```handler``` 时将上文的 ```then``` 操作传递到 ```handler``` 中，如此 ```handler``` 在完成自身操作后调用 ```then```，实现下一步操作的继续，实现如下：

```Objective-C
/// try to handle next operation
- (void)try2Start {
    if (self.isWorking) return;
    
    id <GSPromisable> first = container.firstObject;
    if (!first) return;
    
    self.working = YES;
    dispatch_block_t then = ^{
        self.working = NO;
        
        @synchronized (self) {
            if (!container.firstObject) return;
            [container removeObjectAtIndex:0];
        }
        
        [self try2Start];
    };
    
    first.handler(then);
}
```

值得一提的是，这种设计思路类似于前文的[《[iOS] [OC] 关于block回调、高阶函数“回调再调用”及项目实践》](http://www.jianshu.com/p/5d0c85f9abcf)

##### 源码及demo

[GSPromise@GitHub](https://github.com/beforeold/GSPromise)

#### 延伸
关于异步的操作处理有许多优秀的开源框架可以学习，推荐：
-  [PromiseKit](https://github.com/mxcl/PromiseKit) 
- [RxSwift](https://github.com/ReactiveX/RxSwift)

> 感谢同事 Zoe ，其对工作水平的不懈追求，促使更好方案的实现和改进。
