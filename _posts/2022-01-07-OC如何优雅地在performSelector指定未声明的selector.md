---
layout:       post
title:        "OC如何优雅地在performSelector指定未声明的selector"
author:       "beforeold"
header-style: text
catalog:      true
tags:
    - Objective-C
    - Runtime
    - API Design
---

# 背景
由于 OC 的动态特性，可以通过 performSelector: 实现部分特殊情况下未声明 selector（一般是无法 import 其 API header 文件），比如：
- 调用私有 API
- 在低版本 SDK 编译环境下调用高版本 SDK 的 API
- 处于解耦的目的不便直接依赖导入（PS：常规情况，这样是不推荐的，可以通过不少解耦方案进行实现。）

由于 API 没有事先声明，通过 performSelector: 调用，一般会有两个警告，分别是 selector 的声明以及 selector 的调用，例如：

![构造 selector 调用的警告](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/af1d287c864d4533ad64ff46d390e0b0~tplv-k3u1fbpfcp-watermark.image?)
通过一些编译器宏预处理可以忽略这两个警告，只是每次用都要搜索查找一下，这里分享其他相对优雅的方案，供探讨。

> **注意1**： 编译器警告是为了降低编程风险，应尽量避免 runtime 的滥用

> **注意2**：所有 runtime 类型的调用方案使用，都需要优先确保调用安全，做好前置的可用判断

# 消除声明 selector 的警告
对于 Undeclared selector 的警告，可直接使用 NSSelectorFromString() 函数构造 selector，例如：
```Objective-C
SEL sel = NSSelectorFromString(@"playInstanceWithObject:");
```


# 方案 1：转换为方法的函数实现的调用
执行 selector 的底层实现也就是 objc_msgSend 的调用，基于这一思路，可以不使用 performSelector:， 每一个 selector 通过在 runtime 查询可以找到其实现对应的函数，即 IMP，在 NSObject 中提供了获取 IMP，以实例方法为例：
```Objective-C
// Locates and returns the address of the receiver’s implementation of a method
// so it can be called as a function.
- (IMP)methodForSelector:(SEL)aSelector;
```

利用该方法，获取到方法的实现函数，进而进行函数调用，例如：
```Objective-C
if ([ins respondsToSelector:sel]) {
    IMP imp = [ins methodForSelector:sel];
    BOOL(*func)(id, SEL, NSString *) = (void *)imp;
    func(ins, sel, @"arg1");
}
```
注意需要将实现强转为目标 selector 所对应的参数列表函数指针类型，参数列表按序号分别是：
- 第 0 个：接收消息的对象本身(self)
- 第 1 个：selector 当前 selector(_cmd)
- 其他参数，如果方法有其他参数可以按需要声明，例子中是有一个 NSString 参数，**此处支持 NSInteger 等基本类型**

# 方案 2：调用 NSInvocation 封装调用
 Founation 框架中对于 performSelector: 是有现成的封装的，即 NSInovcation，
 ```Objective-C
if ([ins respondsToSelector:sel]) {
    NSMethodSignature *sig = [ins methodSignatureForSelector:sel];
    NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:sig];
    invocation.target = ins;
    invocation.selector = sel;
    [invocation setArgument:@"arg1" atIndex:2];
    [invocation invoke];
    BOOL ret;
    [invocation getReturnValue:&ret];
}
```
注意到 invocation 使用的特点：
- 将 IMP 封装为 NSMethodSignature 构造 NSInvocation 对象
- 分别设置 target, selector
- 参数序号 index 从 2 开始，同上 0 是 target，1 是 selector
- 参数的传入需要传外部参数的指针
- 返回值的获取需要传入接受返回值的指针
PS：该方案在 [Casa 大佬](http://casatwy.com/) 的解耦框架 [CTMediator](https://github.com/casatwy/CTMediator/blob/master/CTMediator/CTMediator/CTMediator.m) 中有很好的实践。


# 方案 3：针对属性赋值的情况
这种情况下，调用的其实是属性的 setter 方法，以 UNNotification 在 iOS 15 引入的 interruptionLevel 属性为例：
```Objective-C
@property (NS_NONATOMIC_IOSONLY, readonly, assign) UNNotificationInterruptionLevel interruptionLevel API_AVAILABLE(macos(12.0), ios(15.0), watchos(8.0), tvos(15.0));
```
当前业务如暂时无法使用高版本 SDK 编译，即使使用 @available(iOS 15.0,*）判断，也无法直接调用该 API，对于这类属性类的方法调用，可以尝试采用 **KVC** 的方式完后完成。
```Objective-C
[content setValue:1 forKey:@"interruptionLevel"];
```
# 小结
上述方案可见 OC 的 runtime 特性是相互关联的，不同的是 Cocoa 提供的 API 层级差异性，从函数，到 performSelector: 到 NSInvocation，在保留动态特性时提供了易用程度不一的 API，可在其他有动态能力需求的技术方案设计中进行应用。