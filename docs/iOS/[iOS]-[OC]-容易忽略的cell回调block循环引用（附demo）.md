## 循环引用简介
在`Objective-C`的开发中循环引用(`retain cycle`)是指两个（或多个）对象之间产生了互相强引用而导致这些对象因为引用计数(`reference count`)始终大于等于1而不会释放，最后导致内存泄漏(`memory leak`)的状况，可以用下图描述。

![循环引用的产生](http://upload-images.jianshu.io/upload_images/73339-a95e89fcf3e60943.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

因此，解决循环引用的思路，就是避免两个（或多个）对象之间互相产生强引用，常见的`UITableView`等控件的delegate属性的声明使用的就是弱引用（`weak`/`assign`)，避免了`controller`强引用某个对象时，同时作为该对象的`delegate`被该对象强引用产生的循环引用。

## 关于block产生的循环引用
在`OC`的开发使用块(`block`)编程因为简便快捷而越来越常见，而使用`block`时需要注意的一个重点是避免产生循环引用，原因主要在于：
- `block`常常作为`delegate`的替代方案使用，声明为一个对象（**举例**命名为`BlockObject`）的属性时默认为`copy`策略，即`BlockObject`会`copy`并强引用传入的`block`
- `block`的特性会对其内部引用的所有对象进行一次强引用
- 那么如果在`controller`中引入了`BlockObject`属性并强引用，而在设置其`block`属性时传入了`controller`自身，循环引用也就产生了，这种情况下`Xcode`编译器会提示警告，如下图。

![两个对象直接的循环引用](http://upload-images.jianshu.io/upload_images/73339-5590dcc99071dbf6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

对于`block`使用产生的循环引用，解决的办法通常是在`block`外部创建一个`weak`指针后再传入`block`内部，这样循环引用就被断开了。
```objc
    __weak typeof(self) weakSelf = self;
    self.blockObject = [[BlockObject alloc] init];
    self.blockObject.block = ^(){
        [weakSelf log];
    };
```

## Cell使用block潜在的循环引用
回到标题说的情况，使用`UITableViewCell`或者`UICollectionViewCell`的子类定制`cell`时，会遇到`cell`上有个独立的按钮事件需要回调，当使用`block`来实现这个回调的设计时就会发生一个容易忽略的循环引用，`Xcode`编译器无法发现这类隐形循环引用，没有警告提醒，如下图没有警告，但循环引用已经产生，当该控制器被`pop`出去时不会销毁`dealloc`。

![Cell的隐形循环引用](http://upload-images.jianshu.io/upload_images/73339-bdaadfe6c5683cfe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

分析其原因在于`cell`实际是`tableView`的子视图，每个子视图都是会被其父视图的`subviews`（`NSArray *`）属性所强引用，即`tableView` ~> `subviews` ~> `cell`，而`cell`因为使用`block`作为回调强引用了`block`内部的对象，形成了这样的循环引用链条，即 `controller` ~> `tableView` ~> `cell` ~> `controller`，解决的方法同样是使用弱引用传入`block`，如下图所示。

```objc
    __weak typeof(self) weakSelf = self;
    cell.cellBlock = ^(id cell) {
        [weakSelf iLog];
    };
    
    return cell;
```


值得注意的是，在这个例子中，即使`tableView`属性的声明为`weak`，循环引用仍然会产生，原因在于`tableView`还是`controller`的`view`属性的子视图，强引用链接同样存在，因此最好是在`block`内部切开强引用链条。


![Cell的block循环引用的产生和解除](http://upload-images.jianshu.io/upload_images/73339-63c0405e9e17c79c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 小结
使用`block`比起使用**协议+代理**的方式因为可以直接捕获上下文中的变量而简洁方便，但潜在的隐患不能忽视，每次使用block时除了留意编译器的警告外，也要对可能的循环引用进行一番推敲，类似于```UITablerHeaderFooterView```的 回调 ```block```，以及一些控制器直接/简洁地强引用的对象所持有的 ```block``` 处理，都需要保持谨慎。

此外，对于工程内的内存泄漏，`Xcode`本身提供了检查工具(`Leak`)，也有第三方的内存泄漏检查工具值得体验。

## Tip：一行代码安全调用block
一般调用`block`调用时都要对其是否为nil进行判断，是处于安全的考虑，如下
```objc
if(block) {
    block();
}
```
推荐一个优雅的等效写法
```!block ?: block();```


## 鸣谢
感谢 Alan 同学在开发中遇到这个问题让我避免以后再走老路。

[demo地址](https://github.com/beforeold/TestCellBlockCycleRetain)
