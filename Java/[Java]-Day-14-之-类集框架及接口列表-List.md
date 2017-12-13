#### 类集框架
- JDK 一组类和接口
- 包名 java.utl
- 主要用于存储和管理**对象**
- 主要分为三类，集合、列表和映射
- 与数组相比，有更丰富的数据结构进行数组存储

#### 集合 Set
- 集合内的元素，不按特定方式排序
- 集合内无重复对象

![集合示意图](http://upload-images.jianshu.io/upload_images/73339-e3b271da776e0e5f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 列表 List（接口）
- 按照索引 index 位置排序
- 可以有重复的对象

![列表示意图](http://upload-images.jianshu.io/upload_images/73339-fe7aaea44b1f7252.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 映射 Map（接口）
- 按照键值对存储，一个键配对一个值
- 映射的元素对没有排序
- 键不可重复，值可以重复

![映射示意图](http://upload-images.jianshu.io/upload_images/73339-34af7892f38d5cb5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 类集框架的主体结构
- 头两层设计，均为接口
- 先了解顶层接口的定义，再了解子类的实现，再了解子类的其他函数

![类集框架的结构](http://upload-images.jianshu.io/upload_images/73339-c82c928601d98d9b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 练习之 ArrayList
- ArrayList 是泛型类型，在构造时需要指明内部元素类型。
- 类集框架，只能存储对象，不能是基本数据类型
```Java
import java.util.List;
import java.util.ArrayList;

public class TestArrayList {
    public static void main(String[] args) {
        ArrayList<String> a1 = new ArrayList<String>();
        System.out.println(a1);
        
        a1.add("a");
        a1.add("b");
        a1.add("c");
        
        for (int i = 0; i < a1.size(); i++) {
            System.out.println(a1.get(i));
        }
    }
}
```

- 添加元素 ```add(obj)```
- ArrayList 的访问，提供独立的函数 ```get(indx)```。
- 移除元素 ``` remove(index)```
与数组一样，下标的访问不能超出其元素范围。

#### ArrayList 的遍历
- 利用其长度，提供长度函数 ```int size();```
类似数组的遍历

- 利用枚举器和迭代器
在多线程下，还要注意数组数据的安全。参考文章。[《Iterator和Enumeration的区别》](http://www.jianshu.com/p/ff27fe21cb03)。
