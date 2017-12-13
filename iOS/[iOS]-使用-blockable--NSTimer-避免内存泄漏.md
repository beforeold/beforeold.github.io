#### 背景
> 此知识点，从阅读《Effective Objective-C》中学习到，现扩展到 Swift 并兼容 iOS 10+

NSTimer 提供定时执行任务的功能，可用于延时或者重复处理事务。使用 NSTimer 执行重复任务时（非重复任务会在触发后自动撤销 ```invalidate```），必须注意的是一个内存泄露的问题，原因是 iOS 10 以前 Timer 基于 Target-action 的 API 设计下：
OC:
```Objective-C
+ (NSTimer *)scheduledTimerWithTimeInterval:(NSTimeInterval)ti 
                target:(id)aTarget
               selector:(SEL)aSelector 
          userInfo:(nullable id)userInfo
             repeats:(BOOL)yesOrNo;
```
Swift:
```Swift
   class func scheduledTimer(timeInterval ti: TimeInterval, 
                                   target aTarget: Any, 
                            selector aSelector: Selector, 
                                userInfo: Any?,
                             repeats yesOrNo: Bool) -> Timer
```

- NSTimer 被所在的 Runloop 强引用
> Timers work in conjunction with run loops. To use a timer effectively, you should be aware of how run loops operate—see [NSRunLoop
](apple-reference-documentation://hclPs8uY7g) and [Threading Programming Guide](https://developer.apple.com/library/etc/redirect/xcode/content/1189/documentation/Cocoa/Conceptual/Multithreading/Introduction/Introduction.html#//apple_ref/doc/uid/10000057i). Note in particular that run loops maintain strong references to their timers, so you don’t have to maintain your own strong reference to a timer after you have added it to a run loop.
- NSTimer 需要在安装的线程 / runloop 进行移除，如果从非安装线程进行移除，那么 timer 可能无法从 Runloop 中移除，也将导致相关的线程、target 和 userInfo 无法释放，**慎用子线程 NSTimer !!**
>You must send this message from the thread on which the timer was installed. If you send this message from another thread, the input source associated with the timer may not be removed from its run loop, which could prevent the thread from exiting properly.

- NSTimer 会强引用其 target

> The object to which to send the message specified by aSelector when the timer fires. The timer maintains a strong reference to target until it (the timer) is invalidated.


也就是说，如果是在主线程，即便调用 timer 的对象（target）没有对 timer 进行强引用（比如说使用了 weak 弱引用），因为 NSRunloop -> NSTimer -> target，而导致 target 无法释放，除非为 target 独立设计一个 stopTimer 的方法。
OC:
```Objective-C
- (void)stopTimer {
    [_timer invalidate];
}
```
Swift：
```
func stopTimer() {
      timer.invalidate()
}
```
如果单单在 dealloc/deinit 处执行撤销 Timer 的操作无法达到预期效果，因为 target 被强引用不会调用销毁/析构的方法。
OC:
```Objective-C
- (void)dealloc {
    [_timer invalidate];
}
```
Swift:
```Swift
deinit {
     timer.invalidate()
}
```

#### 解决方案

逆向地看：
- 不让 timer 强引用调用者，如此调用者可以正常释放，在调用者释放时即可顺利地撤销 Timer
- 那么选择一个类对象作为 target 是合适的，因为类对象本身就是单例的存在，不存在多余的泄露问题。
- 当 target 成为一个类对象后，Timer 将通过 selector 回调类对象，此时思考如何将定时器事件如何回调给原来的调用者，对比 delegate/notification/block，使用 block 更合适，这样业务紧凑，且无需更多的配套参数。使用 block 作为一个 timer 的 userInfos，在 timer 回调时调用 block 即可：

OC:
```Objective-C
@implementation NSTimer (GSBlockable)

+ (NSTimer *)gs_scheduledTimerWithTimeInterval:(NSTimeInterval)interval
                                       repeats:(BOOL)repeats
                                         block:(void (^)(NSTimer * _Nonnull))block
{
    if ([UIDevice currentDevice].systemVersion.doubleValue <= 10.0) {
        return [self scheduledTimerWithTimeInterval:interval
                                                target:self
                                              selector:@selector(gs_timerTick:)
                                              userInfo:[block copy]
                                               repeats:repeats];
    } else {
        return [self scheduledTimerWithTimeInterval:interval
                                                 repeats:repeats 
                                                block:block];
    }
}


+ (void)gs_timerTick:(NSTimer *)timer {
    void (^block)(NSTimer *) = timer.userInfo;
    !block ?: block(timer);
}

@end

```
Swift:
```Swift
extension Timer {
    open class func gsx_scheduledTimer(withTimeInterval interval: TimeInterval,
                                       repeats: Bool,
                                       block: @escaping (Timer) -> Swift.Void) -> Timer
    {
        if false {
            return scheduledTimer(withTimeInterval: interval,
                                               repeats: repeats, 
                                                  block: block);
            
        } else {
            return scheduledTimer(timeInterval: interval, 
                                  target: self,
                               selector: #selector(Timer.gsx_timerTick), 
                              userInfo: block, 
                          repeats: repeats)
        }
    }
    
    open class func gsx_timerTick(timer: Timer) {
        if let  block = timer.userInfo as? (Timer)-> Swift.Void {
            block(timer)
        }
    }
}
```

- 此时则需要注意不能让 block 强引用了调用者，应该在 block 内使用调用着的 weak 引用，否则仍然会造成内存泄漏， Runloop -> Timer -> UserInfo(block) -> sender。
OC:
```Objective-C
    __weak typeof(self) weakSelf = self;
    self.timer = [NSTimer gs_scheduledTimerWithTimeInterval:3 
                                               repeats:YES
                                               block:^(NSTimer * _Nonnull timer) {
        [weakSelf printSomething];
    }];

- (void)printSomething {
    NSLog(@"TimerVC ....");
}

- (void)dealloc {
    NSLog(@"timervc dealloc");
    [_timer invalidate];
}

```

Swift:
```Swift
timer = Timer.gsx_scheduledTimer(withTimeInterval: 3, 
                                repeats: true, 
                                block: {[weak self](theTimer) in
            self?.printSth()
        })

    func printSth() {
        print("tick")
    }
    
    deinit {
        print("deinit")
        timer?.invalidate()
    }
```


#### 小结
1. 使用类对象对 timer 进行 block 封装来避免 timer，
2. 使用 block 封装 timer 的回调逻辑
3. 在 block 内使用 weak 引用避 block 潜在的强应用
4. 在 dealloc/deinit 时，撤销 timer
5. 当系统版本达到 iOS 10 时直接使用已有 API 即可。

#### update 2017-10-18
- 从 [BlocksKit](https://github.com/BlocksKit/BlocksKit/blob/52671356f157fd76012bfe734bd99b5da1e8db07/BlocksKit/Core/NSTimer%2BBlocksKit.m) 中学习到 利用 CoreFoundation 框架中 CGTimer 的函数应用 ```CFRunLoopTimerCreateWithHandler()```
- 从 YYKit 中了解到一种使用 [YYWeakProxy](https://github.com/ibireme/YYKit/blob/3869686e0e560db0b27a7140188fad771e271508/YYKit/Utility/YYWeakProxy.m)（基于 NSProxy） 的方法来避免 Timer 对 target 强引用的方式。（在评论中 未来行者同学也有提及类似方案。）

#### PS
> 使用 GCD 亦可实现安全的 ```block + timer```
> - 文章 [选择 GCD 还是 NSTimer ？](http://www.jianshu.com/p/0c050af6c5ee) 
>- 开源库 [MSWeakTimer](https://github.com/mindsnacks/MSWeakTimer)
