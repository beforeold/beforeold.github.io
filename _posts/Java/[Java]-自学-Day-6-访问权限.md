#### 四种类型
```Java
public
private
default internal
protected
```
大多数是与包有关系的

```Java
package org.brook
    public class Person {
    public int age;
    public void introduce() {}
}
```

#### public 
public 的类，类名必须和文件名一样，否则可以不用

调用其它包的文件时，public 起到了是否可使用
如果是 pulic 使用外包的类时，需要使用全名，包名中不能带 '-'
所以，public 是不同包之间访问的控制

#### private
一般用于修饰变量和函数，private 只能在当前类中使用，修饰类时用于内部类，

默认不写权限修饰符时，就是 default，在同一个包内时可以调用，又称包级别权限

#### 包的导入
在使用类时要使用全称 com.brook.Peron
这样使用太长不方便，通过导入 import.brook.Person 即可直接使用 Person 类了，PS:除非当前包中恰好有一个同名的 default 及以上权限级别的 Person 类。
所有类 com.boork.* 导入所有的类。

#### protected 与继承有关系
在跨包的继承当中，子类将无法继承父类的成员变量和函数（准确地说，是因为权限问题，无法使用，这些成员变量和函数作为子类实例在实际上是具备的）

protected 首先有 default 同样的级别, protected 只能修饰变量和函数，可以用于跨包的子类内部访问，那么包外的其他类不能访问，包内的其他类是可以访问的。
 
综合地看: public > protected > default > private
利用访问控制，提高对象的封装型。
