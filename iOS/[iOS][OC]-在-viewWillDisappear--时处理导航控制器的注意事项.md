### 背景
导航控制器 ```UINavigationController``` 是 iOS 应用基本非常常见的容器控制器，用于处理业务逻辑的延伸。在实践中，往往有这样的需求，比如：
- 隐藏导航栏
- 改变导航栏 ```UINavigationBar``` 颜色/透明度
- 隐藏导航栏下的分隔线 
- 其他自定义的需求
在这种情况下，常见的实践是在 ```viewWillAppear:``` 时做一些事情，在 ```viewWillDisappar:``` 时恢复原状，举一个隐藏导航栏的例子，如下:

```Objective-C
- (void)viewWillAppear:(BOOL)animated {
    [super viewwillAppear:animated];
    
    [self.navigationController hideNavigationBar];
}

- (void)viewWillDisappear:(BOOL)animated {
    [super viewWillDisappear:animated];
    
    [self.navigationController showNavigationBar];
}

@implementation UINavigationController (ToolCategory)
/// 隐藏和显示导航栏，实现略
- (void)hideNavigationBar { ... }
- (void)showNavigationBar { ... }
@end

```
做上述实现的控制器，正常的 ```push``` 到下一个页面，或者 pop 到下一个页面时导航控制器的功能会在 ```viewWillDisappear:``` 时得以调用恢复现状，而在一些特别的情况时，则会出现异常，比如：
- 直接 ```popToViewController:``` 到上上级页面，或者 ```popToRootViewController:```
- 该控制器被强制从导航控制器的栈中移除
上述两种情况下，导航控制器的导航栏因为被隐藏而无法显示，原因是：
**上述情况下调用 ```viewWillDisappar:``` 时， 通过 ```self.navigationController``` 已经无法获取到原有的导航控制器了，即此时: ```self.navigationController``` 是 ```nil```，因此此时调用 导航控制器的任何方法都将失效。**

### 解决方案
分析上述场景下导致导航控制器返回 nil 的原因在于，上述场景下导航控制器已经将其移除出栈，而通过 ```self.navigationViewController``` 这个 ```getter``` 属性返回的将是 nil，因此解决问题的思路是能够在 ```viewWillDisappear:``` 时也能够获取到原导航控制器，那么一个不错的思路就是，在控制器 ```viewWillAppear: ```时本身对导航控制器本身进行一次**弱引用**，这样只要导航控制器不销毁，则当前业务控制器在 ```viewWillDisappear:``` 时必然可引用到原导航控制器。实现如下：

```Objective-C
@interface BusinessVC ()

@property (nonatomic, weak) UINavigationController *my_naviVC;

@end

@implementation BusinessVC 

- (void)viewWillAppear:(BOOL)animated {
    [super viewwillAppear:animated];
    
    self.my_naviVC = self.navigationController;
    [self.my_naviVC hideNavigationBar];
}

- (void)viewWillDisappear:(BOOL)animated {
    [super viewWillDisappear:animated];
    
    [self.my_naviVC showNavigationBar];
}
```

@end



### 延伸

#### 利用 ```runtime``` 的 ```associatation``` 的方便这类场景的使用
如果这类场景较多，可以将弱引用这件事情处理到一个 ```UIViewController``` 的分类中使用，这样不用每次都去声明 **```weak```** 属性了，**注意要在有导航控制器的时候要调用一次此```getter```，保证已经将导航控制器进行了弱引用**，实现如下：

```Objective-C
@interface UIViewController (WeakNaviVC)

@property (nonatomic, weak, readonly) UINavigationController *my_naviVC;

@end

// #import <objc/runtime.h>
@implementation UIViewController (WeakNaviVC)

- (UINavigationController *)my_naviVC {
    UINavigationController *original = self.navigationController;
    if (original) {
        objc_setAssociatedObject(self, _cmd, original, OBJC_ASSOCIATION_ASSIGN);
        return original;
    }
    return objc_getAssociatedObject(self, _cmd);
}

@end
```


#### 关于强制移除出栈
有的时候一个控制器做为过渡使用，用过之后 push 到下一个页面则不再使用此控制器，如 A -> B -> C， B 是过渡使用的，push 到 C 后即需要将 B 移除出导航栈，以达到可以从 C 直接返回到 A 的目的，其实现依赖 ```UINavigatioinController``` 的 ```setViewControllers:``` 属性，其实现和使用如下：

```Objective-C

/// 使用分类实现，将自身移除出导航栈
@implementation UIViewController (RemoveFromNavigationStack)
- (void)br_removeFromNavigationControllerStack {
    NSMutableArray *newArray = [self.navigationController.viewControllers mutableCopy];
    [newArray removeObject:self];
    [self.navigationController setViewControllers:newArray animated:YES];
}
@end

/// B 控制器的实际调用场景
- (void)clickToPushToControllerC:(id)sender {
        ControllerC *vc = [[ControllerC alloc] init];
        [self.navigationController pushToViewController:vc animated:YES];
        [self br_removeFromNavigationControllerStack];
}



```


>讨论我的专题， QQ群 287698622
