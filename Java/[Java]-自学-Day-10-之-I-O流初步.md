#### 输入/输出流

以 Java 程序作为参照，分为输入流和输出流。
不要局限于输入/输出的方式，来源和去向是可以多样化的。
流，是体现在数据在管道中传递，即 steam 的形态，不是点状的。

#### 分类

字节流、字符流
字节流抽象类 InputStream/OutputStream
文件读取业务类 FileInputStream/FileOutputStream

#### 读写函数

```int read(byte[] b, int off, int len);```
- 从待存储的数据缓存 b 中，
- 从 buffer 的 off 位置开始
- 从读取流的记录的最新的一个位置读取 len 长度的数据存入到 buffer 中
- 返回最终读取到的数据长度。

```int write(byte[] b, int off, int len);```
- 从准备好的数据缓存 buffer 中
- 从 buffer 的 off 位置开始
- 写入 len 长度的数据到输出流中的末尾处。

```Java
import java.io.*;

class Test {
    public static void main(String[] args) {
        FileInputStream f1 = null;
        FileOutputStream out1 = null;
        try {
            // 缓存 buffer
            byte[] buffer = new byte[100];
            
            // 读取
            f1 = new FileInputStream("from.text");  // 读取流实例
            int get = f1.read(buffer, 0, buffer.length); // 从流的最后位置开始读入
            System.out.println("get " + get);
         
            // 查看数据
            for (int i = 0; i < buffer.length; i++) {
                System.out.println(buffer[i]);
            }
            String s1 = new String(buffer); // 字节数组转字符串
            s1.trim();// 移除字符串的收尾空字符和空格
            System.out.println(s1);
            
            // 写入
            out1 = new FileOutputStream("to.text"); // 写入流实例
            out1.write(buffer, 0, get); // 传入 get 的长度 // 从流最后位置开始写入
            
        } catch (Exception e) {
            System.out.println(e);
        } finally {
              try {
                       f1.close(); // 记得关闭输入流
              } catch (Exception inputException){
                      // 捕获到关闭输入流的异常
               try {
                      // 记得关闭输出流，不能和关闭输入流在一个 try 内
                      // 避免关闭输入流异常而导致输入流关闭函数无法得到调用
                      out1.close(); 
                } catch (Exception outputException) {
                        // 捕获到关闭输出流的异常
                }
        }
    }
}
```
