### 背景
由于 ```UIKit``` 的 ```UIAlertController``` 的样式无法满足业务自定义的要求，所以实际开发中经常会自定义 ```AlertController```，参考 ```UIAlertController``` 的范式，为不同的按钮事件设置不同的 ```block/closure/delegate``` 回调。以 ```UIAlertAction``` 为例：
```OC```
```Objective-C
+ (instancetype)actionWithTitle:(nullable NSString *)title 
                          style:(UIAlertActionStyle)style 
                          handler:(void (^ __nullable)(UIAlertAction *action))handler;
```
```<Swift>```
```Swift
public convenience init(title: String?, 
                            style: UIAlertActionStyle,
                         handler: ((UIAlertAction) -> Swift.Void)? = nil)
```
一般地，当点击 UIAlertController 的一个按钮时会触发对应 AlertAction 的闭包回调，比如实现 AlertController 内部的代码举例如下：
```Objective-C
- (void)buttonClick:(UIButton *)sender {
    AlertAction *action = .....;// 找到对应 sender 的 action
    !action.handler ?: action.handler(action);
    [self dismissViewControllerAnimated:YES completion:nil];
}
```
```Swift
func buttonClick(_ sender: UIButton) {
    let action = ...// 找到对应 sender 的 action
    action.handler?(action)
    dismiss(animated: true)
}
```
以上处理回调的思路可以实现
- 按钮点击事件的回调
- 移除 AlertController

但仍旧存在一个隐患在于回调的执行时机不够完善，外部在设置 AlertAction 的 handler 时需要再次 present 一个 AlertController，则会出现异常，无法 present成功:
系统的控制台错误信息如下：


![image.png](http://upload-images.jianshu.io/upload_images/73339-71e062ec7a0d5094.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

原因在于执行回调之前，原来的 AlertController 还没有彻底移除掉，此时再次 present 新的 AlertController 则会有异常。


### 解决方案
根据原因，可以优化为，回调的处理在彻底移除控制器之后，实现如下：
```Objective-C
- (void)buttonClick:(UIButton *)sender {
    [self dismissViewControllerAnimated:YES completion:^{
            AlertAction *action = .....;// 找到对应 sender 的 action
            !action.handler ?: action.handler(action);
    }];
}
```
```Swift
func buttonClick(_ sender: UIButton) {
        dismiss(animated: true) {
            let action = ...// 找到对应 sender 的 action
            action.handler?()
        }
}
```


### 延伸
- 对于一些场景下，点击按钮事件可能需要做一定校验后再移除 AlertController，如此以来，按钮事件不处理移除操作，而在外部调用额闭包内部必要时使用 dismissVC，同样地如果 dismissVC 后紧接着要 present 也需要注意上文的问题，同时还需要注意所有 AlertAction 都会被 AlertController 持有，使用闭包时应避免循环引用的发生。
- 可以考虑为 AlertController 提供一个独立的 presentInVC: 方法，在 present 之前需要注意判断当前是否有 presentedVC，如果有则可以先 dismiss 已有的控制器，这种情况在 present 登录控制器时会出现，实现如下：

```Objective-C
- (void)presentInVC:(UIViewController *)vc
           animated:(BOOL)animated 
          completion:(dispatch_block_t)completion  {
    dispatch_block_t presentBlock = ^{
        [vc presentViewController:self animated:true completion:completion];
    };
    
    if (vc.presentedViewController) {
        [vc dismissViewControllerAnimated:NO completion:presentBlock];
    } else {
        presentBlock();
    }
}
```

```Swift
func presentIn(_ vc: UIViewController, animated: Bool, completion: (()->Void)? = nil) {
    let presentClosure = {
        vc.present(self, animated: animated, completion: completion)
    }
    
    if vc.presentedViewController != nil {
        vc.dismiss(animated: false, completion: presentClosure)
    } else {
        presentClosure()
    }
}
```

> OC 代码中的 ```dispatch_block_t``` 是 ```GCD``` 中的一个宏定义，在接下来的文章中分享。
[四个 block 小技巧](http://www.jianshu.com/p/6189109859d6)


>诚邀你来讨论我的专题， QQ群 287698622
