#### 服务器端多线程将更为复杂
比起移动端来说，更具有挑战。

#### 多进程和多线程
- 多进程：操作系统可以进行多个任务（程序），早期的 iOS 是单进程程序（表面上）
- 多线程，在同一个程序内有多个顺序流（“同时”）执行

粗暴简单的理解：CPU 在快速切换，“看起来”，任务在“同时”执行。比如，Android 是多进程系统，原则上一个程序一个进程；
Android 的 UI 线程，与 iOS 类似，都是主线程，其他任务在子线程处理。

![多线程抢占 CPU 时间](http://upload-images.jianshu.io/upload_images/73339-44fcfd88c7cc81ff.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 线程的执行过程（生命周期）

这部分知识是通用的，生成 -> 就绪 <-> 运行（可能阻塞）-> 结束，因此线程对应了上述几个状态。

![线程执行流程](http://upload-images.jianshu.io/upload_images/73339-1c1da52842843c01.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

现有个直观了解，再上代码学习。

#### Java 中线程的实现：方法一
线程是一个线程对象，Thread 类，创建线程的方法：
- 继承 Thread 类，复写 run() 函数，其函数体称为“线程提及”

- **不要调用 run 函数**，因为那是在当前线程中执行 run() 内容了
- 在线程中执行 run 的线程体，线程体并不一定会完全一次性执行结束




![多线程运行结果](http://upload-images.jianshu.io/upload_images/73339-2319146168589a0e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```Java
class FirstThread extends Thread {
    public void run() {
        System.out.println("start");
        
        for (int i = 0; i < 100; i++) {
            System.out.println("Sub  -> " + i);
        }
        
        System.out.println("end");
    }
    
    public static void main(String[] args) {
        // 1、生成
        FirstThread t1 = new FirstThread();
        
        // 2、启动（就绪）
        t1.start();
        
        for (int i = 0; i < 100; i++) {
            System.out.println("main ->" + i);
        }
    }
}
```

苹果开发者一看，这个和  NSOperation 有些共通性，需要在内部实现 main 方法，在队列 NSOperationQueue 中由系统管理执行。

随着多线程的学习深入，可以更多地了解线程优先级、QOS 等新概念。

#### 在 Java 程序启动后，有三个线程
- 主函数线程
- 新建的子线程
- 垃圾回收线程


#### Java 中线程实现：方法二
实现 Runnable 接口，实现接口中的 run() 函数，成为 Thread 的目标对象，目标对象的函数体，也就是线程体。
没错，从该方案的描述可见，Thread 将线程体交出由外部实现，这就是线程的委托模式了。

代码示范:
```Java
class MyRun implements Runnable {
    public void run() {
        for (int i = 0; i < 100; i++) {
            System.out.println("MyRun -> " + i);
        }
    }
    
    public static void main(String[] args) {
        // 1、生成 MyRun 对象
        MyRun run1 = new MyRun();
        
        // 2、将 MyRun 对象作为参数传入 Thread 构造线程实例
        Thread t1 = new Thread(run1);
        // 3、通知 t1 执行 start 方法
        t1.start();
        
        for (int i = 0; i < 100; i ++) {
            System.out.println("Main  -> " + i);
        }
    }
}
```

执行结果，可以得出与方法一相似，即多线程运行，是同时并发。

![方法二结果](http://upload-images.jianshu.io/upload_images/73339-e8fbc30d9f062903.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 两种线程实现的对比
怎么 Java 出身的同学，都不爱继承呢，我个人还是觉得看情况，继承往往是最后的选项了。

- 就线程的实现来看，通过接口来实现，更可以隔离耦合，很方便，推荐！
- 继承的话，可以封装特定需求任务的线程。

#### 线程的简单控制方法
- 中断线程 
```Thead.sleep(2000) ;``` 
// 休眠 2000 毫秒，休眠完成后，**进入的是就绪状态不是运行**
```Thread.yield();```
// 让出当前运行机会，进入下一轮争夺，仍旧是就绪状态，有可能仍旧执行有可能不会。

- 设置优先级
```getPriority();```
```setPriority();```
默认优先级是 5 ，是静态常量，最大优先级 Thread.MAX_PRIORITY（10） 最小 MIN_PRIORITY（1）

优先级，决定了线程执行的**概率**，优先级高概率（频率）大。


线程控制方法可能抛出异常，需要 try-catch

#### 多线程下的数据安全
资源争夺： 多线程共用同一份数据，典型例子是卖票。

```Java

class XRun implements Runnable {
    int i = 100;
    public void run() {
        while (true) {
            String currentName = Thread.currentThread().getName();
            System.out.println(currentName + i);
            i --;
            
            Thread.yield();
            if (i < 0) {
                String currentName1 = Thread.currentThread().getName();
                System.out.println(currentName1 + "Done.");
                break;
            } else {
                String currentName2 = Thread.currentThread().getName();
                System.out.println(currentName2 + "continued");
            }
        }
    }
    
    public static void main(String[] args) {
        XRun run = new XRun();
        
        Thread t1 = new Thread(run);
        Thread t2 = new Thread(run);
        
        t1.setName("T1 -> ");
        t2.setName("T2 -> ");
        
        t1.start();
        t2.start();
    }
}
```
#### 同步代码块
与 OC 类似，将资源争夺的操作使用同步代码块加锁。关键字 ``` synchronized(token) {} ```
```Java
 // 将 while 语句的代码在锁内执行
// 同步代码块内的参数是锁的 token
synchronized (this) {
      
}
```

同步代码块的应用，类似是一个函数，如果 token 是同一个，那么其他线程的因为无法获取到 token 而无法进行。举例如下面两个函数如果同时被调用，那么会在获取同步锁 token 后才能得以执行。

```Java
public void func1 {
      synchronized(this) {
         // 被 this 作为 token 锁住的代码
      }
}

public void func2() {
           synchronized(this) {
         // 被 this 作为 token 锁住的代码
      }
}

```

#### 同步方法
同步方法类似于同步代码块，锁的 token 是即 this 对象，一下同步方法与上述代码等效。相对而言，同步代码块使用范围更广，同步方法是同步代码块的子集。
```Java
public synchronized void func1() {
}

public synchronized void func2() {
}

```

#### 锁的理解
被锁锁住的是一种行为（一段代码逻辑），比如前面 run() 理解为卖票，那么卖票这个行为是不能同时进行的。如果有其他的不可并行的操作，也需要加锁。
