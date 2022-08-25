## 背景
在有一些需要使用验证码的二级页面，发送验证码按钮点击后，会在下一次可以点击前会进行倒数读秒，以限制频繁地发送验证码请求，而此时用户点击返回按钮重新进入后，如无特别处理而重新加载一个新的验证码页面，此时发送按钮又可再点击，这在一定程度上是有违产品设计的初衷。

## 解决方案
比较好的实现方式是对验证码页面进行```懒加载+强应用```处理，这样省去其他方案依赖外部保存倒计时状态的处理。具体地，在调起验证码的页面，也即验证码的上一级页面，对验证码页面进行强引用，如此下一页返回时并没有释放，用户重复进入，验证码页面的原有倒数计时仍在进行，示例如下：

```Objective-C
@interface VerifyVC : UIViewController // 验证码控制器的声明

@end

@interface ViewController () // 调起验证码的控制器

// 强引用 验证码 控制器
@property (nonatomic, strong) VerifyVC *verifyVC;

@end

@implementation ViewController

// 调起验证码页面
- (IBAction)click:(id)sender {
    [self.navigationController pushViewController:self.verifyVC
                                         animated:YES];
}

// 懒加载 验证码控制器 示例
- (VerifyVC *)verifyVC {
    if (!_verifyVC) {
        _verifyVC = [[VerifyVC alloc] init];
    }
    
    return _verifyVC;
}

@end

```
## 延伸
不光是验证码控制器，一些需要进行网络请求的二级页面，如果这些页面可能会频繁进入而数据请求层没有进行历史请求数据进行有效地缓存处理。同样地，对这类二级页面进行```懒加载+强应用```，用户在第一次数据加载成功后，重复进入时旧的数据及控制器都在内存中缓存备用，数据得以直接展示而不以来重复请求，用户可以在这类二级页面之间**咻咻咻**地快速来回。
> PS: 如若数据加载失败，则应再次请求数据。


#### 欢迎简书交流。
