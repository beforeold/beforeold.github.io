#### 关于 [CYLDeallocBlockExecutor](https://github.com/ChenYilong/CYLDeallocBlockExecutor)
如果需要在一个对象释放后回调一个消息，那么可以为这个对象提供一个 block 属性，复写 dealloc 方法调用该 block 即可实现：

```Objective-C

@implementation XSome
- (void)dealloc  {
    !self.block ?: self.block();
}
@end
```
如果再广泛一点，希望在任意一个对象释放时可以得到回调呢？此时可以使用 CYLDeallocBlockExecutor 这个工具类，其原理是：
- 将回调的消息封装到一个对象 BlockWrapper 中，并在这个对象释放时调用自身所持有的 block：

```Objective-C
@interface XLastWords : NSObject
- (instancetype)initWithWords:(dispatch_block_t)words;
- (void)invalidate;
@end

@implementation XLastWords{
    dispatch_block_t _lastWords;
}

- (instancetype)initWithWords:(dispatch_block_t)words {
    self = [super init];
    if (self) {
        _lastWords = words;
    }
    
    return self;
}

- (void)invalidate {
    _lastWords = nil;
}

- (void)dealloc {
    !_lastWords ?: _lastWords();
}

@end
```
- 下一步，将这个对象通过 objc association 绑定（retain）到任意对象中，并提供一个添加 block 的方法，实现了任意对象释放时，其绑定的 BlockWrapper 释放，进而调用 block 的设计：
```Objective-C
#import <objc/runtime.h>
@interface NSObject (XLastWords)

- (void)x_leave:(dispatch_block_t)lastWords;

@end

static const void *kXLastWordsHolder = &kXLastWordsHolder;
@implementation NSObject (XLastWords)

- (void)x_leave:(dispatch_block_t)lastWords {
    XLastWords *old = objc_getAssociatedObject(self, kXLastWordsHolder);
    [old invalidate];
    
    XLastWords *words = [[XLastWords alloc] initWithWords:lastWords];
    objc_setAssociatedObject(self, kXLastWordsHolder, words, OBJC_ASSOCIATION_RETAIN);
}

@end
```
当然，可以针对一个对象留多个 dealloc - block，具体的实现可以查看 CYLDeallocBlockExecutor 的源码。

#### 延伸之应用场景
在 CYLDeallocBlockExecutor 的介绍中有三种应用场景：
- 第一种：移除监听，好处是添加监听和移除监听逻辑写在一个地方，不需要专门去实现 dealloc 方法。
需要注意的是，在 block 内不能使用 self，因为对 BlockWrapper 的 retain 的关系，直接使用 self 将造成循环引用。其实此时 BlockWrapper 的 dealloc 调用时 self 已经释放时，此时要获取到 self，只能使用 __unsafe_unretained 关键字 而不是 __weak 关键字：
```Objective-C
[[NSNotificationCenter defaultCenter] addObserver:self
                                        selector:@selector(themeChanged:)
                                        name:kThemeDidChangeNotification
                                        object:nil];

__unsafe_unretained typeof(self) unsafeSelf = self;
[self x_leave:^{
    [[NSNotificationCenter defaultCenter] removeObserver:unsafeSelf];
}];
```
这也是文章标题，是“终了”不是“临终”，是因为 block 调用时对象已经释放了。
> PS： __typeof(self) 作为操作符，写在 block 内部是不会造成循环引用的，以前都没有留意到，这**很可能**是因为 __typeof() 作为操作符，是编译用的获取类型的操作符，而不是运行时的函数。

- 第二种：基于第一点，因为是监听和移除在一个地方，则不会存在过度移除的问题，这种情况在 viewDidLoad做监听时可能会出现，因为控制器虽然构造了实例，但是并没有调用 viewDidLoad 却在 dealloc 移除了 KVO 导致过度移除的异常，这种情况在前文[《[iOS] [OC] NSNotificationCenter 进阶及自定义（附源代码）》](http://www.jianshu.com/p/4b2dff2bbdb4) 中有讨论。

- 第三种：实现 weak 属性效果。如果不适用 weak 关键字，如何实现 weak 属性？结合 weak 指针所指向的对象释放后指针自动置为 nil 的特点，这个问题可以采用此 DeallocBlockExecutor 方案，虽然有些玩票，但是可以开一开脑洞。

```Objective-C
@interface XSome : NSObject

- (void)weak_setObject:(id)object;
- (id)weak_object;

@end

@implementation XSome

- (void)weak_setObject:(id)object {
    objc_setAssociatedObject(self, @selector(weak_object), object, OBJC_ASSOCIATION_ASSIGN);
    
    [object x_leave:^{
        [self weak_setObject:nil]; // 对象释放时置为 nil
    }];
}

- (id)weak_object {
    return objc_getAssociatedObject(self, _cmd);
}

@end
```
这个场景正好可以满足前文[《[iOS] [OC] 更安全的 association weak 属性》](http://www.jianshu.com/p/c9c1bcc74228) 的需求，即使用 BlockWrapper 代替 WeakWrapper 实现安全的 object weak assocaition，方案的思想都很有趣。


- 拓展一下第四种，是确实无法得知一个对象的回调情况下的这种处理，比如跳转一个 WebView 页面进行支付，但是无法直接获取回调，则使用此方案监听其页面 dealloc 时进行特定业务的刷新或者请求。
