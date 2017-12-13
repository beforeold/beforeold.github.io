 委托模式是 iOS 开发常用的设计模式，在实现当中必要时可以做一些小优化：
- 设置 delegate 后，即需要向 delegate 获取数据或发送消息，复写 setter：
```Objective-C
- (void)setDelegate:(id<SomeProtocol>delegate {
    _delegate = delegate;
    // do something if needed.
}
```

- 协议的 optional 方法众多，每次都判断 respondsToSelector: 会比较繁琐，提供一个便利方法：
```Objective-C
- (BOOL)delegate_respondsToSelector:(SEL)selector {
    return self.delegate && [self.delegate respondsToSelector:selector];
}
```

- 进一步地，如果业务中有多处需要判断 delegate_respondsToSelector: 那么不妨将这个业务独立成一个方法来处理，比如 DZNEmptyDataSet 的处理方式：
```Objective-C
- (void)dzn_willDisappear
{
    if (self.emptyDataSetDelegate && [self.emptyDataSetDelegate respondsToSelector:@selector(emptyDataSetWillDisappear:)]) {
        [self.emptyDataSetDelegate emptyDataSetWillDisappear:self];
    }
}
```

- 再进一步的，如果是在 category 中新增了一个 delegate 对象，使用了 Runtime 的 objc_setAssociatedObject 函数进行关联，那么使用 assign 策略（因为没有 weak 策略）来实现就有潜在发生野指针异常的情况（见前文分析[《[iOS] 从 application.delegate 引申三点》](http://www.jianshu.com/p/6cd4b16c6c40)）。从 DZNEmptyDataSet 可以学习到 delegate 直接使用 retain 策略，不过 retain 的对象是一个 DZNWeakContainer，使用 container 对 delegate 进行 weak 引用，解决了 association 场景下的 assign 野指针问题。
```Objective-C
- (void)setEmptyDataSetSource:(id<DZNEmptyDataSetSource>)datasource {
    ....
        objc_setAssociatedObject(self,
                             kEmptyDataSetSource,
                             [[DZNWeakObjectContainer alloc] initWithWeakObject:datasource],
                             OBJC_ASSOCIATION_RETAIN_NONATOMIC);
     ....
}

@interface DZNWeakObjectContainer : NSObject

@property (nonatomic, readonly, weak) id weakObject;

- (instancetype)initWithWeakObject:(id)object;

@end

@implementation DZNWeakObjectContainer

- (instancetype)initWithWeakObject:(id)object
{
    self = [super init];
    if (self) {
        _weakObject = object;
    }
    return self;
}

@end
```
对应的调整 getter 方法：
```Objective-C
- (id<DZNEmptyDataSetDelegate>)emptyDataSetDelegate
{
    DZNWeakObjectContainer *container = objc_getAssociatedObject(self, kEmptyDataSetDelegate);
    return container.weakObject;
}
```

- 如果自身要成为自身的 delegate 时，则更有必要进行处理了，这部分分析可以学习 文本输入限制框架 [InputKit](https://github.com/tingxins/InputKit) 的使用，以及开发者介绍的文章 [self.delegate = self?](https://tingxins.com/2017/07/about-delegate-oc/)。



> 如果同时复写了 setter/getter，那么就意味着不需要自动合成属性变量，需要手动合成。
>```Objective-C
>@synthesize delegate = _delegate;
>```

参考：
- [DZNEmptyDataSet](https://github.com/dzenbot/DZNEmptyDataSet)
- [一句话笔记(20) （dzn_canDisplay）](http://www.jianshu.com/p/9044e9bfa969)
