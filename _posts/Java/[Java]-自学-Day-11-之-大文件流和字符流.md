#### 大文件

文件输入输出流内部都做好了索引的处理，每次读写，都进行了 index 的步进，这样可以保证流的连续读入和连续写入。

举例，用字节流读写大文件，注意：
- 流的记录是递进的，可以连续读写
- while 循环适时 break
- 适时关闭流，确保各个 try-catch 之间不会影响下一步的 try-catch 操作

```Java
import java.io.*;

class BigFile {
    public static void main(String[] args){
        byte[] buffer = new byte[1024];
        
        
        System.out.println("start.");
        
        FileInputStream input = null;
        FileOutputStream output = null;
        
        try {
            input = new FileInputStream("from.txt");
            output = new FileOutputStream("to.txt");
            
            while (true) {
                int get = input.read(buffer, 0, buffer.length);
                System.out.println("Get -> " + get);
                if (get <= 0) {
                    System.out.println("done.");
                    break;
                }
                output.write(buffer, 0, get);
            }
        } catch (Exception e) {
            System.out.println(e);
        } finally {
              try {
                       input.close(); // 记得关闭输入流
              } catch (Exception inputException){
                      // 捕获到关闭输入流的异常
               try {
                      // 记得关闭输出流，不能和关闭输入流在一个 try 内
                      // 避免关闭输入流异常而导致输入流关闭函数无法得到调用
                      output.close(); 
                } catch (Exception outputException) {
                        // 捕获到关闭输出流的异常
                }
        }

    }
}

```


#### I/O 流的关闭
流的构造和关闭方法都会产生异常，因此调用时都是需要进行 try-catch 操作

**Java 是不能向空指针发送消息的，这一点和 OC 是完全不同的，因此需要注意这方面的使用。**

因为可能在业务的执行过程前创建的流，过程中抛出了一些异常，所以可能无法及时关闭流，因此，合适的关闭流的位置是 try-catch 的 finally 部分，即便 close() 本身还需要进行 try-catch，而不能图方便放到创建流的代码块内，否则第一个 try-catch 抛出异常后，可能会导致 close() 函数没有机会调用。

```Java
FileInputStream input = null;
try {  
      input = new FileInputStream("SomeFile.ext"); // 创建流对象
}
catch (Exception newException) {
        // 捕获创建流的异常
} finally {
      try {
            input.close(); // 确保关闭流方法最终都会执行
      } catch ( Exception closeException) {
           // 捕获关闭流的异常
      }
}
```

#### 字符流
在读写文件时，以字符为基础，基础抽象类分为：
- 字符输入流 Reader
```  int read(char[] c, int off, int length); ```

- 字符输出流 Writer
```void write(char[] c, int off, int length);```

实现的子类对应的 FileReader、FileWriter

** 对比字节流和字符流，可以看到流的思想有很大的共通性。 **

举例，用字符流读写大文件：

```Java
import java.io.*;

public class FileChar {
    public static void main(String[] args) {
        FileReader reader = null;
        FileWriter writer = null;
        
        System.out.println("Start.");
        
        try {
            reader = new FileReader("from.txt");
            writer = new FileWriter("to.txt");
            
            char[] buffer = new char[1024];
            // buffer 数组 中的空元素是空字符
            
            while (true) {
                int got = reader.read(buffer, 0, buffer.length);
                System.out.println(got);
                if (got <= 0) {
                    System.out.println("done.");
                }
                writer.write(buffer, 0, got);
            }
            
        } catch (Exception e) {
            System.out.println(e);
            
        } finally {
            try {
                reader.close();
            } catch (Exception e) {
                System.out.println(e);
            }
            
            try {
                writer.close();
            } catch (Exception e) {
                System.out.println(e);
            }
        }

    }
}
```

> 接下来是装饰器设计模式，节点流和处理流的学习。
