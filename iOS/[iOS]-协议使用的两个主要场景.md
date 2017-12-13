#### 约定接口
协议最常用的场景是，约定接口，作为接口的抽象而存在，不提供实现，在 iOS 中最常用的场景就是委托模式下的 delgate / dataSource，以及约定对象应该具备的特征或功能，比如：
- UITableViewDelegate/UITableViewDataSource，用于约定回调的对象具备的数据源或者特定事件发生后需要执行的回调
- NSCopying, NSCoding 等
- UITextField/ UITextView 都遵循的 UITextInput 协议，约定了文本输入的对象具备的功能和行为表现
- MapKit 中地图大头针数据源 MKAnoation 协议应该具备 title 和 coordinate 属性
- [Casa 在 [iOS应用架构谈 网络层设计方案](https://casatwy.com/iosying-yong-jia-gou-tan-wang-luo-ceng-she-ji-fang-an.html)](https://casatwy.com/iosying-yong-jia-gou-tan-wang-luo-ceng-she-ji-fang-an.html) 在描述网络层数据处理时数据转换器类应该具备的功能，以及网络服务 HTTPSerbvice 子类对应必须实现的方法等。
- 自定义一个 UIPickerView，传入一个数组，即可进行业务选项的选择，效果如下：

![自定义 Picker](http://upload-images.jianshu.io/upload_images/73339-6dc3fddb3ae940b1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/560)
将 Picker 的接口进行抽象，传入一个 ```NSArray <id <Namable>> * ```数组，可以使这个 Picker 变得更为通用，定义协议如下：
```Objective-C
@protocol XNamable <NSObject>

@property (nonatomic, copy, readonly) NSString *name;

@end
```
这样，需要展示的对象进行遵循即可传入 Picker 进行使用：
```
@interface XSome : NSObject

@property (nonatomic, copy) NSString *text;
@end

// 写一个 category 进行实现

@interface XSome （XNamable） <XNameable>

@end

@implemenation (XNamable) 
/// 实现协议
- (NSString *)name {
    return self.text;
}

@end


```


#### 隐藏实现
有时候为了不暴露对象实际的实现，或者对象的实现由多种方式，对外提供接口时，不提供具体的类型对象，而是遵循了特定协议的对象，比如：

- SDWebImageDownloader 作为 [SDWebImage](https://github.com/rs/SDWebImage) 负责下载图片的对象，在下载图片时，返回一个 id <SDWebImageOperation> 对象，只提供了一个 cancel 方法：
```Objective-C

@protocol SDWebImageOperation <NSObject>

- (void)cancel;

@end
```
其实际内部实现进行了封装，不对外提供更多的接口调用。

```Objective-C
- (id <SDWebImageOperation>)downloadImageWithURL:(NSURL *)url
                                         options:(SDWebImageDownloaderOptions)options
                                        progress:(SDWebImageDownloaderProgressBlock)progressBlock
                                       completed:(SDWebImageDownloaderCompletedBlock)completedBlock;
```

- 通知中心 NSNotificationCenter 使用 block 添加观察者时会返回一个观察者，这个返回值只用来必要时做移除使用，所以不需要对外暴露具体类型，所以返回一个 id<NSObject> 即可，这一点在前文有所讨论[[iOS] [OC] NSNotificationCenter 进阶及自定义（附源代码）](http://www.jianshu.com/p/4b2dff2bbdb4)：

```Objective-C
/// 添加观察者后返回一个观察者对象
- (id <NSObject>)addObserverForName:(nullable NSNotificationName)name 
object:(nullable id)obj 
queue:(nullable NSOperationQueue *)queue 
usingBlock:(void (^)(NSNotification *note))block;

/// 可用于必要时移除观察者
- (void)removeObserver:(id)observer
 name:(nullable NSNotificationName)aName 
object:(nullable id)anObject;
```

- 甚至在必要时声明一个空协议返回对象，用于下文时的处理，更像是一种标识而不是具体的类型，这是基于 NSNotificationCenter 的思路的进一步延伸，此外典型的如 Swift 中的 Error 协议，其声明为空协议。

```Swift
public protocol Error {
}
```

#### 延伸
在 Swift 中，可以对协议进行扩展 extension，从而使得协议具备默认的实现能力，以 XProtocol 举例：
```Swift
protocol XProtocol { 
    func foo();
}

extension XProtocol {
    func foo() {
        print("say foo.")
    }
}
```
在 OC 中无法直接将协议本身作为一种类型来进行扩展，但是借助一些手法，仍旧可以让协议具备默认实现，比如 [ProtocolKit](https://github.com/forkingdog/ProtocolKit)，
可以针对协议：
```Objective-C
@protocol Forkable <NSObject>

@optional
- (void)fork;

@required
- (NSString *)github;

@end
```
提供默认实现如下：
```Objective-C
@defs(Forkable)

- (void)fork {
    NSLog(@"Forkable protocol extension: I'm forking (%@).", self.github);
}

- (NSString *)github {
    return @"This is a required method, concrete class must override me.";
}

@end
```
基于这种情况，可以对一些需要实现千遍一律协议的情况提供统一的实现，比如空的列表页面展示，利用 [DZNEmptyDataSet](https://github.com/dzenbot/DZNEmptyDataSet) 做默认实现。

```Objective-C
@defs(DZNEmptyDataSetSource)

- (NSAttributedString *)descriptionForEmptyDataSet:(UIScrollView *)scrollView
{
    NSString *text = @"暂无数据";
    
    NSMutableParagraphStyle *paragraph = [NSMutableParagraphStyle new];
    paragraph.lineBreakMode = NSLineBreakByWordWrapping;
    paragraph.alignment = NSTextAlignmentCenter;
    
    NSDictionary *attributes = @{NSFontAttributeName: [UIFont systemFontOfSize:14.0f],
                                 NSForegroundColorAttributeName: [UIColor lightGrayColor],
                                 NSParagraphStyleAttributeName: paragraph};
    
    return [[NSAttributedString alloc] initWithString:text attributes:attributes];
}

- (CGFloat)verticalOffsetForEmptyDataSet:(UIScrollView *)scrollView
{
    return 10;
}
@end
```
再结合 objc_assocatioin 进行适当的封装，即可便利地提供空数据的默认展示。
```Objective-C
@interface GSEmptyDataSetHandler : NSObject <DZNEmptyDataSetSource, DZNEmptyDataSetDelegate>

@end

@interface UIScrollView ()

@property (nonatomic, strong, readonly) GSEmptyDataSetHandler *gs_emptyDataSetHandler;

@end

@implementation UIScrollView (GSEmptyDataSet)
- (void)gs_setupEmptyDataSet {
    self.emptyDataSetSource = self.gs_emptyDataSetHandler;
    self.emptyDataSetDelegate = self.gs_emptyDataSetHandler;
}
```

> 简书拖入图片后，从图床返回的图片URL 结尾是 ```../w/1240``` 的字样，可以根据实际情况进行调整，优化排版
