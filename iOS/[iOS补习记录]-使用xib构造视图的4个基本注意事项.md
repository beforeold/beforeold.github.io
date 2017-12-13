> 过去的2年多都是纯代码实现UI的，现在需要补习xib和自动布局的知识了。

### xib简介
xib应该是Xcode Interface Builder这一文件格式的缩写，它的前身是nib，是一种以xml形式描述界面构成和布局的方式，使用xib的好处主要是：
- 快速拖取控件，所见即所得
- 结合自动布局Autolayout方案，更轻松地实现不同屏幕的适配
- 控件的属性和事件都可以直接拖线，省去了大量的构建子控件和事件逻辑的代码
- 相比庞大的storyboard，xib对拆分独立的界面逻辑更加轻量级，适合多人开发

### 几个使用的基本注意点：
- 不会触发```initWithFrame:```或者```init```这两个代码编程的初始化方法
- 会触发```initWithCoder:```和```awakeFromNib```方法，因为原理上是从xml中解码后构建对象
- 注意nib内的对象在```initWithCoder:```的解码过后即已经生成了对象，包括子视图 ，但是在```awakeFromNib```之前，连线```IBOutlet```没有建立起来所以无效，所以要在初始化使用连线的属性的话，逻辑应该在```awakeFromNib```中写
- 对于使用```xib```的视图可以提供一个便利构造方法（如下）返回一个实例

```Objective-C
// 便利构造方法
+ (instancetype)someView {
    // 这里假设nib名称与类名相同
    return [[[NSBundle mainBundle] loadNibNamed:NSStringFromClass(self)
                                          owner:nil
                                        options:nil] firstObject];
}
```
