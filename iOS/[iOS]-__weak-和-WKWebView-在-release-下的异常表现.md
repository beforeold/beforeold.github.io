#### __weak
因为在业务开发中的需求需要几个局部变量，考虑到：
- 局部变量需要先声明在一个 block 变量内使用后再构造实例赋值，因此变量声明为 __block
- 局部变量构造完成实例后会持有此 block，为避免循环引用，将局部变量声明为 __weak
- 变量最终被装入数组被控制器强引用。

先定义对象 XRow 的基本信息，内部有一个字典，用于保存 block 。
```Objective-C
static NSString *const kBlockKey = @"kBlockKey";
@interface XRow : NSObject
@property (nonatomic, strong) NSMutableDictionary *value;
@end

@implementation XRow
- (NSMutableDictionary *)value {
    if (!_value) {
        _value = [NSMutableDictionary dictionaryWithCapacity:1];
    }
    
    return _value;
}

- (void)dealloc {
    NSLog(@"dealloc %@", self);
}

@end
```

接着定义测试的逻辑：
- 创建对象
- 传入 block 
- 保存到数组

示例代码分三种情况测试如下：
- 直接赋值 __weak，查看赋值结果
- 声明 __block __weak 装入数组
- 声明 __block 再进行 __weak 引用，装入数组
分别进行 debug 和 release 的 build mode 运行：
```Objective-C
@interface ViewController ()
@property (nonatomic, strong) NSMutableArray *array;
@end

@implementation ViewController
- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self test__block__weak_1];
    NSLog(@"tested");
}

- (void)testAssignWeak {
    __weak id obj = nil;
    __weak id obj2 = nil;
    __weak id obj3 = nil;
    obj = [self makeRow];
    obj2 = [self makeRow];
    obj3 = [[XRow alloc] init]; // 直接收到编译器警告
    NSLog(@"obj %@", obj);
    NSLog(@"obj2 %@", obj2);
}

- (void)test__block__weak_1 {
    __block __weak XRow *row1 = nil;
    __block __weak XRow *row2 = nil;
    
    void(^block)(void) = ^{
        // 处理 row1 row2 的一些逻辑
        NSLog(@"row1 %@ row2 %@", row1, row2);
    };
    
    row1 = [self makeRow];
    row1.value[kBlockKey] = block;
    
    row2 = [self makeRow];
    row2.value[kBlockKey] = block;
    
    [self.array addObject:row1];
    [self.array addObject:row2];
    NSLog(@"array -> %@", self.array);
}

- (void)test__block__weak_2 {
    __block XRow *row1 = nil;
    __block XRow *row2 = nil;
    
    // 这样 处理 在 block 内部是 nil，没有 __block 效果
    __weak typeof(XRow *) weakRow1 = row1;
    __weak typeof(XRow *) weakRow2 = row2;
    void(^block)(void) = ^{
        // 处理 row1 row2 的一些逻辑
        NSLog(@"row1 %@ row2 %@", weakRow1, weakRow2);
    };
    
    row1 = [self makeRow];
    row1.value[kBlockKey] = block;
    
    row2 = [self makeRow];
    row2.value[kBlockKey] = block;
    
    [self.array addObject:row1];
    [self.array addObject:row2];
    NSLog(@"array -> %@", self.array);
}

- (XRow *)makeRow {
    return [[XRow alloc] init];
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    NSLog(@"touch to excute");
    for (XRow *some in self.array) {
        void(^block)(void) = some.value[kBlockKey];
        if (block) block();
    }
    [self.array removeAllObjects];
    NSLog(@"removed");
}

- (NSMutableArray *)array {
    if (!_array) {
        _array = [NSMutableArray arrayWithCapacity:2];
    }
    
    return _array;
}

@end
```

可以发现：
- 直接 alloc] init] 的对象在复制后立刻销毁，并会收到编译器警告
- 在 release mode 下 __weak 的表现和 debug 有所差异，，通过 makeRow 返回的对象，在 debug mode 下并不会立刻释放，而是可以在方法体内持续持有引用，从而达到了预期的目的
- 然而在 release mode 下，即便是通过 makeRow 返回的对象在赋值给 weak 的引用时也立即释放导致异常。

- **解决方法**
解决方法是先使用强引用的局部变量持有新构造的对象，再赋值给 __block __weak 的变量，并在新构造的实例失去强引用之前，立刻对该变量进行一次变量的强引用（比如加入到数组中），如下：
```Objective-C
- (void)test__block__weak_3 {
    __block __weak XRow *row1 = nil;
    __block __weak XRow *row2 = nil;
    
    void(^block)(void) = ^{
        // 处理 row1 row2 的一些逻辑
        NSLog(@"row1 %@ row2 %@", row1, row2);
    };
    
    XRow *strong = nil;
    strong = [self makeRow];
    row1 = strong;
    row1.value[kBlockKey] = block;
    [self.array addObject:row1];
    // 使用一个强引用的局部变量
    //并在下一步失去 strong 的强引用之前立即装入数组进行强引用。
    
    strong = [self makeRow];
    row2 = strong;
    [self.array addObject:row2];
    row2.value[kBlockKey] = block;
    
    NSLog(@"array -> %@", self.array);
}
```


#### WKWebView
再基于 WKWebView 封装的控制器中，为 WebView 加载了一个自定义了 Post 请求体的 NSURLRequest 对象，这是接入一个第三方支付 URL 的场景，接入完成后，在 debug 模式下 各个系统和设备运行良好，但是在 release 模式下，iOS 8/9/10 的设备显示空白，提示错误 
> WKErrorDomain code 102 帧框加载失败

这种情况很可能是请求体传输不完整导致的，但是针对不同系统的差异原因在哪里，暂时无法直接解决。
**间接的解决方法**，使用 UIWebView，是的，使用 UIWebView 在 debug/ release 下均表现正常。

#### 小结
不论怎么样，给测试同学的包一定要给 release 的，有些情况即便没有手动判断是否 DEBUG 宏也会表现异常，测试同学发现了异常的情况要引起重视。

> PS：关于在 Xcode 切换 debug / release
- 第一步，在左上角的程序名称，选择 edit_scheme

![edit_scheme](http://upload-images.jianshu.io/upload_images/73339-5022b72014400c5b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 第二步，在 Run 的设置中的 Build Configuration 切换 debug / release，默认是 debug 模式下开发。

![切换 build](http://upload-images.jianshu.io/upload_images/73339-2a4da7dc68bd85d4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


> 感谢协同开发的同事大黄人、小黄人，以及前同事 Russell 在 Build 设置方面的提点。
