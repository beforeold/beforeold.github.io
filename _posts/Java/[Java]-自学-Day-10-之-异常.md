#### 异常和错误
Java 的程序运行异常分为 Error 和 Exception，都继承自 Throwable类（Swift 玩多了还以为这是接口）。Error 抛出时无法 catch 程序直接崩溃，Exception 可以通过 try-catch 进行捕获。

#### finally 代码块
try-catch-finally 在 finally 代码块可以做一些清理资源和收尾工作，虽然可以在 try-catch-finally 外部写，但是这样写语义更清晰。

#### 异常的分类
异常均继承自 Exception 类，Java 中函数可能存在的异常在编译层面分为 check 和 uncheck两类，即一种必须进行 try-catch 才能通过，一种则可以忽略。

- 子类 RuntimeException 及其子类均为 uncheck 异常
```Java
int a = 1 / 0;
```
- 其他 Exception 及其子类为 check 异常，使用这些函数时必须进行 try-catch。
```Java
Thread.sleep(1000);
```

check exception 必须在函数内部 try-catch 处理或者声明 throws，而对于 uncheck exception 可以强制要求内部处理或者显示声明 throws。

#### 声明 throws 关键字
如果在定义函数时可能抛出异常，那么这个异常要么在函数内部进行 catch，如果不在内部 catch 则可以抛出给调用者处理，此时使用 throws 关键字在函数的尾部：
```Java
void setAge(int age) throws Exception {
         if (age < 0) {
            Exception e = new Exception("年龄不能小于0");
            throw e;
        }
       
        this.age = age;
}
```

使用 throws 关键字后，外部调用此函数时，如不进行 try-catch 会收到编译错误。
