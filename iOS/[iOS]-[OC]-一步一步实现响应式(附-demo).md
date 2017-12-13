#### demo
[GitHub-XSignal](https://github.com/beforeold/XSignal)

#### 信号
信号是数据的源头，宽泛地说信号也是信号产生、订阅的抽象汇总。定义一个信号的功能则分为两部分：产生和订阅。

#### 信号的产生
信号的产生，是获取获取数据和发送数据的过程，将这一过程定义到一个任务 block 中，

```Objective-C
/// 定义产生器任务 block
XDisposable(^XGenerator)(XSubscriber)

// XSignal 信号构造方法和属性的声明
- (instance)initWithGenerator:(XGenerator)generator;
@property (nonatomic, copy, readonly) XGenerator generator;

//实现
- (instancetype)initWithGenerator:(XGenerator)generator {
    self = [super init];
    if (self) {
        _generator = [generator copy];
    }
    return self;
}
```
由此，信号具备了产生数据和发送数据的功能，而这部分功能则封装在 Generator 任务 block 内，从而使 Signal 具有足够的灵活抽象性。

其中，Subscriber 则信号的订阅者类型，Disposable 是订阅后生成的存根，利用存根可以在需要时取消信号的订阅，这里暂不给出具体的存根和订阅者类型：
```
typedef ... Disposable
typedef ... Subscriber
```

#### 信号的订阅
将信号的产生和消费过程进行抽象，可以分为三类：
- 发送数据 next
- 出现异常错误 error
- 信号结束 completion

信号的订阅者可以针对性地对这三种情况进行处理，由此一个订阅者可由数据处理、异常处理和完成处理构成。定义订阅者类 XSubscriber:

```Objective-C
@interface XSubscriber : NSObject

- (instancetype)initWithNextHandler:(void(^)(id))nextHandler
errorHandler:(void(^)(id))errorHandler
completionHandler:(void(^)(void))completionHandler;

@end
```
对应在内部保存这些处理方式：

```Objective-C
@interface XSubscriber ()

@property (nonatomic, copy, readonly) void(^nextHandler)(id);
@property (nonatomic, copy, readonly) void(^errorHandler)(id);
@property (nonatomic, copy, readonly) void(^completionHandler)(void);

@property BOOL terminated;

@end

@implementation XSubscriber
- (instancetype)initWithNextHandler:(void (^)(id))nextHandler
                        errorHandler:(void (^)(id))errorHandler
                   completionHandler:(void (^)(void))completionHandler
{
    self = [super init];
    if (self) {
        _nextHandler = [nextHandler copy];
        _errorHandler = [errorHandler copy];
        _completionHandler = [completionHandler copy];
    }

    return self;
}
```

在订阅者类，定义完成处理方式后，还需要提供入口，让信号可以发送消息给订阅者，包括数据、数据和完成：
```Objective-C

- (void)gotNext:(id)next;
- (void)gotError:(id)error;
- (void)completed;

```
订阅者接收到信号发来的消息后，调用 handler 进行对应的处理。注意：当接收到异常或者完成时，订阅者即不再处理之后收到的消息。
```Objective-C
- (void)gotNext:(id)next {
    if (self.terminated) return;

    !self.nextHandler ?: self.nextHandler(next);
}

- (void)gotError:(id)error {
    if (self.terminated) return;

    !self.errorHandler ?: self.errorHandler(error);
    self.terminated = YES;
}

- (void)completed {
    if (self.terminated) return;

    !self.completionHandler ?: self.completionHandler();
    self.terminated = YES;
}
```

为信号新增一个订阅者，订阅时即启用信号产生器。

```Objective-C
- (XDisposable)subscribeWithNextHandler:(void(^)(id))nextHandler
            errorHandler:(void(^)(id))errorHandler
            completionHandler:(void(^)(void))comletionHandler;
            
/// 实现
- (XDisposable)subscribeWithNextHandler:(void (^)(id))nextHandler
        errorHandler:(void (^)(id))errorHandler
            completionHandler:(void (^)(void))comletionHandler
{
    XSubscriber *sub = [[XSubscriber alloc] initWithNextHandler:nextHandler
                                            errorHandler:errorHandler
                                            completionHandler:comletionHandler];
    XDisposable subbedDisposable = _generator(sub); // 启用产生器
    
    /// 返回存根
    return [[XSubscriberDisposable alloc] initWithSubscriber:sub
                                             disposable:subbedDisposable];
}

```
在这里，订阅者与完成信号产生后的存根组合成为一个新的存根，XSubsriberDisposable，在订阅完成后返回，这个细节以后需要进一步完善，初级的组合存根对象声明如下：
```Objective-C
@interface XSubscriberDisposable : NSObject

- (instancetype)initWithSubscriber:(XSubscriber *)subscriber
            disposable:(XDisposable)XDisposable;

@end

@interface XSubscriberDisposable ()

@property (nonatomic, readonly) XSubscriber *subscriber;
@property (nonatomic, readonly) XDisposable disposable;

@end

@implementation XSubscriberDisposable
- (instancetype)initWithSubscriber:(XSubscriber *)subscriber
                disposable:(XDisposable)disposable {
    self = [super init];
    if (self) {
        _subscriber = subscriber;
        _disposable = disposable;
    }

    return self;
}

@end
```
至此，实现了信号的产生、订阅和消费，写一个简单的测试代码如下：
```
    XSignal *signal = [[XSignal alloc] initWithGenerator:^XDisposable(XSubscriber *subscriber) {
        [subscriber gotNext:@1];
        [subscriber completed];
        
        return nil;
    }];
    
    [signal subscribeWithNextHandler:^(id next) {
        NSLog(@"next -> %@", next);
    } errorHandler:nil completionHandler:^{
        NSLog(@"signal completed");
    }];
```

#### 应用举例
基于响应式的设计，应用非常广泛，下面以 Signal 封装实现 UIButton 的 target-action 事件为例。
其中，实现的思路如下：
- 以 category 的形式，为 UIButton 对象提供可以获取点击事件信号的属性 getter 方法
- 通过 associate 的形式为按钮关联一个 signal 对象
- 信号的产生任务 block，是通过封装一个 TargetAction 对象成为按钮的事件的响应者
- 而 Target-action 对象同时也持有了 订阅者，从而在收到点击事件时，发送消息给订阅者
```Objective-C
@interface UIButton (XSignal)

@property (nonatomic, strong) XSignal *x_signal;

@end


@interface XButtonTargetAction : NSObject

@property (nonatomic, weak) XSubscriber *subscriber;

- (void)onClick:(id)sender;

@end

@implementation XButtonTargetAction

- (void)onClick:(id)sender {
    [self.subscriber gotNext:sender];
}

@end

@implementation UIButton (XSignal)
- (XSignal *)x_signal {
    XSignal *signal = objc_getAssociatedObject(self, _cmd);
    if (!signal) {
        signal = [self x_createSignal];
        objc_setAssociatedObject(self, _cmd, signal, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }
    
    return signal;
}

- (XSignal *)x_createSignal {
    __weak typeof(self) weakSelf = self;
    XSignal *signal = [[XSignal alloc] initWithGenerator:^XDisposable(XSubscriber *subscriber) {
        XButtonTargetAction *holder = [[XButtonTargetAction alloc] init];
        holder.subscriber = subscriber;
        [weakSelf addTarget:holder action:@selector(onClick:) 
                                forControlEvents:UIControlEventTouchUpInside];
        return holder;
    }];
    
    return signal;
}

@end

```
使用信号的方式，对按钮的事件进行订阅，如下：
```Objective-C
- (void)testSubscribe {
    self.disposable = [self.button.x_signal
                       subscribeWithNextHandler:^(id next) { NSLog(@"next -> %@", next); }
                       errorHandler:nil
                       completionHandler:nil];
    NSLog(@"subscribed %@", self.button);
}
```

#### 小结

通过拆解响应式的基本原理，初步实现了一个响应式的结构，在[[iOS] [OC] 以 Uber-signals 一窥响应式](http://www.jianshu.com/p/0a58b646d0c8) 的基础上，进一步地加深了对响应式的认识，对比其他成熟的框架而言，这次实现主要是梳理信号和订阅的关系，有好些不足：
- 订阅后存根的处理
- 对象的引用关系
- 多线程访问的问题
- 函数式的支持
