# 背景
公司新的项目告一段落开始进行优化，进行一次内存泄漏的检查
使用的是由[腾讯微信阅读团队](http://wereadteam.github.io/2016/07/20/MLeaksFinder2/)开源的[MLeaksFinder](https://github.com/Zepo/MLeaksFinder/)工具，可以在调试阶段即时地发现项目中内存泄漏的情况，主要面向OC对象未释放的问题

# 发现UISwitch问题
在一个业务```ViewController```显示```UISwitch```未释放的问题，经过多次检查和排除后均未找到原因，开始怀疑```MleaksFinder```的准确性时，通过自定义一个```UISwitch```的子类重写```dealloc```方法来确认，发现使用确实未调用```dealloc```方法。

接着从网上找到一个可靠的解释，来自```RxSwift```一个[issue](https://github.com/ReactiveX/RxSwift/issues/842)的回答，说明使用Instruments-Leaks工具找出了UISwitch内部属性retain了UISwitch造成的循环引用，如下图：
![](http://upload-images.jianshu.io/upload_images/73339-4d657e5ab2341b36.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](http://upload-images.jianshu.io/upload_images/73339-23a2d85b343722c5.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
此外实测iOS9和iOS8下可顺利释放。
 这显然是UIKit自身的bug，已经有老司机上报了。

感谢一起调试的Alan和Luffy小伙伴

# demo
[demo 地址](https://github.com/beforeold/TestUISwitchNotDealloc)
