#### 为区分同名类
```Java
package com.brook;
class User {}
```

**包名的声明必须在源文件第一行**

使用命令 ```javac -d . User.java``` 进行编译后，class 文件会放在包名路径下的文件中，如图：

![包名和类型关系](http://upload-images.jianshu.io/upload_images/73339-6d68838a354d27d0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

对应地，要运行该类则需要使用完整的类型 ```java com.brook.User```。


#### 包名的命名规范
- 所有字母均为小写
- 包名一般为域名倒过来写，如```com.apple.xxx```

注意，带```.``` 的包名，生成的文件夹会按照包名进行分割。
同一个包名编译生成的 class 在同一个文件夹内

#### 包名与访问权限关系
- 如果一个类声明为 public 则类名必须与源文件同名
- 权限的定义是包与包之间、包内文件与文件之间的访问权限隔离

#### import 关键字
- 如果不使用 import 关键字导入类名或者整个包，那么在需要使用其他包的类，则需要写全名:
```Java
com.brook.Person p1 = null;
```
- 如果使用 import 制定类名，则在使用外包的类时不需要写全名
```Java
import com.brook.User;

User u1 = null;
```
- 如果要用到某个包的多个类，可以使用通配符，但是前提是**当前类本身是后包名的**：
```
packge com.apple; // 当前类也必有包名
import com.brook;

User u2 = null;
```

#### 包、访问权限与继承之间的关系
- public class 的 default 的属性、函数可以继承过来，但无法使用
- public class 的 protected 属性、函数 可以只能被外部子类继承和使用，不能被外部非子类使用。
- protected 只用于修饰变量和函数，相当于在 default 的基础上，放宽到跨包访问时子类可以使用。

访问权限，可以理解为这个文件、这个函数、变量对另外一个类是否可见。
访问权限，是控制封装性的重要手段。

> 下面开始学习 Java 的协议 protocol 功能 --> 接口 interface
