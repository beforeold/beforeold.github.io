### 未完待续

#### 0x0 集约型 API 设计的含义
概念来自 [Casa](https://casatwy.com/) 在 [iOS应用架构谈 网络层设计方案](https://casatwy.com/iosying-yong-jia-gou-tan-wang-luo-ceng-she-ji-fang-an.html) 中的分析，引用原文如下：
>集约型API调用其实就是所有API的调用只有一个类，然后这个类接收API名字，API参数，以及回调着陆点（可以是target-action，或者block，或者delegate等各种模式的着陆点）作为参数。然后执行类似startRequest这样的方法，它就会去根据这些参数起飞去调用API了，然后获得API数据之后再根据指定的着陆点去着陆。比如这样：
>```Objective-C
>集约型API调用方式：
>[APIRequest startRequestWithApiName:@"itemList.v1"
>                           params:params
>                            success:@selector(success:)
>                            fail:@selector(fail:)
>                            target:self];
>```

有集约型对应地就有离散型，引用原文如下：
>离散型API调用是这样的，一个API对应于一个APIManager，然后这个APIManager只需要提供参数就能起飞，API名字、着陆方式都已经集成入APIManager中。比如这样：
>```Objective-C
>离散型API调用方式：
>
>@property (nonatomic, strong) ItemListAPIManager *itemListAPIManager;
>
>// getter
> - (ItemListAPIManager *)itemListAPIManager
> {
>    if (_itemListAPIManager == nil) {
>        _itemListAPIManager = [[ItemListAPIManager alloc] init];
>        _itemListAPIManager.delegate = self;
 >   }
>
>     return _itemListAPIManager;
> }
> 
> // 使用的时候就这么写：
> [self.itemListAPIManager loadDataWithParams:params];
> ```
回归主题，集约型 API 的设计在服务器后端的设计整体较为统一的情况下，从使用的角度看十分的便利。因此，不妨以集约型的 API 的设计角度来梳理 HTTP 请求的过程。

#### 0x1 集约型 XCenterHTTP 框架
基于小伙伴 **Russell** 、 **Zoro** 封装 [AFNetworking](https://github.com/AFNetworking/AFNetworking) 的宝贵成果，进行适当的扩展形成了一套基础的HTTP 请求框架： **XCenterHTTP**。其属于集约型 API 设计，回调采用 block 的形式，核心代码共 300 行左右，实际调用的效果如下：

```Objective-C
[XCenterHTTP dataTaskWithMethodType:XMethodPOST
                                api:@"some/api"
                         parameters:@{@"key":@"value"}
                            success:^(id responseObject)
 {
     NSLog(@"successed -> %@", responseObject);
 }
                            failure:^(NSError *error, id responseObject)
 {
     NSLog(@"failed %@", error);
 }];
```
通过 XCenterHTTP 逐项梳理请求和回调的过程。

#### 0x3 梳理请求过程
> 以下内容分析的 AFNetworking 版本为 3.1.0
###### ① 配置 AFHTTPSessionManager
请求基于 AFHTTPSessionManager（下称 AFHTTP） 发起即可，从源代码可以了解到 AFHTTP 作为 AFURLSessionManager（下称 AFURL） 的子类，主要是增加了针对 HTTP 请求的参数和返回数据的序列化处理。所以总结：
- 在 AFURL 作为 NSURLSession 的 delegate 进行数据上下行交接
- AFHTTP 主要是对不同的 HTTP 请求方法以及数据的序列化

从 AFHTTP 的构造方法也可以看出其为 HTTP 请求做的主要工作：
- 保存 baseURL，准备请求序列化器，拼装 URL 和参数成为 NSURLRequest 对象
- 准备响应序列化器，用于对返回数据的解析

从而可以在父类的基础上完成 HTTP 请求，构造方法见源代码：
```Objective-C
- (instancetype)initWithBaseURL:(NSURL *)url
           sessionConfiguration:(NSURLSessionConfiguration *)configuration
{
    self = [super initWithSessionConfiguration:configuration];
    if (!self) {
        return nil;
    }

    // Ensure terminal slash for baseURL path, so that NSURL +URLWithString:relativeToURL: works as expected
    if ([[url path] length] > 0 && ![[url absoluteString] hasSuffix:@"/"]) {
        url = [url URLByAppendingPathComponent:@""];
    }

    self.baseURL = url;

    self.requestSerializer = [AFHTTPRequestSerializer serializer];
    self.responseSerializer = [AFJSONResponseSerializer serializer];

    return self;
}
```
除开 AFURL 对 NSURLSession 的封装，序列化的实现也是向 AFNetworking 学习的重点。回归到 XCenterHTTP 的集约型设计，设计一个单例对象持有一个 AFHTTP 实例即可：
- 构造单例
- 因为是单例，可以将单例封装在实现内部，以类方法提供 API
- 提供配置 BaseURL 接口以生成 AFHTTP 的 manager 实例作为实例变量，实现如下：
```Objective-C
+ (void)configureWithBaseUrl:(NSString *)baseURLString {
    NSURLSessionConfiguration *configuration = [NSURLSessionConfiguration defaultSessionConfiguration];
    // 构造 AFManager，默认是 HTTPRequestSerilizer， JSONResponseSerializer
    AFHTTPSessionManager *manager = nil;
    NSURL *baseURL = [NSURL URLWithString:baseURLString];
    manager = [[AFHTTPSessionManager alloc] initWithBaseURL:baseURL
                                       sessionConfiguration:configuration]; // 传 nil 等效
    
    [self sharedAPI].manager = manager;
}
```
作为请求序列化器，其核心功能是两点：
- 构造请求对象
- 使用协议来定义的，要求

其中，所谓的序列化除了基本拼装外，还包括数据存储结构的类型不同，常用的方式有 HTTP / JSON / XML 等，根据具体的服务端定义各不相同：
- 根据实际的情况选择不同的序列化器，特定的业务可以进行继承后自主实现
- AFHTTP 默认 HTTP **请求**序列化器（AFHTTPRequestSerializer），JSON **响应**序列化器（AFJSONResponseSerializer），其他序列化器均为对应的子类
- 接着可以配置一些默认的请求序列化参数，比如缓存策略、请求超时时间等：
```Objective-C
+ (void)configureDefaultRequestSerilizer {
    NSTimeInterval timeoutInterval = 15.f;
    NSURLRequestCachePolicy cachePolicy = NSURLRequestReloadIgnoringCacheData;
    
    AFHTTPRequestSerializer *http = [AFHTTPRequestSerializer serializer];
    http.timeoutInterval = timeoutInterval;
    http.cachePolicy = cachePolicy;
    
    XCenterHTTP *service = [self sharedAPI];
    service.manager.requestSerializer = http;
}
```

很多时候，也会需要在 HTTP 的请求中添加一些业务字段，比如用户 uid、令牌 token、设备和语言信息等，可以利用请求序列化器父类 AFHTTPRequestSerializer 的 setValue:forHTTPField: 方法：
```Objective-C
+ (void)setValue:(NSString *)value forHTTPRequestHeaderField:(NSString *)field {
    [[self sharedAPI].manager.requestSerializer setValue:value forHTTPHeaderField:field];
}
```
###### ② 请求发起


###### ③ 请求的回调


###### ④ 请求过程的拦截



#### 0x4 小结
**瓶颈：** 集约式的设计十分简便，粗放的封装不具备灵活性。面对多服务和个性化的请求需求时，就需要进行离散型设计。
**迁移：** 集约型的已经在用的接口，可以在引入离散型接口设计后，将集约型的底层实现调用离散型接口而保持旧代码的调用不受影响，在迁移的过程中两者共存是可行且适宜的。
