#### Collection 接口
定义了集合 Collection 的常用函数声明：
add /  remove / clear / isEmpty / size 等函数

#### Set 接口
- 继承自 Collection 接口
- 实现类 HashSet
- 是泛型类型

```Java
import java.util.Set;
import java.util.HashSet;

class TestSet {
    public static void main(String[] args) {
        HashSet<String> hash1 = new HashSet<String>();
        
        Set<String> set = hash1; // 可以省略的向上转型
        set.add("a");
        set.add("b");
        set.add("c");
        set.add("d");
        set.add("d"); // 被忽略，不能有重复元素
        
        System.out.println(set.size());
    }
}
```

#### 遍历 Set
HashSet -> Set -> Collection -> Iterator 继承关系

Iterator 接口
```Java
boolean hasNext()
Object next()
```
- 调用 Set 的 iterator 函数生成一个 Iterator 对象。
- Iterator 调用 next() 函数返回下一个对象，并使得索引 +1 。
- 迭代器同样是泛型类型
```Java
import java.util.Set;
import java.util.HashSet;
import java.util.Iterator;

class TestSet {
    public static void main(String[] args) {
        HashSet<String> hash1 = new HashSet<String>();
        
        Set<String> set = hash1; // 可以省略的向上转型
        set.add("a");
        set.add("b");
        set.add("c");
        set.add("d");
        set.add("d"); // 被忽略，不能有重复元素
        
        Iterator<String> it1 = set.iterator(); 
        System.out.println(it1);
        
        while (it1.hasNext()) {
            System.out.println(it1.next());
        }
    }
}
```

#### 迭代器模式
在不暴露集合细节情况下，遍历内部元素。
