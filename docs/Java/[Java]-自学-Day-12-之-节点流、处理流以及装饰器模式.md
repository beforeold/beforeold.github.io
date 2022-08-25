#### 处理流和节点流
字符输入处理流 BufferedFileReader
从 [Oracle](http://docs.oracle.com/javase/7/docs/api/java/io/BufferedReader.html) 查看 此类以装饰器的方式的构造器声明：
**[BufferedReader](http://docs.oracle.com/javase/7/docs/api/java/io/BufferedReader.html#BufferedReader(java.io.Reader))**([Reader](http://docs.oracle.com/javase/7/docs/api/java/io/Reader.html) in)
Creates a buffering character-input stream that uses a default-sized input buffer.

**[BufferedReader](http://docs.oracle.com/javase/7/docs/api/java/io/BufferedReader.html#BufferedReader(java.io.Reader,%20int))**([Reader](http://docs.oracle.com/javase/7/docs/api/java/io/Reader.html) in, int sz)
Creates a buffering character-input stream that uses an input buffer of the specified size.

使用 BufferedReader 的常用目的是利用其 readLine() 函数从流中读出一行字符，而其本身不直接读取文件流，而是通过装饰一个字符流来实现，举例如下：

```Java
import java.io.*;

public class Test {
    public static void main(String[] args) {
        FileReader in = null;
        BufferedReader reader = null;
        
        try {
            in = new FileReader("file.txt");
            reader = new BufferedReader(in);
            
            while (true) {
                String line = reader.readLine();
                if (line == null) {
                    System.out.println("done.");
                    break;
                }
                System.out.println(line);
            }
            
        } catch (Exception e) {
            System.out.println(e);
            
        } finally  {
            try {
                in.close();
            } catch (Exception e) {
                System.out.println(e);
            }
            
            try {
                reader.close();
            } catch (Exception e) {
                System.out.println(e);
            }
        }
        
    }
}
```

在 BufferedReader 这个案例中，内部被装饰的 FileReader 是这个场景下的**节点流**，而 BufferedReader 作为装饰器，称为**处理流**。
此外，在 I/O 系统中还有压缩流，解压缩流等，是类似的设计模式 -> 装饰器模式。

#### 装饰器模式 Decorator  的实现
装饰器模式， 是 **Java 的I/O 系统中常用设计模式**
这装饰器的应用，与组合模式有些相似，不过不用去纠结名称。是一种面向对象封装的思路，比起继承来说，可以将一类对象A1, A2, A3 三个类，进一步封装到一个类 B 中，从而使得 B 可以作为 A1/A2/A3 的容器，在 A的方法执行过程进行装饰性的封装了。
拿装修举例，用接口 Worker 表述工人的功能：工作 work() 函数，分别有水管工Plumber 和木匠 Carpenter都实现了 work()。针对性的 A公司在工作前需要进行打印“你好”，而B公司在工作前需要打印“穿鞋”，这种情况，通过为 A、B 公司各自新建一个装饰器类 AWorker、BWorker 即可，而不是需要在为此生成 APlumber、BPlumber、ACarpenter、BCarpenter 等等。
实现如下：
- 声明工人的接口
```Java
public interface Worker {
    public void work();
}
```
- 实现水管工和木匠类
```Java
public class Plumber implements Worker {
    public void work() {
        System.out.println("水管工");
    }
}

public class Carpenter implements Worker {
    public void work() {
        System.out.println("木工干活");
    }
}

```
- 为A公司定义工人的装饰器
```Java
public class AWorker implements Worker {
    private Worker worker;
    
    public AWorker(Worker worker) {
        this.worker = worker;
    }
    
    public void work() {
        System.out.println("Hello");
        worker.work();
    }
}

```

- 调用测试装饰器的效果
```Java
public class Test {
    public static void main(String[] args) {
        Plumber p1 = new Plumber();
        AWorker a1 = new AWorker(p1);
        a1.work();
        
        Carpenter c1 = new Carpenter();
        AWorker a2 = new AWorker(c1);
        a2.work();
    }
}

```
打印出：
```Java
Hello
水管工
Hello
木工干活
```

与组合模式的区别：
- 装饰器模式，装饰器 B，显式地纳入被封装的对象 A（被装饰者），在 A 的功能基础上进行修饰处理（比如在被装饰者执行方法前后插入其他的逻辑）
- 组合模式，组合对象 Y，可以显式地或者隐式地纳入被封装对象 X，将被组合者 X 的功能作为 Y 的一部分，并进一步实现更多的功能，常用于完全包含的封装或者将业务逻辑进行拆分。

联想一下 iOS 中，经典的案例
- 装饰器模式 AOP method-swizzling，
- 组合模式， UIView 与 CALayer 的关系


#### 键盘输入流等其他节点流的处理流实现思路
使用 BufferedReader 装饰键盘输入，添加新的功能
核心是针对键盘输入流装饰后的 readLine() 等函数的实现
