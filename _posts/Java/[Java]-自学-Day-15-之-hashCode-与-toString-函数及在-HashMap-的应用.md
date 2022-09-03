#### 同样均为 Object 函数
同样的，父类有默认实现，由子类进行定制。

#### Hash 算法
- 任意长度数据 ==> 固定长度散列值
- 输入和输出值的映射是唯一的，且每次计算结果一致，应尽量避免重装
- 用于加密、签名等。


![Hash 算法](http://upload-images.jianshu.io/upload_images/73339-2ad67b9a8406830a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果将散列值与地址关联起来，那么可以用于数据的查询。

#### hashCode() 函数
方法产生一个对象的哈希码，整型值。
根据 Java 的要求，如果两个对象调用 equals 方法相等，那么两者的 hashCode 也应该相等。

#### hashCode 函数在类集框架中有广泛的应用
要保证两个对象可以作为 HashMap 的 key 达到可以等效，那么需要满足两个条件
- equals 返回 true
- hashCode() 值相等
```Java
import java.util.HashMap;

class User {
	int age;
	String name;

	public int hashCode(){
		int result = 17;
		resutl = 31 * result + age;
		result = 31 * result + name == null ? 0 + name.hashCode();

		return result;
	}

	public boolean equals(Object o) {
		return true;
	}

	User(int age, String name) {
		this.age = age;
		this.name = name;
	}

	public static void main(String[] args) {
		// 根据 key 的 hashCode 匹配

		User u1 = new User(12, "Brook");

		HashMap<User, String> map = new HashMap<User, String>();
		map.put(u1, "Ret");

		String ret = map.get(new User(12, "Brook"));
		System.out.println(ret);

		HashMap<String, String> map2 = new HashMap<String, String>();
		map2.put("Hello", "World");

		String ret2 = map2.get("Hello");
		System.out.println(ret2);
	}
}
```
同样地， hasCode 函数自身一般不调用，而在集合的应用中被集合调用。回头，可以学习类集框架的源代码。

#### toString() 函数
通过复写 toString 函数为对象提供个性化的描述。类似 OC 的 description 方法。
将对象转成字符串描述，这样在系统打印时，会调用对象的 toString 函数。
```Java
class Student {
	int age;
	String name;

	public String toString() {
		return "Student: " + "age " + age + " name " + name;
	}

	public static void main(String[] args) {
		Student s1 = new Student();
		s1.age = 18;
		s1.name = "Brook";

		System.out.println(s1);
	}
}
```

#### 延伸
- 联想一下 OC 的 NSObject 的类似实现：
``` isEqual: ``` 方法
``` hash ``` 方法

- 联想 Swift 中的类似实现
``` Hashable``` 协议
``` Equatable ``` 协议
