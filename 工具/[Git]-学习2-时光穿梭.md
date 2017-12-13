> 资源 [廖雪峰 Git 教程](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000)

#### git 管理的是修改
每次 add / commit 记录的都是修改的情况

#### 回退 or 前进
```git checkout -- <filename>``` 撤销某个文件在工作区的修改，丢失所有的修改
如果文件已经提交到暂存区，使用 ```git reset HEAD <filename>``` 来撤销在暂存区stage的提交，即 unstage 操作，工作区的状态恢复到 add 之前的状态
HEAD 表示本地当前工作的版本，每次 commit 都会生成一个 id，是 SHA1 值：
```git reset --hard HEAD^``` 一个 ^ 括号代表回退几个版本
```git reset --hard 2rjlajf``` 回退/前进到特定 id 的一次 commit 记录，如果多了可以使用 ```git reset --hard HEAD~100```回退一百次以前；
```git reflog``` 如果忘记了历史记录，通过 reflog (reference log) 命令可以查看历史的命令记录从而可以在 Git 回到较新的记录；

#### 工作区、版本库和暂存区
- 工作区 working directory 创建 git init 的文件夹
- 版本库 repository 在工作区内的一个隐藏文件夹 .git

版本库内有很多内容，主要的是暂存区、分支及其 HEAD 指针，
- 调用 git add 命令，现将文件保存到暂存区；
- 调用 git commit 将暂存区的文件都保存到当前分支中并清空暂存区；

#### 管理修改
Git 管理的是修改，而不是文件，因此没有 add 到暂存区的修改，是不会被 commit 到分支的

#### 撤销修改
①撤销工作区文件的修改
```git checkout -- some.file``` 撤销对工作区 some.file 的修改，注意```--``` 特别重要，会将文件恢复到上一次 git add / commit 之前的状态：
- 上一次还没有 add 到暂存区，则恢复到与版本库当前分支相同
- 上一次已 add 到暂存区，则恢复到与暂存区相同
②撤销暂存区文件的修改
```git reset HEAD some.file``` 撤销对暂存区 some.file 的修改返回到工作区，如果希望进一步撤销工作区的修改，则执行上一条 checkout 命令
③撤销版本库分支的修改
这个是之前提及的回退功能，前提是还没有推送到远程仓库。

#### 辅助命令
```cat some.file``` 直接打印文件信息 concatenate
```rm some.file``` 从文件管理器中删除文件 remove

#### 删除文件
当文件从工作区删除后：
- ```git rm some.file``` 从暂存区中删除文件，然后就可以 commit 到分支了
```git checkout -- some.file``` 撤销文件的删除，恢复到暂存区的状态（不是删除前的状态，因为删除前的修改 没有 add 到暂存区）

#### 重命名文件
如果对文件了重名，那么就相当于 先 remove 旧的，再 add 新的。

> 对比学生时代，现在学习时做的练习真是太少了。
