#### Object 的函数
Object 是所有类的父类，就像 OC 中的 NSObject，在 JDK 内。
因此每个类都继承了 equals 函数

``` boolean equals(Object o);```

#### 操作符 == 的作用
- 基本数据类型，是比较值是否相等
- 引用类型，判断两个引用是否指向内存中的同一个地址
在下图中， u1 == u3 ..> true , u1 == u2 ==> flase
![引用类型的相等判断](http://upload-images.jianshu.io/upload_images/73339-1e280699077437f4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 函数 equals 的作用
equals 的语义是比较两个对象的内容是否相等。
- 类型相同，使用 instanceof 操作符比较
- 成员变量的值完全相同

子类需要主动复写 equals 方法，才能得到判断内容相等的功能。

在 Object 内部的实现是：
```Java
//Object
boolean equals(Object o) {
    return this == o;
}
```
子类需要在必要时主动复写，注意：
- 内部的引用类型成员变量，一般使用 equals 判断是否内容相等
- 内部的值类型，使用 == 判断
- 内部的引用类型可能会出现 null，做好判断否则调用 equals 会抛异常
- 今后可以借助开发工具自动生成 equals 代码
- 为了更好的性能，先判断 ==，再判断是否同一类型，再继续
举例如下：
```Java

public class TestUser {
    int age;
    String name;
    
    public boolean equals(Object obj) {
        if (this == obj) return true;
        
        boolean sameClass = obj instanceof TestUser;
        if (!sameClass) return false;
        
        TestUser user = (TestUser)obj;
        boolean sameAge = this.age == user.age;
        boolean sameName = false;
        
        if (this.name == null) {
            if (user.name == null) {
                sameName = true;
            }
        } else {
            sameName = this.name.equals(user.name);
        }
        
        return sameAge && sameName;
    }
    
    public static void main(String[] args) {
        TestUser u1 = new TestUser();
//        u1.name = "Hi";
        TestUser u2 = new TestUser();
//        u2.name = "Hi";
        
        boolean b1 = u1 == u2;
        boolean b2 = u1.equals(u2);
        System.out.println(b1);
        System.out.println(b2);
    }
}
```
