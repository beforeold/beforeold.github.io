### 背景

![](http://upload-images.jianshu.io/upload_images/73339-4274e9baf99ee3c8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
SDWebImage 是著名的 iOS 的 OC 第三方库，可以很好地辅助开发者做好网络图片的请求加载和缓存工作，而在一些后台逻辑场景下，存在着局限性，需要开发者去扩展。比如，通过URL请求一张图片，此图片不存在或者图片无权限时，后台接口不是返回错误 statusCode，而是会返回一个提示出错的图片，并在返回 response 的请求头中协议一个错误 errorCode 字段，SDWebImage 成功接收到一张图片后即展示并缓存图片，一段时间内都无法再次请求这个 URL 图片，因而出现这类出错的图片长期无法刷新的问题。
 针对这类问题，分两种情况，
 - 如果这类情况只针对少数图片，在设置图片时设置缓存策略```SDWebImageOptions```为“仅适用于内存缓存”```SDWebImageCacheMemoryOnly```，如此不存在硬盘缓存从而会发起新的请求
 - 如果这种情况十分普遍，则合适的方案是对 SDWebImage 的每一次请求的回调进行请求头的过滤，当存在错误 errorCode 字段时不再执行成功的回调而走向失败，显示 app 端的占位图片 placeholderImage ，以避免在硬盘中缓存了错误图片。
 
### 请求头过滤的实现
 SDWebImage中，网络请求的发起和回调的操作处理由 ```SDWebImageDownloader``` 执行，最后转发给 ````SDWebImageDownloaderOperation```这个类进行操作处理，此类继承自```NSOperation```，其遵循了```NSURLSessionTaskDelegate``` 和 ```NSURLSessionDataDelegate```两个协议，封装了请求的发起和回调逻辑，通过将这些操作加入 NSOperationQueue 后进行管理和执行。
 查看 SDWebImageDownloaderOperation 的源码可以发现，执行请求成功/失败的判断是在 URLSessionDelegate 的一个协议方法中执行的，代码如下：
 ```Objective-C
 - (void)URLSession:(NSURLSession *)session
 dataTask:(NSURLSessionDataTask *)dataTask
 didReceiveResponse:(NSURLResponse *)response
 completionHandler:(void (^)(NSURLSessionResponseDisposition disposition))completionHandler {
 
 //'304 Not Modified' is an exceptional one
 if (![response respondsToSelector:@selector(statusCode)] || ([((NSHTTPURLResponse *)response) statusCode] < 400 && [((NSHTTPURLResponse *)response) statusCode] != 304)) {
     NSInteger expected = response.expectedContentLength > 0 ? (NSInteger)response.expectedContentLength : 0;
     self.expectedSize = expected;
     if (self.progressBlock) {
        self.progressBlock(0, expected);
     }
     
     self.imageData = [[NSMutableData alloc] initWithCapacity:expected];
     self.response = response;
     dispatch_async(dispatch_get_main_queue(), ^{
     [[NSNotificationCenter defaultCenter] postNotificationName:SDWebImageDownloadReceiveResponseNotification object:self];
     });
 }
 else {
     NSUInteger code = [((NSHTTPURLResponse *)response) statusCode];
     
     //This is the case when server returns '304 Not Modified'. It means that remote image is not changed.
     //In case of 304 we need just cancel the operation and return cached image from the cache.
     if (code == 304) {
        [self cancelInternal];
     } else {
        [self.dataTask cancel];
     }
     dispatch_async(dispatch_get_main_queue(), ^{
        [[NSNotificationCenter defaultCenter] postNotificationName:SDWebImageDownloadStopNotification object:self];
     });
     
     if (self.completedBlock) {
        self.completedBlock(nil, nil, [NSError errorWithDomain:NSURLErrorDomain code:[((NSHTTPURLResponse *)response) statusCode] userInfo:nil], YES);
     }
        [self done];
     }
 
     if (completionHandler) {
         completionHandler(NSURLSessionResponseAllow);
     }
 }
 
 ```
 从中可以看到当请求回调的 response 中 statusCoe 在一定范围内( s < 400 && s != 304)会走向业务成功的处理（保存图片数据和reponse），否则走向失败的处理（取消请求任务 task->cancel），因此应该在执行 response 的判断之前，进行拦截。
 > 顺便说一下，这里可以看到 NSURLSession 的回调，采用的是 ```delegate+block``` 的```回调后再调用`` 的设计思路。可以参考文章了解： [[iOS] [OC] 关于block回调、高阶函数“回调再调用”及项目实践](http://www.jianshu.com/p/5d0c85f9abcf)
 
 在不修改 SDWebImage 源代码的情况下，有两种可行的方案，分别是多态和runtime交换方法实现 method-swizzling，分别实现如下:
#### 多态方案
 从 SDWebImageDownloader 的 API 中可以找到用设置 SDWebImageDownloaderOperation 生成类的方法 ```setOperationClass:```:
 ```Objective-C
 
 * Sets a subclass of `SDWebImageDownloaderOperation` as the default
 * `NSOperation` to be used each time SDWebImage constructs a request
 * operation to download an image.
 *
 * @param operationClass The subclass of `SDWebImageDownloaderOperation` to set
 *        as default. Passing `nil` will revert to `SDWebImageDownloaderOperation`.
 
- (void)setOperationClass:(Class)operationClass;
 ```
 也就是说，可以通过设置自定义的 SDWebImageDownloaderOperation 子类来自定义请求逻辑，再结合继承后多态的特性，在子类中 复写协议方法对 NSURLSessionTaskDelegate 方法进行重写判断完请求头后再调用 super 继续父类原有逻辑。代码如下：
 ```Objective-C
 @interface WBWebImageDownloaderOperation : SDWebImageDownloaderOperation
 
 @end
 
 @implementation WBWebImageDownloaderOperation
 
 - (void)URLSession:(NSURLSession *)session
 dataTask:(NSURLSessionDataTask *)dataTask
 didReceiveResponse:(NSHTTPURLResponse *)response
 completionHandler:(void (^)(NSURLSessionResponseDisposition disposition))completionHandler {
     NSInteger statusCode = response.statusCode;
     NSDictionary *headers = response.allHeaderFields;
 
     if ([headers[@"ErrorCode"] integerValue] == 100) {
     // 异常
     statusCode = 444; // 写任意一个 SDWebImageDownloaderOperation 会判断为错误的 code 即可 （大于400）
     // 置换一个新的实例
     response = [[NSHTTPURLResponse alloc] initWithURL:response.URL
                                                    statusCode:statusCode
                                                    HTTPVersion:@"HTTP/1.1" // 这里写死了没有影响
                                           headerFields:headers];
 }


 [super URLSession:session dataTask:dataTask didReceiveResponse:response completionHandler:completionHandler];

 }
 
 @end
 ```
 其核心是当 errorCode 判断无效时，返回一个伪造的异常 response 使得原有逻辑走向错误处理 else，在 程序启动时配置好下载图片的子类如下:
 ```Objective-C
 // application:didFinishLaunchWithOptions:
 [[SDWebImageDownloader sharedDownloader] setOperationClass:[WBWebImageDownloaderOperation class]];
 
 ```

方案二：利用 method-swizzling，在调用 SDWebImageDownloader 方法之前，先 执行请求头的判断，再执行原方法。过程是创建 SDWebImageDownloaderOperation 分类 category，交换内部代理方法的实现，尝试实现如下:
 ```Objective-C
 #import "SDWebImageDownloaderOperation+WBSwizzle.h"
 #import <objc/runtime.h>
 
 @implementation SDWebImageDownloaderOperation (WBSwizzle)
 
 + (void)load {
   [self swizzleInstanceMethod:@selector(URLSession:dataTask:didReceiveResponse:completionHandler:)
                   withMethod:@selector(wb_URLSession:dataTask:didReceiveResponse:completionHandler:)
                       class:self];
 }
 
 - (void)wb_URLSession:(NSURLSession *)session
 dataTask:(NSURLSessionDataTask *)dataTask
 didReceiveResponse:(NSHTTPURLResponse *)response
 completionHandler:(void (^)(NSURLSessionResponseDisposition disposition))completionHandler {
     NSInteger statusCode = response.statusCode;
     NSDictionary *headers = response.allHeaderFields;
     if ([headers[@"ErrorCode"] integerValue] == 100) {
     // 异常
     statusCode = 444; // 写任意一个 SDWebImageDownloaderOperation 会判断为错误的 code 即可 （大于400）
     // 置换一个新的实例
     response = [[NSHTTPURLResponse alloc] initWithURL:response.URL
          statusCode:statusCode
        HTTPVersion:@"HTTP/1.1" // 这里只能写死了，应该没有影响
      headerFields:headers];
 }
 
 [self wb_URLSession:session dataTask:dataTask didReceiveResponse:response completionHandler:completionHandler];
 }
 
 + (void)swizzleInstanceMethod:(SEL)origSelector withMethod:(SEL)newSelector class:(Class)cls
 {
 // if current class not exist selector, then get super
Method originalMethod = class_getInstanceMethod(cls, origSelector);
Method swizzledMethod = class_getInstanceMethod(cls, newSelector);
// add selector if not exist, implement append with method
if (class_addMethod(cls,
                    origSelector,
                    method_getImplementation(swizzledMethod),
                    method_getTypeEncoding(swizzledMethod)) ) {
    // replace class instance method, added if selector not exist
    // for class cluster , it always add new selector here
    class_replaceMethod(cls,
                        newSelector,
                        method_getImplementation(originalMethod),
                        method_getTypeEncoding(originalMethod));
    
} else {
    // swizzleMethod maybe belong to super
    class_replaceMethod(cls,
                        newSelector,
                        class_replaceMethod(cls,
                                            origSelector,
                                            method_getImplementation(swizzledMethod),
                                            method_getTypeEncoding(swizzledMethod)),
                        method_getTypeEncoding(originalMethod));
}
}

@end
 ```
### 小结
两种方案的效果是一致的，在 SDWebImage 有 API 支持的情况下，建议优先使用既有 API 的方案一， 而 swizzle 可以在其他类似场景下使用。
