#### 1、并不总是需要移除观察者
自 **```iOS 9```**  开始（见 [release notes](https://developer.apple.com/library/content/releasenotes/Foundation/RN-FoundationOlderNotes/index.html#10_11NotificationCenter) ），```Foundation``` 调整了 ```NSNotificationCenter``` 对观察者的引用方式（ ```zeroing weak reference```），不再给已释放的观察者发送通知，因此以往在 ```dealloc``` 时移除观察者的做法可以省去。

如果是需要适配 ```iOS 8```，那么 ```UIViewController```及其子类可以省去移除通知的过程（亲测有效），而其他对象则需要在 ```dealloc``` 前移除观察者。

> 感谢 **Ace** 同学第一时间的测试发现

#### 2、控制器添加和移除观察者的良好实践
控制器对象对于通知的监听通常是在生命周期的 ```viewDidLoad``` 方法处理，也就是说，在 ```viewDidLoad``` 之前，还未添加观察者，对应地在在移除通知通知时可以做是否加载了视图的判断如下：

```Objective-C
- (void)dealloc {
    if (self.isViewLoaded) {
        [[NSNotificationCenter defaultCenter] removeObserver:self];
    }
}
```
这一点 ```isViewLoaded``` 的判断，对于 NSNotification 的监听来说不是必要的，因为在未监听通知的情况下，调用 ```removeObserver:``` 方法是仍旧是安全的，而 ```KVO ( key-value observing```，则不然。因为 ```KVO``` 在未监听的情况下移除观察者是不安全的，所以如果是在 ```viewDidLoad``` 监听```KVO``` ，则 ```KVO``` 的移除就需要执行判断：

```Objective-C
- (void)dealloc {
    if (self.isViewLoaded) {
        [self removeObserver:someObj forKeyPath:@"someKeyPath"];
    }
}

```
此外，很多时候控制器的视图还未加载，也需要监听特定的通知，此时通知的监听适合在构造方法 ```initWithNibName:bundle``` 方法中监听，此构造方法在代码或者 ```Interface Builder``` 构建实例时都会调用：

```Objective-C
- (instancetype)initWithNibName:(NSString *)nibNameOrNil bundle:(NSBundle *)nibBundleOrNil {
    self = [super initWithNibName:nibNameOrNil bundle:nibBundleOrNil];
    if (self) {
        [[NSNotificationCenter defaultCenter] addObserver:self
                                                 selector:@selector(onNotification:)
                                                     name:@"kNotificationName"
                                                   object:nil];
    }
    
    return self;
}
```

#### 3、系统 ```NSNotificationCenter``` 是支持 ```block``` 手法的

自 ```iOS 4``` 开始通知中心即支持 ```block``` 回调，其 ```API``` 如下：

```Objective-C
- (id <NSObject>)addObserverForName:(nullable NSNotificationName)name
                             object:(nullable id)obj
                              queue:(nullable NSOperationQueue *)queue
                         usingBlock:(void (^)(NSNotification *note))block
                                    NS_AVAILABLE(10_6, 4_0);
```
回调可以指定操作队列，并返回一个观察者对象。调用示例：

```Objective-C
- (void)observeUsingBlock {
    NSNotificationCenter *center = [NSNotificationCenter defaultCenter];
    observee = [center addObserverForName:@"kNotificationName"
                                   object:nil
                                    queue:[NSOperationQueue mainQueue]
                               usingBlock:^(NSNotification * _Nonnull note) {
                                   NSLog(@"got the note %@", note);
                               }];
}

- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:observee];
}

```

其中，有几点值得注意：
- 方法返回一个 ```id<NSObject>``` 监听者对象，其实是系统的私有类的实例，因为没必要暴露其具体类型和接口，所以用一个 ```id<NSObject>``` 对象指明用途，从中可见协议的又一个应用场景。
- 这个返回值对象是充当了原来的 ```target-action``` 的封装实现，在其内部触发了 ```action``` 后调用起初传入的 ```block``` 参数。
- 返回的观察者和 ```block``` 都会被通知中心所持有，因此使用者有义务在必要的时候调用 ```removeObserver:``` 方法，将此监听移除，否则监听者和 ```block```及其所捕获的变量都不会释放，从而导致内存泄露。

#### 4、在必要时提前拦截通知的发送
通知的使用在跨层和面向多个对象通信时十分便利，也因此而导致难以管理的问题颇受诟病，发送通知时可能需要统一做一些工作，此时对通知进行拦截是必要的。```NSNotificationCenter``` 是 ```CFNotificationCenter``` 的封装，有使用类似 ```NSArray``` 的类簇设计，并采用了单例模式返回共享实例 ```defaultCenter```。通过直接继承的方式进行发送通知的拦截是不可行的，因为获得的是始终是静态的单例对象，从 ```Telegram``` 公司的[开源项目工程](https://github.com/peter-iakovlev/Telegram)中可以看到：通过借鉴 ```KVO``` 的实现原理，将单例对象的类修改为特定的子类，从而实现通知的拦截。

第一步，修改通知中心单例的类：

```Objective-C
@interface GSNoteCenter : NSNotificationCenter

@end


/// 修改单例的类为一个子类的类型
void hack() {
    id center = [NSNotificationCenter defaultCenter];
    object_setClass(center, GSNoteCenter.class);
}
```

第二步，拦截通知的发送事件：
利用继承多态特性，在发送通知的前后进行拦截：

```Objective-C
@implementation GSNoteCenter

- (void)postNotificationName:(NSNotificationName)aName
                      object:(id)anObject
                    userInfo:(NSDictionary *)aUserInfo
{
    // do something before post
    [super postNotificationName:aName
                         object:anObject
                       userInfo:aUserInfo];
    // do something after post
}

@end

```

PS：拦截之后可以发现系统发送通知的数量和频率真高，从这个侧面看发送通知的性能问题不用太过顾忌。

#### 5、自定义不需要移除监听的 block 的通知中心（附源代码）
既不愿意手动移动通知，又想使用 ```block``` 实现通知监听，那么必要的封装是必须的。比如， [ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa) 中的实现如下：

```Objective-C
@implementation NSNotificationCenter (RACSupport)

- (RACSignal *)rac_addObserverForName:(NSString *)notificationName object:(id)object {
	@unsafeify(object);
	return [[RACSignal createSignal:^(id<RACSubscriber> subscriber) {
		@strongify(object);
		id observer = [self addObserverForName:notificationName
                                        object:object
                                         queue:nil
                                    usingBlock:^(NSNotification *note) {
			[subscriber sendNext:note];
		}];

		return [RACDisposable disposableWithBlock:^{
			[self removeObserver:observer];
		}];
	}] setNameWithFormat:@""];
}
@end
```
将通知作为一个信号源，直接订阅 ```next``` 收听结果即可，十分优雅地解决了 ```block``` 的使用以及通知的移除。

在不引入响应式框架的情况下，通过自定义通知名称与观察者的关系的方式，可以满足要求。基本思路是：
- 以一个观察者对应一个 block，存入一个类似字典的集合，但需要实现当观察者释放时同时也释放 block，这里考虑采用支持弱引用的集合，比如 ```NSMapTable```。
- 观察者对通知名进行监听，因此一个通知名对应了一个集合，当触发一个通知名时，通知集合内的存在的所有观察者。
- 通知中心统一持有所有通知名及其关联关系。

由此实现的初步封装完成放在 [GitHub](https://github.com/beforeold/GSNoticeManager)，通知的注册如下：

```Objective-C
- (void)registerBlock:(GSNoticeBlock)block service:(NSString *)service forObserver:(id)observer {
    GSServiceMap *mapModel = [self mapForService:service];
    [mapModel.map setObject:block forKey:observer];
}

```

通知的触发如下：
```Objective-C
- (void)triggerService:(NSString *)service userInfo:(id)userInfo {
    GSServiceMap *mapModel = [self mapForService:service];
    NSString *key = nil;
    NSEnumerator *enumerator = [mapModel.map keyEnumerator];
    while (key = [enumerator nextObject]) {
        GSNoticeBlock block = [mapModel.map objectForKey:key];
        !block ?: block(userInfo);
    }
}
```

如果需要提前移除监听，操作如下：
```Objective-C
- (void)unregisterService:(NSString *)service forObserver:(id)observer {
    GSServiceMap *mapModel = [self mapForService:service];
    [mapModel.map removeObjectForKey:observer];
}
```
> 感谢 Mark 同学说通知中心不安全，才尝试自定义一个安全的通知中心。

#### 源代码 

[GitHub](https://github.com/beforeold/GSNoticeManager)

#### 小结
通知中心，作为观察者模式的运用，通过 ```block``` 的运用可以有更灵活的表现，比如前文分享的 **Uber** 用于解决通知中心难以管理的解决方案 [以 Uber-signals 一窥响应式](http://www.jianshu.com/p/0a58b646d0c8)。

再到 ```ReactiveCocoa```、```RxSwift``` 函数响应式的思想的进一步抽象，编程的思维从命令式地调用一个方法/函数，转换为因为某个通知/信号而触发了下一步的操作，值得去进一步探索。


#### 参考资料

[Unregistering NSNotificationCenter Observers in iOS 9]([https://useyourloaf.com/blog/unregistering-nsnotificationcenter-observers-in-ios-9/](https://useyourloaf.com/blog/unregistering-nsnotificationcenter-observers-in-ios-9/))
[Telegram 源代码](https://github.com/peter-iakovlev/Telegram)
[Microsoft/WinObjc Reimplementate NSNotificationCenter](https://github.com/Microsoft/WinObjC/commit/a2d4b3fa279d9ea599acf3f1d108921128a3bd1f)
[Microsolf/WinObjc 真是一座金山啊](https://github.com/Microsoft/WinObjC)
