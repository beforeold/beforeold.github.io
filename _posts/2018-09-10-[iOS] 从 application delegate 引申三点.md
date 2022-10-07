---
layout:       post
title:        "[iOS] 从 application delegate 引申三点"
author:       "beforeold"
header-style: text
catalog:      true
tags:
    - iOS
    - 内存管理
    - 设计模式
---

#### 1、声明的 ```delegate``` 属性不总是 ```weak``` 策略
委托 ```delegation``` 是 ```iOS``` 开发的常用设计模式，为了避免对象与其代理之间的因为相互 retain 导致循环引用的发生，```delegate``` 属性在如今的 ```ARC``` 时代通常声明为 ```weak``` 策略，然而在早期的手动管理内存的时代，还未引入 ```strong/weak``` 关键字，使用 ```assign``` 策略保证 ```delegate``` 的引用计数不增加， 在 ```Swift``` 中是 ```unowned(unsafe)```， ```UIApplication``` 的 ```delegate``` 声明如下：
```Objective-C
// OC
@property(nullable, nonatomic, assign) id<UIApplicationDelegate> delegate;
```


```Swift
// Swift
unowned(unsafe) open var delegate: UIApplicationDelegate?
```

```assign``` 和 ```weak``` 都只复制一份对象的指针，而不增加其引用计数，区别是：```weak``` 的指针在对象释放时会被系统自动设为 ```nil```，而 ```assign``` 却仍然保存了 ```delegate``` 的旧内存地址，潜在的风险就是：如果 ```delegate``` 已销毁，而对象再通过协议向 ```delegate``` 发送消息调用，则会导致野指针异常 ```exc_bad_access```，这在使用 ```CLLocationMananger``` 的 ```delegate``` 时**尤其需要注意**。如果使用了 ```delegate``` 为 ```assign``` 策略，则需要效仿系统对 ```weak``` 的处理，在 ```delegate``` 对象释放时将 ```delegate``` 手动设置为 ```nil```。

```Objective-C
@implementation XViewController
- (void)viewDidLoad {
        [super viewDidLoad];
        self.locationManager = [[CLLocationManager alloc] init];
      self.locationManager.delegate = self;
      [self.locationManager startUpdatingLocation];
}

/// 必须手动将其 delegate 设为 nil
- (void)dealloc {
    [self.locationManager stopUpdatingLocation];
    self.locationManager.delegate = nil;
}

@end
```
除了 ```assgin``` 的情况，还有一些情况下 ```delegate``` 是被对象强引用 ```retain``` 的，比如 ```NSURLSession```，```delegate``` 将被 ```retain``` 到 ```session``` 对象失效为止。

```Objective-C
/* .....
 * If you do specify a delegate, the delegate will be retained until after
 * the delegate has been sent the URLSession:didBecomeInvalidWithError: message.
 */
+ (NSURLSession *)sessionWithConfiguration:(NSURLSessionConfiguration *)config
                     delegate:(nullable id <NSURLSessionDelegate>)delegate
                   delegateQueue:(nullable NSOperationQueue *)queue;
```
对于这种情况，处理方式，循环引用是肯定存在的，解决的方式是通过移除引用的方式来手动打破，因此  ```NSURLSession``` 提供了 ```session``` 失效的两个方法：

```Objective-C
- (void)finishTasksAndInvalidate;
- (void)invalidateAndCancel;
```
作为 ```NSURLSession``` 的第三方封装 ```AFNetworking```，其 ```AFURLSessionManager``` 对应地提供了失效方法：
```Objective-C
/**
 Invalidates the managed session, optionally canceling pending tasks.
 @param cancelPendingTasks Whether or not to cancel pending tasks.
 */
- (void)invalidateSessionCancelingTasks:(BOOL)cancelPendingTasks;
```
此外，```CAAnimation``` 的 ```delegate``` 也是 ```strong``` 引用，如果因为业务需要发生了循环引用，需要在合适的时机提前手动打破。
```Objective-C
@interface CAAnimation
//...
/* The delegate of the animation. This object is retained for the
 * lifetime of the animation object. Defaults to nil. See below for the
 * supported delegate methods. */

@property(nullable, strong) id <CAAnimationDelegate> delegate;
```
总之，使用 ```delegate``` 时需要留意其声明方式，因地制宜地处理。

#### 2、既然是 ```assign```, 那么 ```AppDelegate``` 为什么不会销毁
上文讨论 ```UIApplication``` 对 ```delegate```，也就是 ```AppDelegate``` 类的实例，其声明为 ```assign``` 策略，```AppDelegate``` 实例没有其他对象引用，在应用的整个声明周期中是一直存在的，原因是什么？

在 ```stackoverflow``` 的一个回答 [Why system call UIApplicationDelegate's dealloc method?](https://stackoverflow.com/questions/11685025/why-system-call-uiapplicationdelegates-dealloc-method) 中可以了解到一些情况，大意是：

- 从 ```main.m``` 中初始化了第一个 ```AppDelegate``` 实例，被系统内部隐式地 ```retain```
- 直到下一次 ```application``` 被赋值一个新的 ```delegate``` 时系统才将第一个 ```AppDelegate``` 实例释放
- 对于新创建的 ```application``` 的 ```delegate``` 对象，由创建者负责保证其不会立即销毁，举例如下：

```Objective-C
// 声明为静态变量，长期持有
static AppDelegate *retainDelegate = nil;

/// 切换 application 的 delegate 对象
- (IBAction)buttonClickToChangeAppDelegate:(id)sender {
    AppDelegate *delegate = [[AppDelegate alloc] init];
    delegate.window.rootViewController = [[ViewController alloc] init];
    [delegate.window makeKeyAndVisible];
    
    retainDelegate = delegate;
    [UIApplication sharedApplication].delegate = retainDelegate;
}

```
```application``` 的 ```delegate``` 可以在必要时切换（通常不这样做），```UIApplication``` 单例的类型同样是支持定制的，这个从 ```main.m``` 的启动函数可以看出：

```Objective-C

// If nil is specified for principalClassName, 
// the value for NSPrincipalClass from the Info.plist is used. If there is no
// NSPrincipalClass key specified, the UIApplication class is used. 
// The delegate class will be instantiated using init.
UIKIT_EXTERN int UIApplicationMain(int argc, 
                              char *argv[], 
                              NSString * __nullable principalClassName, 
                            NSString * __nullable delegateClassName);


```
通过 ```UIApplicationMain()``` 函数传参或者在 ```info.plist``` 中注册特定的 ```key``` 值，自定义应用的 ```Application``` 和 ```AppDelegate``` 的类是可行的。
```Objective-C
@interface XAppDelegate : UIResponder
@property (nonatomic, strong) UIWindow *window;
@end

@interface XApplication : UIApplication
@end

int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc,
                                 argv,
                                 NSStringFromClass([XApplication class]),
                                 NSStringFromClass([XAppDelegate class]));
    }
}
```
又或者，干脆使用 ```runtime``` 在必要时将 ```application``` 对象替换为其子类。

```Objective-C
/// runtime, isa-swizzling，置换应用单例的类为特定子类
object_setClass([UIApplication sharedApplication], [XApplication class]);
```

#### 3、使用 ```Notification``` 和 ```category``` 避免 ```AppDelegate``` 的臃肿
因为 ```AppDelegate``` 是应用的 ```delegate```，其充当了应用内多个事件的监听者，包括应用启动、收到推送、打开 ```URL```、出入前后台、收到通知、蓝牙和定位等等。随着项目的迭代，```AppDelegate``` 将越发臃肿，因为 ```UIApplication``` 除了 ```delegate``` 外，还同时有发送很多通知 ```NSNotification```，因而可以从两个方面去解决：

- 尽可能地通过 ```UIApplication``` 通知监听来处理事件，比如应用启动时会发送
 ```UIApplicationDidFinishLaunchingNotification``` 通知
- 没有通知但是特定业务的 ```UIApplicaiondDelegate``` 协议方法，可以按根据不同的业务类型，比如通知、```openURL``` 分离到不同的 ```AppDelegate``` 的 ```category``` 中

进一步地，对于应用启动时，就需要监听的通知，合适时机是在某个特定类的 ```load``` 方法中开始。针对性地，可以为这种 ```Launch``` 监听的情况进行封装，称为 ```AppLaunchLoader```：
- 在 ```AppLaunchLoader``` 的 ```load``` 方法中监听应用启动的通知
- 在  ```AppLaunchLoader``` 的 ```category``` 的 ```load``` 方法中注册启动时需要执行的任务 ```block```
- 当监听到应用启动通知时执行注册的所有 ```block```，完成启动事件与 ```AppDelegate``` 的分离。

```Objective-C
typedef void(^GSLaunchWorker)(NSDictionary *launchOptions);

@interface GSLaunchLoader : NSObject

/// 注册启动时需要进行的配置工作
+ (void)registerWorker:(GSLaunchWorker)worker;
@end

@implementation GSLaunchLoader
+ (void)load {
    NSNotificationCenter *c = [NSNotificationCenter defaultCenter];
    [c addObserver:self selector:@selector(appDidLaunch:) name:UIApplicationDidFinishLaunchingNotification object:nil];
}

+ (void)appDidLaunch:(NSNotification *)notification {
    [self handleLaunchWorkersWithOptions:notification.userInfo];
}

#pragma mark - Launch workers
static NSMutableArray <GSLaunchWorker> *_launchWorkers = nil;
+ (void)registerWorker:(GSLaunchWorker)worker { [[self launchWorkers] addObject:worker]; }
+ (void)handleLaunchWorkersWithOptions:(NSDictionary *)options {
    for (GSLaunchWorker worker in [[self class] launchWorkers]) {
        worker(options);
    }
    
    [self cleanUp];
}

+ (void)cleanUp {
    _launchWorkers = nil;
    NSNotificationCenter *c = [NSNotificationCenter defaultCenter];
    [c removeObserver:self name:UIApplicationDidFinishLaunchingNotification object:nil];
}

+ (NSMutableArray *)launchWorkers {
    if (!_launchWorkers) {
        static dispatch_once_t onceToken;
        dispatch_once(&onceToken, ^{
            _launchWorkers = [NSMutableArray array];
        });
    }
    return _launchWorkers;
}

@end
```

#### 源代码

[点我去 GitHub 获取源代码，✨鼓励](https://github.com/beforeold/GSLaunchLoader)

> 推荐阅读 苏合的 [《关于AppDelegate瘦身的多种解决方案》](http://www.jianshu.com/p/a926fd605b7a)
