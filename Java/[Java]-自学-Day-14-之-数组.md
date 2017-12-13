#### 数组的类型和元素的类型
引用数据类型
```
int arr [] = {1, 2, 3}; // 静态整形数组
```
中括号的位置，可以在变量前面，也可以在变量后面。
个人感觉明显放在前面更科普呀，读起来更贴切 int [] 就是 int 数组。

数组内元素的序号，称为下标 index，下标从 0 开始。
- 读取 ```int a1 = arr[1];```
- 写入 ```arr[0] = 8;```

相对 OC 来说，Java 的数组声明后都是可变的，可以进行内部元素的替换操作。
此外，数组有一下特征：
- 数组的长度是固定的，一旦声明不可变更；
- 数组内的元素类型必须一致

#### 数组的遍历
利用数组的长度
```Java
int length = arr.length;
for (int i = 0; i < count; ++i) {
      System.out.println(arr[i]);
}
```

#### 动态定义数组
特殊的数组声明方式，创建一个长度为 10 的整形数组。
```int [] arr1 = new int[10];```
基本类型数组，动态定义数组，其内部的值均为该类型的默认值，int/double 为 0， bealean 为 false。
对于引用类型数组，比如 String[]，Test[] 来说，内部默认值则为 null。
```Java
class TestArray {
    public static void main(String[] args) {
        System.out.println(args);
        
        for (int i = 0; i < args.length; i++) {
            System.out.println(args[i]);
        }
        
        String[] arr = new String[5];
        
        for (int i = 0; i < arr.length; i++) {
            System.out.println(arr[i]);
        }
    }
}

```

程序 main 函数也是一个数组，是程序启动的参数数组。

#### 二维数组
数组的元素仍然是数组
```Java
int [][] arr = {{1, 2, 3}, {4, 5, 6}, {7, 8, 9}};
```
类似矩阵的结构，也是当前热门的机器学习的常用数组结构。
```Matrix
1  2  3
4  5  6
7  8  9
```

二维数组的访问，是与一维数组一样的。完全遍历二维数组，需要两层 for 循环。
```Java
for (int i = 0; i < arr.length; i++) {
      int[] innerArr = arr[i];
      int innerLength = innerArr.length;
     for (int j = 0; j < innerLength; j++) {
          System.out.println(innerArr)[j]);
     }
}
```
注意：
- 二维数组内部数组的长度可能不一致
- 访问数组的下标不能超出数组的 index 范围，会出现越界异常 out of bounds exception。


#### 数组的操作
