### 理解泛型 ```Generics```
从 ```Xcode``` 7 以后 ```Objective-C```（后称```OC```） 支持了泛型 ```Generics``` 的使用，相对 ```Swift``` 而言，```OC``` 的泛型要轻量级得多。```OC``` 的泛型的几点理解：
- 泛型用于指代一种类型，在 ```OC``` 中声明了泛型后至少是 id 类型
- 泛型往往在容器类中使用，比如 ```NSArray```、```NSDictionary``` 等
- 具体的类型，由使用具体的泛型指定。


### 泛型的应用实践
以一个 NSNumber 数组为例，看一下 NSArray 的泛型声明
```
@interface NSArray<__covariant ObjectType> : NSObject
```
使用泛型前，要访问一个数组内部对象的属性，只能通过 getter 方法，不能使用点语法，因为通过 index 取值的返回值是 id 类型，而 id 类型是不确定属性的。

```Objective-C
- (void)noGenerics {
    NSArray *numbers = @[@1, @2, @3];
    
    // NSInteger i = numbers[0].integerValue;
    // 编译报错 Property 'integerValue' not found on object of type 'id'


    NSInteger j = [numbers[1] integerValue];
    // 或者
    NSNumber *number = numbers[0];
    j = number.integerValue;
}
```
使用泛型后，返回值的类型是确定的，因而可以直接访问属性，十分便利。
```Objective-C
- (void)userGenerics {
// 泛型 <ObjectType> 这里是 NSNumber *
    NSArray <NSNumber *> *numbers = @[@1, @2, @3];
    NSInteger j = numbers[0].integerValue;
}

```
在其他 Fundation 的集合中也有这类应用，比如
```Objective-C
@interface NSMutableArray<ObjectType> : NSArray<ObjectType>
@interface NSDictionary<__covariant KeyType, __covariant ObjectType> : NSObject
@interface NSMapTable<KeyType, ObjectType> : NSObject
@interface NSSet<__covariant ObjectType> : NSObject
```
在这些集合对象的相关方法的参数以及返回值中都会用到泛型，有显而易见的好处：
- 比起 id 类型，在编译期间可以方便地进行访问参数的属性或者方法
- 在传参数时会得到编辑器的辅助，对于类型不匹配的对象会提出警告。
- 提高了代码的可读性，阅读代码的人可以清晰地知道内部对象的类型

```Objective-C
/// NSArray 带有泛型的方法定义
- (void)addObject:(ObjectType)anObject;
- (ObjectType)objectAtIndex:(NSUInteger)index;

NSMutableArray <NSNumber *> *array = [NSMutableArray array]; 
// 声明为 NSNumber 数组

[array addObject:@"str"]; // 在其中加入 NSString 对象会有编译警告
// incompatible pointer types sending 'NSString *' to 
// parameter of type 'NSNumber * _Nonnull'
// 事实上，Xcode 的自动完成会提示录入 一个 NSNumber 对象
// [array addObject:<#(nonnull NSNumber *)#>]
```
### 泛型结合协议的使用
如果声明泛型时同时希望对象响应特定的方法/属性，那么可以在泛型上附加协议，实际应用如下:
```
@protocol SomeProtocol <NSObject>

@property (nonatomic, copy) NSString *name;

@end

@interface ViewController ()

@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSArray <UIViewController <SomeProtocol> *> *array;

// 调用
- (void)useGenericProtocol {
    NSString *name = self.array.firstObject.name;
    NSLog(@"name-> %@", name);
}

@end

```

### 自定义泛型类
注意 泛型的使用，只能在 @interface 和 @end 之间，即类声明 拓展、分类中，是不能再实现中使用的。
可以限制泛型的范围，比如：
- 要求泛型继承某父类 :

**@interface GenericClass <__covariant T: SuperClass *>  : NSObject **
- 要求泛型遵循协议
**@interface GenericClass <__covariant T: id<SomeProtocol>>  : NSObject **
- 既要继承父类又要遵循协议
**@interface GenericClass <__covariant T:  SuperClass<SomeProtocol> *>  : NSObject **


 以一个弱引用容器举例 ```WeakWrapper```
```Objective-C
@interface WeakWrapper < __covariant T> : NSObject

@property (nonatomic, weak) T content; // 酷酷的泛型代替了 id

- (void)justCallAMethodUserGenerics:(T)someObject;
@end

@implementation WeakWrapper

- (void)justCallAMethodUseGenerics:(T)someObject {
        self.content = someObject;
}
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    WeakWrapper <UIViewController *> *ww1 = [[WeakWrapper alloc] init];
    ww1.content = self;
    // 直接访问 controller 的 title 属性
    NSLog(@"title-> %@", ww1.content.title);
    
    WeakWrapper <UIView *> *ww2 = [[WeakWrapper alloc] init];
    ww2.content = self.view;
    // 直接访问 view 的 frame 属性
    NSLog(@"frame-> %@", NSStringFromCGRect(ww2.content.frame));
}
```
嗯，看到这么吊炸天的 T 类型属性声明，就快要高潮了。


### 参考文章
[Swift教程 - 泛型](http://wiki.jikexueyuan.com/project/swift/chapter2/23_Generics.html)
[Objective-C 轻量级泛型](http://www.cnblogs.com/zenny-chen/p/5094075.html)
[Objective C Generics](http://drekka.ghost.io/objective-c-generics/)
[2015 Objective-C 新特性](http://blog.sunnyxx.com/2015/06/12/objc-new-features-in-2015/)
