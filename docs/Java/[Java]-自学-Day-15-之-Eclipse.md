#### 版本
支持 Java 9 需要 Eclipse OXYGEN，难怪之前的 Neon 在 Java 9 环境下无法启动。
使用 MacOS 也很方便。

#### 将自己的类定义在包内
新建一个 package -> 新建一个类

stubs => 模板代码填充，还支持新建类时指定需要实现的接口，并自动填充接口的实现代码部分。


![新建类](http://upload-images.jianshu.io/upload_images/73339-4c35e95e34096411.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 文件目录

Eclipse 会在保存后自动编译，并在 src 和 bin 文件夹下生成 二进制文件 .class
选择目录中的 .java 文件可以选择 run as Java Application， Eclipse 会执行对应的 .class 文件。

#### System.out.println()
现在，利用 Eclipse 可以查看文档来理解了，这个连续的点语法调用实际上是
- 获取 System 类
- 获取 out 这个静态变量 （OutStream 类型）
- 调用 这个变量的 println()方法

> 开源的 Java 真是太好了。编程早期，如果能有人指点尝试 Java 就好了。感慨。

![JDK 源代码 之 println](http://upload-images.jianshu.io/upload_images/73339-af6f2ac74bf7faa5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 错误处理
修改一次代码后，及时保存代码，这样编译器会重新编译查看新的编译结果。所以，使用 Eclipse 要经常手动保存。

#### 快捷键
option + / 提示自动完成，没事就点点
Command + D 删除一行
undo -> Command + Z
redo -> shift + command + Y
option + command + S 打开 source 菜单多个功能，其中有生成构造器功能
注释/取消注释 command + /
option + 上下，移动代码
command + option + 上下，赋值代码在上下

用快捷键做壁纸，是好主意。

#### 代码重构
读书：Martin Fowler 《重构-改善既有代码的设计》
提升代码的可重用性
- 改善软件设计
- 软件更容易理解
- 协助查找 bugs
- 提升开发效率

Eclipse
重构  shitf + command + T
重命名 shift + command + R
移动  shift + command + V
修改方法的签名 shift + command + C 
方法移动到父类/子类 pull up /  push down 
抽取类 extract class
