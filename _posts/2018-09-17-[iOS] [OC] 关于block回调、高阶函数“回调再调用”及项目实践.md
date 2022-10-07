---
layout:       post
title:        "[iOS] [OC] 关于block回调、高阶函数“回调再调用”及项目实践"
author:       "beforeold"
header-style: text
catalog:      true
tags:
    - iOS
    - Objective-C
    - 函数 
---

#### 1/3 回调
使用```block```进行回调处理是十分便利的处理方式，在```UIKit```的设计中也屡见不鲜，例如：
- ```UIView```动画，动画执行后调用```completion```内的```block```代码。

```Objective-C
+ (void)animateWithDuration:(NSTimeInterval)duration
                 animations:(void (^)(void))animations
                 completion:(void (^ __nullable)(BOOL finished))completion;
```
- 模态展示一个页面，在展示结束后调用```completion```内的```block```代码。

```Objective-C
- (void)presentViewController:(UIViewController *)viewController
                     animated:(BOOL)flag
                   completion:(void (^)(void))completion;
```
- 用于实现纸质打印的控制器```UIPrintInteractionController```，其模态展示方式，同样是展示结束后调用```completion```的```block```代码。

```Objective-C
- (BOOL)presentAnimated:(BOOL)animated 
      completionHandler:(nullable UIPrintInteractionCompletionHandler)completion;
```

此外，在 ```WKWebView``` 中分析```JavaScript```代码时也有类似应用，实际开发中存在 ```completion```、```completionHandler``` 或者```callBack```等不同的命名方式，归根结底目的都是实现一个事件完成后的“回调作用”，与使用“委托模式”的 ```delegate + protocol``` 有异曲同工之妙，这也是很多二级页面控制器或者视图的回调流行使用一个 ```block``` 属性来做回调处理的原因。 

而类似 UIView 动画的 ```animations```的```block```动画参数，以及自动布局框架```Masonry``` 的设置约束的 ```make/update/remake``` 中 block 使用，以及实例初始化方法中的``` block```的应用，则用于更便利地囊括接口设计者的意图，比如开源网络框架```XMNetworking```的请求构造方法或者七牛云上传的管理类```QNUploadManager```的配置构造方法等，即可在 block 内便利地对请求参数进行配置，对外提供```API```时省去了类似 ```[[XMRequest alloc] init]```实例初始化这一步。
```Objective-C
[XMCenter sendRequest:^(XMRequest * _Nonnull request) {
    request.api = @"example/blabla";
    request.httpMethod = kXMHTTPMethodGET;
} ];
```
#### 2/3 回调再调用
上文提及委托模式也是典型的回调方式之一，在```iOS``` 应用程序的入口就采用了委托模式，即整个应用程序```UIApplication```单例的及其委托对象```AppDelegate```，```UIApplicationDelegate```协议中声明了诸多可选```optional```方法，将程序的运行情况相关事件/状态回调给委托者```AppDelegate```。与本文关联的是，自``` iOS 7 ```后系统升级了远程推送策略而新增了一系列 ```API```，其中就包括``` UIApplicationDelegate``` 协议中一个使用``` block``` 的协议方法如下（含典型实现）：
```Objecitve-C
- (void)application:(UIApplication *)application
        didReceiveRemoteNotification:(NSDictionary *)userInfo
        fetchCompletionHandler:(void (^)(UIBackgroundFetchResult))completionHandler {
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        // 当收到推送后异步加载一些数据
        // 然后告知回去数据的结果情况
        completionHandler(UIBackgroundFetchResultNewData);
    });
}
```
此协议方法是告知```AppDelegate```程序收到了远程推送，```AppDelegate``` 可以做一些获取数据的处理，并要求在获取数据完成后调用```completionHandler```告知 ```UIApplication``` 获取数据的结果情况，从而让 ```UIApplication``` 来估算电量和数据消耗情况，作为系统进行资源管理的一部分，要求```completionHander```必须尽快调用（30 s以内），这个场景就是**回调再调用**，其中：

1. 这是```UIApplication```委托的协议方法，而不是其实例方法，是 ```UIApplication```调用 ```AppDelegate``` 的方法，即**回调**。
2. 在回调的协议方法中，携带一个```block```类型的参数，将一段代码传递给 ```AppDelegate``` ，并要求 ```AppDelegate```完成业务逻辑后执行此```block```代码，以达到调用```UIApplication```的目的，即**回调再调用**。

> 这类将```block```作为参数或者返回值使用通常称为**高阶函数**。 

这种设计方式，有一种变换的实现方式：由```UIApplication```单独再提供一个 ```API ```让 ```AppDelgate```来主动调用，写一个伪代码方法如下：
```Objective-C
// UIApplication 类的伪代码
// 处理 delegate 后台获取数据后的结果
- (void)handleBackgroundFetchResult:(UIBackgroundFetchResult)result;
```
则，上述协议方法及其典型实现，可以替换为如下伪代码：
```Objective-C
// 注意移除了 回调的 block，改为直接调用伪代码 API
- (void)application:(UIApplication *)application
        didReceiveRemoteNotification:(NSDictionary *)userInfo {
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        // 异步获取一些数据
        // 然后告知获取数据的结果情况
        // completionHandler(UIBackgroundFetchResultNewData); // 改为 直接调用
    [application handleBackgroundFetchResult:UIBackgroundFetchResultNewData];
    });
}
```
对比两种方式，后者显然不如在回调 ```delegate``` 时直接带入需要执行的逻辑来得直观。这种“回调再调用”用法，后来在``` iOS 10``` 发布的系统重构的通知管理框架 ```UserNotification``` 中频繁使用，比如上述方法在 ```UNUserNotificaionCenter``` 的 ```UNUserNotificationCenterDelegate``` 中的声明。
```Objective-C
- (void)userNotificationCenter:(UNUserNotificationCenter *)center
didReceiveNotificationResponse:(UNNotificationResponse *)response
         withCompletionHandler:(void(^)())completionHandler;
```
总结可见，在如下场景中：```对象A```回调给```被回调者B```完成后，仍需要```被回调者B```去调用```A```并传递一些参数（或无参数）执行延续逻辑。采用类似回调一个 block 参数实现**回调再调用**是很不错的方案。

#### 3/3 项目实践
一个使用```block```做回调处理，并在回调中返回```block```参数用于延续逻辑在实际项目中的应用案例：

3个项目中，均需要通过网络接口请求的方式来获取客服联系电话后弹窗提示可拨打，三个网络接口各不相同。将该业务逻辑封装为一个 API ，方便多处业务入口的调用，具体是在通讯管理的单例``` [ContactHelper sharedInstance] ```:
```Objective-C
// 提示拨打客服电话，实例方法
- (void)callCustomerServerInVC:(UIViewController *)VC;
```
由于不同项目中网络接口不一致，且接口可能会变动，因此不在``` ContactHelper ```写网络请求逻辑，而是通过``` block ```的形式回调给具体项目进行实现，同时将弹窗的逻辑和样式封装在``` ContactHelper ```内部进行统一。

- 第一步，设置获取客服信息的逻辑，通过```ContactHelper```的声明为 ```fetcher```的```block```属性保存，当需要时进行调用

```
typedef void(^ContactCompletion)(NSDictionary *userInfo, NSString *errorMsg); // 

- (void)configCustomerPhoneFetcher:(void (^)(ContactCompletion completion, UIViewController *vc))fetcher;

/// 保存获取联系方式的逻辑
- (void)configCustomerPhoneFetcher:(void (^)(ContactCompletion))fetcher {
    _fetcher = [fetcher copy]; 
}

/// 项目中具体配置的调用示例
ContactHelper *helper = [ContactHelper sharedInstance];
[helper configCustomerPhoneFetcher:^(ContactCompletion completion, 
                                      UIViewController *vc) {
    // 通过网络请求异步获取电话号码
    NSString *tel = @"400xxxxxxx";
    /// 执行 回调再调用，实现电话号码拨叫
    completion(@{kContactPhoneKey:tel,nil);
}];
```
- 第二步，当业务方调用```ContactHelper```以弹窗拨打客服电话时，```ContactHelper```调用**第一步**配置好的获取方式 ```fetcher```属性。
- 第三步，利用```fetcher```获取到并再调用的信息进行弹窗拨号提示，因此```ContactHelper```内部实现调用客服电话后再弹窗提醒如下：

```Objective-C
//获取客服电话
- (void)callCustomerServiceInVC:(UIViewController *)controller{
    if (!_fetcher)  return;
     // 配置获取到客服电话后的操作
    ContactCompletion completion = ^(NSDictionary *dic, NSString *errorMsg){
    if (errorMsg) {
        // 提示获取号码出错
    } else {
        NSString *phone = dic[kContactPhoneKey];
       // 弹窗提示拨号
    };

  // 执行保存的回调，并将下一步的操作传递过去
   _fetcher(completion, controller);
}
```
综上，利用**回调再调用**这个思路，可以将3个项目的不同接口的请求客服电话的请求隔离在3个项目中设置，而弹窗提示的逻辑则在 ```ContactHelper```中统一处理，而在其他的一些需要外部获取数据后再返回到调用者延续执行的情况都可以使用该方案。

#### 参考文献
iOS程序犭袁:  [有一种 Block 叫 Callback，有一种 Callback 叫 CompletionHandler](http://www.jianshu.com/p/061c393d6c9d)
其中，在第三方云服务 ```LeanCloud``` 的一些```SDK```中有类似的高阶函数应用。
