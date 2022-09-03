#### 背景
在典型的信息录入或者订单流程场景下，经常需要跳转到到一个二级页面去获取一些信息再回调到上一级页面，一般地，都会在回调时执行 ```[self.navigationController popViewController]```操作，举例从 ```A -> B``` 的回调如下：

```Objecitive-C
// B 控制器的声明
@interface ControllerB : UIViewController
/// 声明回调 block
@property (nonatomic, copy) void(^completion)(id userInfo);
@end

@interface ControllerB ()
/// 保存页面信息
@property (nonatomic, strong) id userInfo;
@end

@implementation ControllerB
/// 完成二级信息的编辑，点击完成按钮回调
- (void)completeClick:(id)sender {
        !self.completion ?: self.completion(self.userInfo);
        [self.navigationController popViewControllerAnimated:YES];
}
@end

@implementation ControllerA
/// 点击跳转二级页面
- (void)infoClick:(id)sender {
        ControllerB *vc = [[ControllerB alloc] init];
        vc.completion = ^(id userInfo){
                [self refreshWithUserInfo:userInfo];
        };
        [self.navigationController pushViewController:vc animated:YES];
}
/// 利用回调信息进行页面的刷新
- (void)refreshWithUserInfo:(id)userInfo { ... }

@end
```


不建议这样做，原因有以下两个：
- 有可能这个控制器是被模态```modally present``` 出来的，直接执行 ```pop``` 操作显然不合适
- 一级页面 A 在获取到回调信息时，未必要直接 ```pop``` 回来而是要在二级页面B执行进一步的操作，比如利用 B 页面完成后进一步 ```push``` 到 C 页面开展下一步流程。

#### 解决方案
针对性地，从三方面去考虑：
- 每个控制器有个各自的分工，当自身任务完成后的如果需要回调的情况，则应该回调出去，而不是承担额外的业务而导致不必要的耦合
- B 控制器任务完成后不一定要自己进行 ```pop```，而将 ```pop``` 的职责回调出去
- B 控制器也要将控制器自身回调方面调用者在自身页面执行下一步的处理。


回顾采用 ```delegate + protocol``` 设计回调的模式，好的实践会在协议中将实例自身回调出去，采用 block 做回调也应如此，实现如下:

```Objective-C
@class ControllerB;
@protocol ControllerBDelegate
@optional
- (void)controllerB:(ControllerB *)vc didCompleteWithUserInfo:(id)userInfo;
@end

@interface ControllerB : UIViewController
@property (nonatomic, weak) id <ControllerBDelegate> delegate;
@property (nonatomic, copy) void(^completion)(id userInfo, ControllerB *bVC);
@end

@interface ControllerB ()
@property (nonatomic, strong) id userInfo;
@end

@implementation ControllerB
/// 完成二级信息的编辑，点击完成按钮回调
- (void)completeClick:(id)sender {
      !self.completion ?: self.completion(self.userInfo, self);
      if ([self.delegate respondsToSelector:@selector(...)]) {
            [self.delegate controllerB:self didCompleteWithUserInfo:self.userInfo];
        }

        // 不再 pop
        //[self.navigationController popViewControllerAnimated:YES];
}
@end
```

如此，在一级页面调用控制器B则有了充分多的自由度，比如：
- 可以直接 ```pop``` 回来
- 或者，先执行一个自定义逻辑确认后，再 ```pop``` 回来
- 或者，不```pop```而是紧接着 ```push``` 到下一步的C页面，并将B移除出导航栈

分别实现如下：

```Objective-C
/// 举例一，跳转到二级页面获取信息后 pop 回来
- (void)infoClickCompleteToPop:(id)sender {
    ControllerB *vc = [[ControllerB alloc] init];
    vc.completion = &(id userInfo, ControllerB *bVC){
        // 刷新 本页面A的内容
        [self refreshWithUserInfo:userInfo];
        [self.navigationController popViewControllerAnimated:YES];
    };
    self.navigationController pushViewController:vc animated:YES];
}

/// 举例二，跳转到二级页面，获取信息后执行一个自定义逻辑后再 pop 回来
- (void)infoCickCompleteToPopAfterComfirm:(id)sender {
    ControllerB *vc = [[ControllerB alloc] init];
    vc.completion = ^(id userInfo, ControllerB *bVC){
        // 示例用于将信息用 UIAlertController 提示用户确认
        NSString *desc = [userInfo description]; 
        UIAlertController *alert = nil;
        alert = [UIAlertController alertControllerWithTitle:desc
                                                    message:nil
                                             preferredStyle:UIAlertControllerStyleAlert];
        UIAlertAction *done = nil;
        done = [UIAlertAction actionWithTitle:@"OK"
                                        style:UIAlertActionStyleDefault
                                      handler:^(UIAlertAction * _Nonnull action) {
              [self.navigationController popViewController];
              [self refreshWithUserInfo:userInfo];
          }];
        [alert addAction:done];
        // 注意这里使用 bVC 在 bVC页面展示 alertVC， 而不是 pop回来展示
        [bVC presentViewController:alert animated:YES];
    };
    self.navigationController pushViewController:vc animated:YES];
}

/// 举例三：信息提交后不pop而是直接push到 C 页面，并移除B页面
- (void)infoClickCompletionToPushThenRemoveControllerB:(id)sender {
    ControllerB *vc = [[ControllerB alloc] init];
    vc.completion = ^(id userInfo, Controller *bVC){
        ControllerC *cVC = [[ControllerC alloc] init];
        [self.navigationController pushViewController:cVC animated:YES];
        // 将 bVC 移除出导航栈，见下文备注
        [bVC br_removeFromNavigationStack];
    };
    [self.navigationController pushViewController:vc animated:YES];
}
```


> 关于移除出导航栈，见上一篇文章 [[iOS][OC] 在 viewWillDisappear: 时处理导航控制器的注意事项](http://www.jianshu.com/p/c6f71bceb22e)


#### 延伸
- 理清楚上文的思路后，也能发现将跳转逻辑回调而不是直接```pop```这种形式，在参数传递方面的好处：
对于一些需要在二级页面紧接着前往三级页面的情况，可以回调到一级页面，由一级页面跳转到三级页面，省去了跳转到三级页面时需要将一些业务参数，从一级页面传到到二级页面再传递到三级页面的过程。
- 对于一些公用的页面，将页面本身回调可方便不同业务的一级页面分别处理，可减少在二级页面引入不同业务场景下的业务判断和处理，降低耦合。
- 对于一些二级页面存在级联跳转的场景，比如 A -> B -> C，其中 C 确实是 B 的子业务页面，而不是 A 明确知悉的页面，合理地处理回调链也颇有价值。
举例如下：
A 业务页面需要从 B 列表页面选择一个 item，在 B 页面 可以直接选择一个 item，或者前往一个 item 的详情页面再选择一个 item。
上述场景下，没有确认的必要让 A 页面知悉 C 页面的存在，只需要在 B 页面的回调中综合考虑 可能在C 页面下发起业务回调到 页面A 即可，举例调整上文中， B 页面的回调声明、其他页面的声明及回调处理的实现如下：

```Objective-C

@interface ControllerC : UIViewController 
@property (nonatomic, strong) id itemInfo;
@property (nonatomic, copy) void(^completion)(id userInfo, ControllerC *cVC);
@end

@implementation ControllerC

- (void)c_completeClick:(id)sender {
    !self.completion ?: self.completion(self.itemInfo, self);
}

@end

@interface ControllerB : UIViewController 
/// 注意这里回调的控制器类型不再是 ControllerB * 而是 UIViewController *
/// 因为，这个回调的对象有可能来自 ControllerC *
@property (nonatomic, copy) void(^completion)(id itemInfo, UIViewController *anyVC);
@end

@implementation ControllerB : UIViewController
/// 当直接选择了列表的某个 item 时，回调 item 信息和自身
- (void)chooseSomeItemClick:(id)sender {
     id itemInfo = ...;// 获取点击的列表中某个 item 的信息
    !self.completion ?: self.completion(iteminfo, self); 
}

- (void)gotoSomeItemDetail:(id)sender {
     id itemInfo = ...;// 获取点击的列表中某个 item 的信息
     ControllerC *vc = [[ControllerC alloc] init];
      vc.itemInfo = itemInfo;
      vc.completion = ^(id userInfo, ControllerC *cVC){
          id itemInfo = ....;// 必要时由 userInfo 重组为 itemInfo
          // 注意此处传递回调参数为 cVC，而不是 self
          // 从而达到从 C 回调到 A 的效果
          !self.completion ? self.completion(itemInfo, cVC);
      }
    [self.navigationVC pushViewController:vc animated:YES];
}

@end

/// 一级页面 A 控制器的实现
@implementation ControllerA 

- (void)infoClick:(id)sender {
      // 将传递链上的控制器回调到一级页面，有足够的自由度进行特别的处理。
      Controller *bVC = [[ControllerB alloc] init];
      /// 注意，此处回调的 vc 未必是 ControllerB 
     /// 可能是由B 以后的其他类型
      bVC.completion = ^(id itemInfo, UIViewController *anyVC){
          // 举例一，pop 到当前页面
         // 其他情况，根据实际业务处理，可参考上述解决方案内容
        /// 注意这里是 popTo **self**，不是单纯的 pop 
        [self.navigationController popToViewController:self animated:YES];
        [self refreshWithUserInfo:itemInfo];
      }; 
      self.navigationController pushViewController:bVC animated:YES];
}

@end
```


#### 小结
一些整体是思路是：
- 回调控制器实例，便于调用者使用
- 由调用者进行 pop 或者回调后的进一步的处理
- 对于传递链较深的情况，可以约定将控制器逐级回调到最顶级
- 每个页面负责自身的业务，避免耦合多方面的业务处理。


>  补充： 回顾上述 block 的回调设计，仍旧不够充分考虑到：一种也需要知晓 block 持有者本身的情况，因此应新增一个回调参数，进行修改举例如下：
```Objective-C

typedef UIViewController * VC;
/// 第一个 actionVC 为触发回调的实际控制器，而 selfVC 为持有该 block 的控制器
@property (nonatomic, copy) void(^completion)(id userInfo, VC actionVC, VC selfVC);

/// 对于以下 delegate + protocol 的情况

- (void)controllerB:(ControllerB *)bVC didCompletionInVC:(VC)actionVC;

/// 同样地回顾一下类似 UINavigationControllerDelegate 或者 UITabbarControllerDelegate 的设计是类似的情形。

- (void)navigationController:(UINavigationController *)navigationController 
        willShowViewController:(UIViewController *)viewController
                       animated:(BOOL)animated;

```


>诚邀你来讨论我的专题， QQ群 287698622
