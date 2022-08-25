#### Map 是单独的接口
键值对存储，键不可重复，重复的键将被新的值覆盖。
泛型类型 <Key, Value>


#### 使用帮助文档
JDK Documentation

#### 访问 Map
``` put(key, value)```
// 放入键值对，如果 key 之前存在，旧值将被覆盖
``` get(key)```
// 取值

```Java
import java.util.Map;
import java.util.HashMap;


public class TestMap {
    public static void main(String[] args) {
        HashMap<String, String> m1 = new HashMap<String, String>();
        
        
        Map<String, String> map = m1; // 不是必须的向上转型
        
        map.put("name", "Brook");
        map.put("age", "28");
        map.put("age", "14"); // 将覆盖之前的 28
        
        System.out.println(map.size());

      System.out.println(map.get("age"));
    }
}
```
