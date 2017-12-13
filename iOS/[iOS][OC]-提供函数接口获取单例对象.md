#### 背景
单例模式在 ```iOS``` 开发中的运用十分常见，比如系统的 API 中就有 ```UIApplication```、```NSNotificationCenter```等，一般提供单例的 API 是 OC 的类方法（```class method```）调用方式，典型的范式如下：
```Objective-C

[UIApplication sharedApplication]
[NSNotificationCenter defaultCenter]
[UNUserNotificationCenter currentNotificationCenter]

```
在实际的调用中，是这样的
```Objective-C
[[NSNotificationCenter defaultCenter] addObserver:self
                                         selector:@selector(onRefreshBiddingList:)
                                         name:kGSRefreshBiddingGoods
                                        object:nil];
```
因为 OC 的命名习惯，这样的调用显得冗长而无美感，对于有强迫症的同学来说有些不能接受。
当然，可以尝试变换一下：
```Objective-C
NSNotificationCenter *center = [NSNotificationCenter defaultCenter];
[center addObserver:self
            selector:@selector(onRefreshBiddingList:)
             name:kGSRefreshBiddingGoods
                     object:nil];
```
或者干脆定义一个宏，不过宏定义在接口头文件一般不容易被发现。
```Objective-C
#define NSNotifcaitionCenterInstance [NSNotificationCenter defaultCenter]
```
### 用函数提供接口获取单例
使用静态函数提供接口是更简洁的方法，举例如下：
````Objective-C
// 声明
FOUNDATION_EXTERN GSPushManager *GSPushManagerInstance();
/// 实现
GSPushManager *GSPushManagerInstance() {
    static id instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[GSPushManager alloc] init];
    });
    
    return instance;
}
```
调用接口时，使用函数接口如下：
```Objective-C
[GSPushManagerInstance() doSomething];
```
也可以考虑兼容，提供一个类方法 API 供调用，实现如下：
```Objective-C
// 声明
+ (instancetype)sharedInstance;
// 实现
+ (instancetype)sharedInstance {
    return GSPushManagerInstance();
}
```

### 延伸
Swift 的单例用法，是将单例设置为类的一个属性
```Swift
public class Singleton {
    //通过关键字static来保存实例引用
    private static let instance = Singleton()
    
    //私有化构造方法
    private init() { }
    
    //提供静态访问方法
    public static var shared: Singleton {
        return self.instance
    }
}
```
实际上， ```OC``` 为了兼容 ```Swift```，为此定义了类属性的概念，即在属性声明中加入 ```class```关键字，比如 ```UIApplication```
```Objective-C
/// 声明
@property(class, nonatomic, readonly) UIApplication *sharedApplication
//调用
[UIApplication.sharedApplication openURL:....];
```
