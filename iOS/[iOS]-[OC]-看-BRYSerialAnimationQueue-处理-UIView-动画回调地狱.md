#### 背景
上文 [《自定义 Promise 处理串行的异步操作
》](http://www.jianshu.com/p/7677ad404f49) 分析了串行的异步操作的自定义 ```Promise``` 处理。```Promise``` 适用于类似串行的 ```UIView``` 动画的场景，如下：

```Objective-C
[UIView animateWithDuration:duration animations:^{
    label.alpha = 1;

} completion:^(BOOL finished) {
    [UIView animateWithDuration:duration delay:delay animations:^{
        label.alpha = 0;

    } completion:^(BOOL finished) {
        [label removeFromSuperview];
    }];
}];
```

```Promise``` 的处理

```Objective-C
- (void)promiseUIViewAnimation {
    UILabel *label;
    
    GSPromise *a = [GSPromise promiseWithHandler:^(dispatch_block_t then) {
        [UIView animateWithDuration:0.3
                         animations:^{ label.alpha = 1; }
                         completion:^(BOOL finished) { then(); }];
    }];
    
    GSPromise *b = [GSPromise promiseWithHandler:^(dispatch_block_t then) {
        [UIView animateWithDuration:0.3
                                delay:3
                              options:0
                         animations:^{ label.alpha = 0; }
                         completion:^(BOOL finished) {
                             [label removeFromSuperview];
                             then(); }];
    }];
    
    [GSPromiseManager manangerWithPromises:@[a, b]];
}
```

```BRYSerialAnimationQueue``` 的处理

```Objective-C
- (void)testBRYSeialAnimationQueue {
    UILabel *label;
    
    BRYSerialAnimationQueue *queue = [[BRYSerialAnimationQueue alloc] init];
    
    [queue animateWithDuration:0.3 animations:^{
        label.alpha = 1;
    }];
    
    [queue animateWithDuration:0.3 delay:3 options:0 animations:^{
        label.alpha = 0;
        
    } completion:^(BOOL finished) {
        [label removeFromSuperview];
    }];
}
```

#### 分析 ```BRYSerialAnimationQueue``` 的实现原理
```BRYSerialAnimationQueue``` （以下简称 ```BRYQueue```） 是专门用于 ```UIView``` 动画回调处理的类，将 ```UIView``` 动画的 ```API``` 复制到 ```BRYQueue``` 中，如此每个动画都移动到 ```BRYQueue``` 对象中去处理，而```BRYQueue```创建管理了一个串行队列，在动画方法的实现中将动画封装为
 block 任务放入到串行队列中，如下：

```Objective-C
// API

- (void)animateWithDuration:(NSTimeInterval)duration animations:(void (^)(void))animations
                 completion:(void (^)(BOOL finished))completion;


// 实现
- (void)animateWithDuration:(NSTimeInterval)duration animations:(void (^)(void))animations
                 completion:(void (^)(BOOL finished))completion {
    [self performAnimationsSerially:^{
        [UIView animateWithDuration:duration animations:animations completion:^(BOOL finished) {
            [self runCompletionBlock:completion animationDidFinish:finished];
        }];
    }];
}

```

串行队列的创建：
```Objective-C
- (id)init {
    if (self = [super init]) {
        _queue = dispatch_queue_create(NULL, DISPATCH_QUEUE_SERIAL);
    }
    
    return self;
}
```


其中，```performAnimationsSerially:``` 和 ```runCompletionBlock:animationDidFinish:```方法是实现的核心，分别处理了动画任务的开始和完成，动画任务的开始封装入一个 名为 ```animation``` 的 ```block``` 传入到串行队列中去异步执行，动画作为 ```UI``` 操作最终在主队列处理。

```Objective-C
- (void)performAnimationsSerially:(void (^)(void))animation {
    dispatch_async(self.queue, ^{
        self.semaphore = dispatch_semaphore_create(0);
        
        dispatch_async(dispatch_get_main_queue(), animation);
        
        dispatch_semaphore_wait(self.semaphore, DISPATCH_TIME_FOREVER);
    });
}
```

注意，其中使用了 ```GCD``` 的信号量控制```dispatch_semaphore```，此处的应用类似于一把线程锁，当一个动画任务开始前，创建一个值为 0 的信号量 ````dispatch_semaphore_create```，执行动画任务，等待任务完成  ```dispatch_semaphore_wait```，等待过程会导致当前线程，同时也是串行队列所管理的线程锁住，不再执行后续任务，直到上一个动画任务完成，完成的实现如下：

```Objective-C
- (void)runCompletionBlock:(void (^)(BOOL finished))completion animationDidFinish:(BOOL)finished {
    if (completion) {
        completion(finished);
    }
    
    dispatch_semaphore_signal(self.semaphore);
}

```
动画完成时执行上面的方法，释放一个信号后 ```dispatch_semaphore_signal```，线程的信号量恢复到 0 值，线程苏醒后即可安排继续下一个动画任务 ```animation block```。


#### 总结
[BRYSerialAnimationQueue](https://github.com/irace/BRYSerialAnimationQueue) 作为轻量级的 UIView 动画回调嵌套处理框架，从中可以学习到:
- ```GCD``` 的串行队列和信号量的使用来控制异步动画回调的连续执行，优化动画代码结构。
- 沿用 ```UIView``` 动画的 ```API```，没有使用上的转移学习门槛
- ```BRYSerialAnimationQueue``` 对象在执行中会一直被动画 block 所捕获，而不用担心释放，因此不必要额外地持有 ```BRYSerialAnimationQueue``` 对象，这一点和 自定义 的 ```GSPromiseManager``` 是类似的。
