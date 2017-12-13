#### 背景
因为 OC 中 无法直接为类新增属性（继承、私有 extension 除外），那么通过 category 结合 object association 是常用的为一个类添加属性的手段。主要是用到 <objc/runtime.h> 的两个函数：
```
OBJC_EXPORT void
objc_setAssociatedObject(id _Nonnull object, const void * _Nonnull key,
                         id _Nullable value, objc_AssociationPolicy policy)

OBJC_EXPORT id _Nullable
objc_getAssociatedObject(id _Nonnull object, const void * _Nonnull key)
    OBJC_AVAILABLE(10.6, 3.1, 9.0, 1.0, 2.0);

```
从而为一个对象新增属性：
```Objective-C
#import <objc/runtime.h>
@interface NSObject (XAssociate)

@property (nonatomic, weak) id xProperty;

@end

@implementation NSObject (XAssociate)

- (void)setXProperty:(id)xProperty {
    objc_setAssociatedObject(self, @selector(xProperty), xProperty, OBJC_ASSOCIATION_ASSIGN);
}

- (id)xProperty {
    return objc_getAssociatedObject(self, _cmd);
}

@end
```
回到标题的 weak ，由于 objc association 提供的策略中，没有直接提供 weak 的策略，一般情况下折中使用 上文实例代码中的assign 的方式：
```Objective-C
typedef OBJC_ENUM(uintptr_t, objc_AssociationPolicy) {
    OBJC_ASSOCIATION_ASSIGN = 0,           /**< Specifies a weak reference to the associated object. */
    OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1, /**< Specifies a strong reference to the associated object. 
                                            *   The association is not made atomically. */
    OBJC_ASSOCIATION_COPY_NONATOMIC = 3,   /**< Specifies that the associated object is copied. 
                                            *   The association is not made atomically. */
    OBJC_ASSOCIATION_RETAIN = 01401,       /**< Specifies a strong reference to the associated object.
                                            *   The association is made atomically. */
    OBJC_ASSOCIATION_COPY = 01403          /**< Specifies that the associated object is copied.
                                            *   The association is made atomically. */
};
```
使用 assign 策略本质上是保存了对象的地址而不是真正的弱引用，在一些情定的情况下 属性对象释放时再调用方法会出现野指针异常，在前文有讨论[《[iOS] 从 application.delegate 引申三点
》](http://www.jianshu.com/p/6cd4b16c6c40)。

#### 解决方案
从 [DZNEmptyDataSet](https://github.com/dzenbot/DZNEmptyDataSet) 中可以学习使用 strong + WeakContainer 的方式，实现对属性对象的 weak 引用。
思路是：
- 声明一个 WeakContainer 类对真实的属性对象进行 weak 属性引用
- 通过 ```OBJC_ASSOCIATION_RETAIN_NONATOMIC``` 策略对 WeakContainer 进行 retain association
- 这样在 get 关联属性对象时由于 WeakContainer 对真是属性对象的 weak 引用，会返回 nil 而不是野指针

实现 WeakContainer 如下：
```Objective-C
@interface XWeakObjectContainer : NSObject

@property (nonatomic, readonly, weak) id weakObject;

- (instancetype)initWithWeakObject:(id)object;

@end

@implementation XWeakObjectContainer

- (instancetype)initWithWeakObject:(id)object {
    self = [super init];
    if (self) {
        _weakObject = object;
    }
    
    return self;
}

@end
```
从而变相地实现 weak  association 如下：
```Objective-C
@implementation NSObject (XAssociate)

- (void)setXProperty:(id)xProperty {
    XWeakObjectContainer *container = [[XWeakObjectContainer alloc] initWithWeakObject:xProperty];
    objc_setAssociatedObject(self, @selector(xProperty), container, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

- (id)xProperty {
    XWeakObjectContainer *container = objc_getAssociatedObject(self, _cmd);
    return container.weakObject;
}

@end
```

#### 小结
虽然 retain 了一个 WeakContainer，但是 WeakContainer 最终会随着属性的持有对象一起销毁，不存在泄露。

> update 2017-11-02
采用 后文方案亦可实现。
[《[iOS] [OC] 利用 CYLDeallocBlockExecutor 为对象添加终了遗言》](http://www.jianshu.com/p/d4a99b31ab37)
