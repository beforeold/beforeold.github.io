## 业务的主要情况

业务的流程呈一条线的分布，如图

﻿﻿
![业务流程图](http://upload-images.jianshu.io/upload_images/73339-1194c8517e986e6d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


描述如下：

1. 从主页选择某一项业务页面，该页面将决定需要哪些可选业务

2. 按优先级1-2-3，依次跳转到需要的需要的可选服务页面（0-3项，如果没有可选服务则直接跳到必选业务）

3. 之后选择和配置两项必选服务

4. 将所有服务信息收集在确定订单展示

5. 跳转支付页面进行优惠券和支付方式选择后在线支付

6. 业务完成后跳回到主页。

## 代码的分块规范

不论是ViewController或者是View内部，需要遵循一定的分块规范，增强代码的可读性，这次的项目中因为是demo的性质，纯属UI和写死的数据传递，所以分为了LifeCycle、action/update、setter和getter三部分，前几日看了[Casa关于View层架构谈](http://casatwy.com/iosying-yong-jia-gou-tan-viewceng-de-zu-zhi-he-diao-yong-fang-an.html)的博客后觉得按照Casa的分层很好，摘出如下：

    #pragma mark - property                // class-extension 类扩展的属性
    #pragma mark - delegate                // 细分不同类型的delegate
    #pragma mark - event response     // 事件响应
    #pragma mark - private                    // 迫不得已的私有方法
    #pragma mark- setter and getter    //  懒加载或者其他目的复写

## 界面布局的注意点

因为本项目是纯代码开发，所以不会有xib或者storyboard那样直观，尽管开发是参照设计效果图展开，但是页面/视图的UI布局，实际开发的过程中因人而异，所以最好在代码注释中简单描述自己的布局方案，举一个简单的描述例子

    // 例子：该页面上部为TableView，下部为按钮

* 对于复杂的tableView的header/cell/footer的分组方案必要时也需要加以说明

* 对于一个视图内部，每个子视图的方位和分布思路也一定做好描述，这样进来一看就明白这个cell大概的样子

* 不在布局代码中写死数值(magic number)，写死的数值将来会难以追溯，应该声明为静态常量，并加以注释。



## 两个业务很相似的页面的开发

不建议使用同一个类或者继承，这样会牵一发就动全身，建议这样：

* 尽量将共用的东西独立抽取出来

* 既然相似那么先集中开发一个即可 ，等到业务基本稳定后再去复制过去开发另外一个，避免频繁地在两个业务之间来回修改



## 几个共用页面的调用逻辑设计，使用block回调

项目中页面是共用的，就像一个picker一样，比如地址/城市等类似业务参数的选择，由不同的同事独立开发，
在这些picker页面选择后回调一个数据模型即可（这里使用的是block的形式），同时这些Picker页面不负责页面的跳转逻辑，做到了相对的独立性


## 可选业务的跳转管理

流程中有0-3项可选服务，存在的问题
1. 从单个可选服务页面看，不确定是否需要显示
2. 当需要跳转下一个页面时，独立的逻辑中也无法指定下一个页面是什么

项目中实际的处理的办法时将定制的可选服务信息和页面跳转逻辑（也包括可选服务的回调信息）封装在一个类中，这个类做的工作：
- 统一记录需要定制哪些服务页面；
- 处理下一个页面的跳转逻辑
- 记录可选服务的所回调的业务信息
 
其中处理下一个页面跳转的逻辑是根据当前页面，根据流程定制的需求确定下一个需要的页面，同时设置下一个页面的回调。
```objc
- (UIViewController *)getNextViewControllerForController:(UIViewController *)controller {
    if (!controller) {
        // nil，则从头开始
        if (self.needRoute) {
            return [self getRouteVC];
            
        }else if (self.needHotel) {
            return [self getHotelVC];
            
        }else if (self.needTicket) {
            return [self getTickVC];
            
        }else {
            return [self getCarListVC];
        }
        
    }else if ([controller isKindOfClass:[RCChooseRouteViewController class]]) {
        if (self.needHotel) {
            return [self getHotelVC];
            
        }else if (self.needTicket) {
            return [self getTickVC];
            
        }else {
            return [self getCarListVC];
            
        }
    }else if ([controller isKindOfClass:[RCBookHotelViewController class]]) {
        if (self.needTicket) {
            return [self getTickVC];
            
        }else {
            return [self getCarListVC];

        }
        
    }else if ([controller isKindOfClass:[RCBookScenicSpotViewController class]]) {
        return [self getCarListVC];
    }

    return nil;
}
```

```objc
/**
 *  获取定制路线控制器
 */
- (UIViewController *)getRouteVC {
    RCChooseRouteViewController *routeVC = [[RCChooseRouteViewController alloc] init];
    routeVC.startCity = self.takeCity;
    routeVC.endCity = self.returnCity;
    __weak typeof(routeVC) weakRouteVC = routeVC;
    routeVC.selectedBlock = ^(id route) {
        self.hasRoute = YES;
        self.route = route; // 记录服务回调信息
        [weakRouteVC.navigationController pushViewController:[self getNextViewControllerForController:weakRouteVC] animated:YES];
    };
    
    return routeVC;
}
```

## 容器控制器（父子控制器）的应用
对于一个页面内部有不同部分的没有耦合但各自有较复杂的逻辑时，比如切换显示多个tableView的业务逻辑这种场景，可以为每个tableView单独简历一个controller，好处是各自的逻辑在各自的页面处理，给一个控制器拆分为三个控制器，控制器逻辑分开处理，是控制器瘦身和解耦的利器，见下图
﻿﻿
![容器控制器.png](http://upload-images.jianshu.io/upload_images/73339-4ec9ce690d97e288.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```objc
/**
 *  添加子控制器
 */
- (void)addChildViewControllers {
    self.customVC = [[RCCustomRouteViewController alloc] init];
    self.customVC.startCity = self.startCity;
    self.customVC.endCity = self.endCity;
    [self addChildViewController:self.customVC];    // 子控制器1
    
    self.recommendedVC = [[RCRecommendedRouteViewController alloc] init];
    [self addChildViewController:self.recommendedVC];   // 子控制器2
}
```

```objc
/**
 *  显示特定的子控制器
 *
 *  @param index 子控制器序号
 */
- (void)showViewControllerAtIndex:(NSInteger)index {
    [self.showingVC.view removeFromSuperview];
    [self.showingVC willMoveToParentViewController:nil];
    
    UIViewController *controller = self.childViewControllers[index];
    controller.view.frame = self.contentView.bounds;
    [self.contentView addSubview:controller.view];
    self.showingVC = controller;
    [self.showingVC didMoveToParentViewController:self];
}
```
