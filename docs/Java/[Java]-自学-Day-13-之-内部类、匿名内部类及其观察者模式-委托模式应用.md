#### 语法
```Java
class A {
     class B {
     }
}
```

构造内部类实例，先 new 外部类对象，再 new 内部类对象，声明为 A.B
```Java
A a1 = new A();
A.B b1 = new A().new B();
```

#### 内部类对外部对象的成员变量和函数的访问
每个内部类对象都有一个外部类对象，内部类可以直接访问外部类对象的成员变量和函数，举例可直接访问外部对象的成员 ex 时，实际上相当于 ```A.this.ex ```：

```Java
class A {
      int ex;
      class B {
            int internal;
            void funcB() {
                    ex += 1; // 相当于 A.this.ex += 1;
            }
      }
}
```

这就像是 lambda 表达式的意思了，可以捕获外部对象数据，这样内部对象就可以作为函数参数或者函数的返回值了，是走向函数式的基础。（PS：Java 8 后有 lambda 表达式了，之前的话可以使用内部类来实现。）


####  匿名内部类
结合接口/继承使用，可以实现回调逻辑的封装（十分类似  lambda 表达式了）。
匿名类一般都是写在参数里，因为类的声明不重要，因此省去了声明部分，只保留内部实现，结合接口一起使用。

比如，某个 View 的有一个配置点击事件的回调方法：
```Java
interface OnClick {
     void onClick(SomeView view);
}


class SomeView {
      void setListener(OnClick listener) {
              this.listener = listener;
      }
}
```
当 View 发生点击事件时会向其监听者 listener 调用接口中声明的 OnClick() 函数：

```Java
    this.listener.onClick(this);
```

这样当某个类 XClass 在使用 View 时，为监听 View 的点击事件，需要成为其监听者，有两种方案：
- XClass 自身实现 OnClick 接口，成为 View 的监听者
```Java
class XClass implements OnClick {
         void useView() {
                 SomeView view = new SomeView();
                view.setListener(this);
          }
         
          void doMore() { }

         void onClick(SomeView view) {
                System.out.println("clicked on " + view);
         }
}
```
- XClass 声明一个匿名内部类对象作为监听者，在匿名类中实现接口，将回调的逻辑放在发生配置回调的地方，更紧凑。

```Java
class XClass implements OnClick {
         void useView() {
                 SomeView view = new SomeView();
                view.setListener(new OnClick{
                      void onClick(SomeView view) {
                             System.out.println("clicked on " + view);
                             doMore();
                      }
                });
          }
         
          void doMore() { }
}

```

#### 延伸
- 内部对象可以外部对象的函数和成员变量，这意味着一个外部对象对于其内部对象来说，资源是共享的，这就好比静态成员变量与静态函数的关系。
- 内部类因为可以捕获外部对象的属性，使得其可以用于专门抽象出外部对象的逻辑
- 比起来，lambda 表达式更具威力，期待以后的学习。还有同样有趣的 RxJava 哩。


> 感谢同事 **小明** 同学指导 Java 中匿名类的的应用场景，分享 Android 中的回调逻辑。
