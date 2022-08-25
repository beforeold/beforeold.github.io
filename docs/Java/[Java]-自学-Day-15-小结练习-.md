#### 编程还是要靠敲
看书，看视频，是不足的。

#### 练习需求
- 接收用户的命令行输入
- 以文件为基础完成数据的增删改查操作

#### 模块划分
典型的模块划分
- UI 模块
  - 帮助模块
  - 输入接收模块
  - 结果输出模块
- 文件 IO 模块
  -  数据插入模块
  - 数据查询模块
- 数据验证模块（必须重视）
- 逻辑模块（调用业务逻辑）

#### UI 模块相关技术

![命令行输入/输出](http://upload-images.jianshu.io/upload_images/73339-d2e339e25c53a53a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 文件 IO 模块相关技术


![image.png](http://upload-images.jianshu.io/upload_images/73339-b4ef60261c96993c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


关于 RandomAccessFile，就是可以自由地查看和写入文件。

#### 一个类的成员变量应该尽量 private
可以提供 setter / getter
是对成员变量的封装

#### 提示信息的管理
- 可能变化和维护
- 需要国际化
资源文件的使用，基于键值对
获取项目的路径
获取文件的路径，项目路径 + 分隔符 + 文件夹 + 文件名
```
String path = System.getProperty("uer.dir');
String filePath = path + File.separator + "resources" + File.separator + "first.properties";
```

资源文件，是基于文件输入流，算是组合模式的应用。

#### 主循环的设计
runloop 来了
![程序循环流程](http://upload-images.jianshu.io/upload_images/73339-c478483721f99cc3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 下一步，做一些 Java 练手项目


#### 高级内容以后再跟进
泛型、泛型、设计模式

#### 学习资料来源：
Mars 老师的 [Java4android](http://study.163.com/course/courseMain.htm?courseId=201001)
