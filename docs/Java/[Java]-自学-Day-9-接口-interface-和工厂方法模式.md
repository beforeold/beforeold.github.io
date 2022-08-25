#### 接口
接口是声明，定义了需要具备的功能要求，而不提供实现。

#### 接口的定义
- 使用 interface 关键字定义
- 可以理解为纯粹的抽象类，所有方法均为抽象方法（隐式）
- 接口所有方法的访问权限均为 public，（隐式）

```Java

//USB.java
interface USB {
    void read();
    void write();
}
```

##### 接口的使用，继承实现
与抽象类相同，需要使用的类对接口进行继承（实现），并实现复写接口中的抽象方法，且必须声明为 public （因为在接口声明中默认隐式的 public）。
```Java
class Phone implements USB {
      public void read() {
           // phone read
     }

    public void write() {
        // phone write
    }
}

// Phone p1 = new Phone();
// p1.read();
// p1.write();
```

#### 实现是特殊的继承
所以，可以进行必要的向上转型。 
```
USB u1 = phone1;
```

一个类可以实现多个接口（间接的多继承效果）
```Java
class Phone implments USB, TCP {}
```

一个接口可以继承多个接口。
```Java
interface InterfaceC extends InterfaceA, InterfaceB {}
```

#### 接口是面向对象吗？
据说是 Java 面向对象的重要特性。
如果将接口的遵循理解为继承的话，那么这也算多态的一种体现，就是面向对象的特性，这一点不展开。（对比多继承的对象语言C++）。

在 Swift 语言中，因为可以使用 extension 对接口进行扩展，所以 Swift 的开发有一种面向协议（接口）编程的范式。

> 对于 OC 开发者过来人，接口很容易理解。

#### 为什么要使用接口
回顾 Printer 的例子，父类 Printer 提供了三个方法，但是如果子类继承 Printer ，那么三个方法默认实现是一样的，如果忘记进行了复写，则可能导致实现不完全，使用接口则更合适地提炼出一些需要要求特别地实现的功能。
也就是说将 ```class Printer ``` 修改为 ```interface Printer```。
> 话说回来，也不是说抛弃了继承的使用，对于实现逻辑有共同点的，继承仍旧有用武之地。

> 不知道 Java 能否支持接口的扩展，边学边看吧。
 
#### 工厂方法模式
工厂方法是通过封装内部构造实例的逻辑的模式。
结合接口和工厂模式可以优化Printer 的使用：
- 避免打印机的根据条件创建实例过程和类型信息的暴露
- 将相应构造实例的逻辑封装到一个工厂方法中。

// 定义静态方法，不需要创造工厂实例（如有必要为工厂设置状态变量时，仍可使用实例）

```Java
interface Printer {}

class PrinterFactory {
   public static Printer getPrinter(int flag) {
        if (flag == 0) {
            return new HuiPuPrinter();
        } else if (flag == 1 {
             return new CanonPrinter();
         } else {
           // 默认打印机
         }
   }
}
```

这是封装同样代码的好处。（封装的利用不是单纯的面向对象的封装概念。）
