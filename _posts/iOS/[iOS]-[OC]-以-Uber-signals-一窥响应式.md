#### 简介 Signals

[Singals](https://github.com/uber/signals-ios) 是 Uber 开源的一个 ```OC``` 事件处理库，在不使用代理或者通知的形式实现以可观察的模式。

> Signals is an eventing library that enables you to implement the Observable pattern without using error prone and clumsy NSNotifications or delegates.


举例，将一个网络请求的对象定义为信号发送者 ```Emitter```，声明一个信号属性
```Objective-C
/// 定义信号类型以及回调参数格式
CreateSignalType(NetworkResult, NSData *result, NSError *error)

@interface UBNetworkRequest

@property (nonatomic, readonly) UBSignal<NetworkResultSignal> *onNetworkResult;

@end
```

外部对这个属性进行观察（监听）

```Objective-C
[networkRequest.onNetworkResult addObserver:self 
            callback:^(typeof(self) self, NSData *data, NSError *error) {
    // Do something with the result. The self passed into the block is 
    // weakified by Signals to guard against retain cycles.
}];
```

在 ```Emitter``` 内部的实现是这样的:
```Objective-C
- (instancetype)init {
  self = [super init];
  if (self) {
      /// 在构造器中创建一个特定的信号，此处举例是网络请求完成的信号
    _onNetworkResult = (UBSignal<NetworkResultSignal> *)
         [[UBSignal alloc] initWithProtocol:@protocol(NetworkResultSignal)];
   }
   return self;
}

/// 网络请求返回数据或者错误
- (void)receivedNetworkResult(NSData *data, NSError *error) 
{
  /// 通知所有监听者 数据或者错误
  _onNetworkResult.fire(myData, myError);
}

...

@end
```


#### 分析实现原理

```Signals``` 十分轻量级，核心的类是两个：```UBSignalObserver``` 和 ```UBBaseSignal``` ，```UBsignalObserver``` 封装了观察者的信息，包括观察者、回调 ```block``` 及回调队列。

```Objective-C
- (instancetype)initWithSignal:(UBBaseSignal *)signal
                      observer:(id)observer
                         queue:(NSOperationQueue *)queue
                      callback:(UBSignalCallback)callback;
```

```UBBaseSignal``` 封装了观察者的添加、移除和信号的触发、回调观察者。理论上，到此为止已经实现了信号的发送和回调，但实践时的一个问题是 ```block``` 回调的参数格式和个数的处理，作为 ```Signals``` 另一个核心是使用宏处理 ```#define``` 的方式，为特定的业务，自定义回调的参数格式和数量的回调参数列表并以协议的形式提供了添加观察者的接口。

```Objective-C
/// 信号构造器
- (instancetype)initWithProtocol:(Protocol *)protocol;

/// 可变参数列表
typedef void (^UBSignalCallback) (id listener, ...);
/// 为一个信号对象添加一个观察者 listener/ observer
- (UBSignalObserver *)addObserver:(id)observer 
                         callback:(UBSignalCallback)callback
```


```Objective-C
/// 根据名称和参数列表定义一套协议
#define CreateSignalType(name, signature...)\
    CreateSignalType_(PP_NARG(signature),name,signature)
```

其实现如下：

```Objective-C
#define CreateSignalType_(signatureParameterCount, name, signature...) \
    CreateSignalType__(signatureParameterCount, name, signature)

#define CreateSignalType__(signatureParameterCount, name, signature...) \
    @protocol name ## Signal <UBSignalArgumentCount ## signatureParameterCount>\
    - (UBSignalObserver *)addObserver:(id)observer callback:(void (^)(id self, signature))callback; \
    - (UBSignalObserver *)addObserver:(id)observer queue:(NSOperationQueue *)queue callback:(void (^)(id self, signature))callback; \
    - (void (^)(signature))fire; \
    - (void (^)(UBSignalObserver *signalObserver, signature))fireForSignalObserver; \
    @end
```

上述宏定义的方式，在 GCD 中也有类似的使用，比如 ```DISPATCH_DECL``` 这个宏定义。上述宏定义定义的是一个协议，其声明了一套 ```API```，这套 API 在 ```UBBaseSignal``` 的实现中已经做好处理，而宏定义正是利用协议的形式为不同的 block 参数列表定义了特定的接口，便于业务的明确使用。目前 ```Signals``` 最多支持协议定义 ```block``` 有 5 个参数，并在内部封装了不同个数参数列表的父协议 ```super protocol```，从而可以根据业务宏定义的信号协议 ```signal protocol``` 的继承关系确定了参数个数。

```Objective-C
/// 空协议的另一种用法，做一些内部约定
@protocol UBSignalArgumentCount0 @end
@protocol UBSignalArgumentCount1 @end
@protocol UBSignalArgumentCount2 @end
@protocol UBSignalArgumentCount3 @end
@protocol UBSignalArgumentCount4 @end
@protocol UBSignalArgumentCount5 @end

```

信号的触发和调用，同样转换为 ```block``` 的形式，在 ```UBBaseSignal``` 定义了两个只读  ```readonly``` 的 ```block``` 类型属性，以空数据信号为例：
```Objective-C
/**
 Returns a block that fires the signal when invoked.
 */
- (void (^)(void))fire;

/**
 Returns a block that fires the signal for a specific observer when invoked.
 */
- (void (^)(UBSignalObserver *signalObserver))fireForSignalObserver;
```

在 ```UBBaseSignal``` 内部的调用逻辑，在构造信号时，同时构造了触发回调的 ```fire``` 逻辑，根据父协议的参数个数，构造了 作为 ```block``` 类型属性 ```fire``` 的实现，以两个参数的 ```block``` 回调的实现为例：
```Objective-C
else if (protocol_conformsToProtocol(protocol, @protocol(UBSignalArgumentCount2))) {
            _fire = ^void(id arg1, id arg2) {
                __strong typeof(weakSelf) strongSelf = weakSelf;
                [strongSelf _fireNewData:@[WrapNil(arg1), WrapNil(arg2)] 
                      forSignalObservers:strongSelf.signalObservers];
            };
            _fireForSignalObserver = ^void(UBSignalObserver *signalObserver, id arg1, id arg2) {
                __strong typeof(weakSelf) strongSelf = weakSelf;
                [strongSelf _fireNewData:@[WrapNil(arg1), WrapNil(arg2)] 
                        forSignalObservers:@[signalObserver]];
            };
        }
```
可见，这里最多的处理是 ```block``` 的参数列表的处理，其中 ```fireNewData: forSignalObservers:```方法的最终实现如下：

```Objective-C
- (void)_fireData:(NSArray *)arguments forSignalObservers:(NSArray *)signalObservers
{
    NSAssert(arguments.count < 6, @"A maximum of 5 arguments are supported when firing a Signal");

    [self _purgeDeallocedListeners];

    id arg1, arg2, arg3, arg4, arg5;
    switch (arguments.count) {
        case 5:
            arg5 = arguments[4] != [NSNull null] ? arguments[4] : nil;
        case 4:
            arg4 = arguments[3] != [NSNull null] ? arguments[3] : nil;
        case 3:
            arg3 = arguments[2] != [NSNull null] ? arguments[2] : nil;
        case 2:
            arg2 = arguments[1] != [NSNull null] ? arguments[1] : nil;
        case 1:
            arg1 = arguments[0] != [NSNull null] ? arguments[0] : nil;
    }

    NSArray *signalObserversCopy;
    @synchronized(_signalObservers) {
        signalObserversCopy = [signalObservers copy];
    }
    for (UBSignalObserver *signalObserver in signalObserversCopy) {
        __strong id observer = signalObserver.observer;

        void (^fire)() = ^void() {
            UBSignalCallback callback = signalObserver.callback;

            if (signalObserver.cancelsAfterNextFire == YES) {
                [signalObserver cancel];
            }

            switch (arguments.count) {
                case 0:
                    callback(observer); break;
                case 1:
                    ((UBSignalCallbackArgCount1)callback)(observer, arg1); break;
                case 2:
                    ((UBSignalCallbackArgCount2)callback)(observer, arg1, arg2); break;
                case 3:
                    ((UBSignalCallbackArgCount3)callback)(observer, arg1, arg2, arg3); break;
                case 4:
                    ((UBSignalCallbackArgCount4)callback)(observer, arg1, arg2, arg3, arg4); break;
                case 5:
                    ((UBSignalCallbackArgCount5)callback)(observer, arg1, arg2, arg3, arg4, arg5); break;
            }
        };

        if (signalObserver.operationQueue == nil || NSOperationQueue.currentQueue == signalObserver.operationQueue) {
            fire();
        } else {
            [signalObserver.operationQueue addOperationWithBlock:^{
                fire();
            }];
        }
    }
}
```
这一段代码的核心是：
- 遍历所有需要的观察者
- 根据参数列表类型，设定 ```block``` 的回调格式和数量
- 根据观察者的回调回调，执行回调

#### 小结
比之传统的 ```delegate``` 和 ```NSNotifcation```，```block``` 可以保存一段回调逻辑，并可捕获上下文变量，所以 ```Signals``` 可以保存回调逻辑，在适当的信号触发逻辑调用，而具备很大的灵活性。

同时 ```block``` 可以作为方法 / 函数的参数、返回值使用，充分地加以利用，可以实现更多有趣的特性，这一点需要在一些响应式的框架中去继续挖掘，比如经典的 ```ReactiveCocoa``` 和 ```RxSwift``` 等。
