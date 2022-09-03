> 资源 [廖雪峰 Git 教程](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000)

#### 创建版本库

在制定路径下
```git init``` 会在此目录下生成 git 管理相关文件，是隐藏文件
一些辅助命令：
```pwd``` 打印当前路径名称，用于表达：print working directory；
``` mkdir  ``` 创建文件夹，用于表达：make directory；
```ls```，打印当前路径下的文件列表，表示：list
```ls -ah```，打印当前路径下的包括隐藏文件的文件列表，表示：list and hidden

#### 简单的 vim 命令
创建/打开文件
```vim some.file```
打开文件后可以移动光标，或者输入一些字母快速跳动，当输入 ```I``` 后进入编辑状态；
这样就可以在命令行编辑文件，编辑完成后，点击 **ESC** 按钮退出编辑，选择输入不同命令进行退出：
```:wq```   写入保存并退出；
```:cq```   关闭（不保存）并退出；

#### 提交文件
```git add some.file``` 将文件的修改添加到待提交列表，可以多次调用；
```git add .``` 使用 `.` 一次性将当前路径下的修改都 add 到暂存区
```git commit -m "some log about this commit "```  提交之前 add 过的修改， -m 参数后面的字符串是记录这一次提交的内容；

#### 查看状态
```git status``` 查看当前仓库的状态，可以看到仓库整体的修改情况；
```git diff some.file``` 查看特定文件的变化情况，会显示新增/删除了那些问本行变化；

#### 配置用户信息
```git config --global user.name "Brook"```
```git config --global user.email "example@abc.com"```


#### 查看历史日志
```git log```  打印详细的日志
```git log --pretty=oneline``` 打印简单的日志到一行

#### 小结
学新东西总有个走出舒适区的过程，不要给自己压力，先来一点点。

> 感谢 Luffy 在 vim 使用上的指导。
