#### 一、 一行代码执行 block 的安全调用
基于 判断对象是否为 nil 的处理，```obj ?: someDefault``` 的思路，调整 block 的调用
```Objective-C
void(^block)(void) = ...;
!block ?: block(); // 安全调用
```
PS: 如果是 Swift 的可选类型闭包，则可利用可选链处理

```Swift
var closure: (() -> Void)? = ...
closure?()  // 可选链调用
```

#### 二、利用系统现成的 typedef 的相关 block 
比如 GCD 中关于无参数无返回值的 typedef
```Objective-C
typedef void (^dispatch_block_t)(void);
```
这样就可以很愉快地利用 dispatch_block_t 来声明无参数和无返回值的 block 了，而在 UIKit 中有很多 block 参数都是无参数无返回值类型 block，在实际业务中也可以利用，比如 上一篇文章中末尾的应用 [注意自定义 AlertController 的回调时机](http://www.jianshu.com/p/0cecb03afbc6)。


Swift 中的此用法暂时没有找到很合适的，可以从一个协议中看到类似的定义
```
DispatchSourceProtocol.DispatchSourceHandler
extension DispatchSourceProtocol {

    public typealias DispatchSourceHandler = @convention(block) () -> Swift.Void
....
}1
```
更多的，继续挖掘学习。

#### 三、使用 block 作为函数/方法多个返回值的方案
在 OC 中一个方法返回多个参数有一些方案，比如使用指针的指针（类似返回 NSError 的用法），或者声明字典、模型或者自定义一个 元组Tuple 类等。再者，就是使用 block 了，通过传递 block 将返回值取出的方式，举例如下：
```Objective-C

typedef NSInteger Int;

- (void)doSomeCalculate {
    Int a = 1;
    Int b = 2;
    __block Int sumR =  0;
    __block Int minR = 0;
    __block Int maxR = 0;
    
    [self calculateA:a b:b result:^(Int sum, Int min, Int max) {
        sumR = sum;
        minR = min;
        maxR = maxR;
    }];
    
    NSLog(@"sum %zd min %zd max %zd", sumR, minR, maxR);
}

- (void)calculateA:(NSInteger)a
                 b:(NSInteger)b
            result:(void(^)(Int sum, Int min, Int max))resultBlock {
    if (!resultBlock) return;
    
    resultBlock(a + b, MIN(a, b), MAX(a, b));
    
    return [self doNothingButPrint];
}

```

存在使用了类的成员变量而导致循环引用的情况，需要使用 weak- strong-dance，并在访问实例变量使用结构体访问成员的操作符 ```->```
```Objective-C
@interface SomeClass : NSObject

@property (nonatomic, copy) dispatch_block_t block;

@end

typedef NSInteger Int;

@implementation SomeClass
{
    Int _sum;
}


- (void)changeIvar {
    __weak typeof(self) weakSelf = self;
    self.block = ^{
        __strong typeof(SomeClass *) strongSelf = weakSelf;
        if (!strongSelf) return;
        strongSelf->_sum = 1024;
    };
}

@end
```

#### 四、使用返回 对象本身的 block 实现链式语法
经典的例子是 OC 的自动布局第三方框架 [Masonry](https://github.com/SnapKit/Masonry) 的使用，
```Objective-C
[view1 mas_makeConstraints:^(MASConstraintMaker *make) {
    make.top.equalTo(superview.mas_top).with.offset(padding.top); //with is an optional semantic filler
    make.left.equalTo(superview.mas_left).with.offset(padding.left);
    make.bottom.equalTo(superview.mas_bottom).with.offset(-padding.bottom);
    make.right.equalTo(superview.mas_right).with.offset(-padding.right);
}];
```
其内部实现的主要思路是两个
- 声明一个只读 block 属性供外部调用，此 block 属性返回实例本身
- 声明一个属性，调用后返回一个实例（可以是本身），并可以执行额外操作

```Objective-C
- (MASConstraint * (^)(CGFloat))offset {
    return ^id(CGFloat offset){
        self.offset = offset;
        return self;
    };
}

```
不难看出，在一定程度上说，方法都可以转换成 block 的形式调用，而 block 连续返回实例本身，从而可以实现链式语法调用，提高代码的可读性，尝试实现一个链式加法调用如下：

```Objective-C
@interface Dot : NSObject

@property (nonatomic, readonly) Dot *then;
@property (nonatomic, readonly) Dot *(^add)(NSInteger a);

- (NSInteger)getCurrent;

@end

@interface Dot ()
{
    NSInteger _current;
}

@end

@implementation Dot

- (NSInteger)getCurrent {
    return _current;
}

- (Dot *)then { return self; };

- (Dot *(^)(NSInteger))add {
    return ^Dot *(NSInteger a){
        _current += a;
        return self;
    };
}

- (void)dealloc {
    NSLog(@"dot dealloc %@", self);
}

@end

/// 调用
        Dot *dot = [[Dot alloc] init];
        dot.add(1).then.add(2).then.add(3);
        NSLog(@"now -> %zd", [dot getCurrent]); // 打印出 6
        dot = nil;

```
使用链式语法有一个很大的前提是调用 block 前该 block 不能为空，所以使用时注意挑选调用者不为空的场景，参考 ```Masonry```。
再者，链式语法在构造构造时有很好的优势，比如纯代码构造 UI 控件，推荐阅读以下 [LinkBlock](https://github.com/qddnovo/LinkBlock)， 从而这样写一个 ```Label```
```Objective-C
UILabelNew
.labText(@"UILable").labNumberOfLines(0).labAlignment(NSTextAlignmentCenter)
.viewSetFrame(20,200,150,80)
.viewBGColor(@"#CCCCCC".strToUIColorFromHex())
.viewAddToView(self.view);
```

> 此外，block 在很多开源项目中都有很好的实践，推荐:
>- [ReactiveCocoa的 OC 版本](https://github.com/ReactiveCocoa/ReactiveObjC) 
>- [BlocksKit](https://github.com/BlocksKit/BlocksKit)

>诚邀你来讨论我的专题， QQ群 287698622
