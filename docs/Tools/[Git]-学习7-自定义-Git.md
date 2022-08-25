> 资源 [廖雪峰 Git 教程](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000)


#### 让 Git 在终端上显示特定的颜色
```git config --global color.ui = true``` 就像 设置 user.name user.email 那样，可以配置 git 特定的颜色延时，如果只用第三方终端，比如 [iTerm2](https://iterm2.com) 则不需要了。

#### 忽略特殊文件
很多文件是不需要通过 git 来管理的，
>忽略操作系统自动生成的文件，比如缩略图等；
忽略编译生成的中间文件、可执行文件等，也就是如果一个文件是通过另一个文件自动生成的，那自动生成的文件就没必要放进版本库，比如Java编译产生的.class文件；
忽略你自己的带有敏感信息的配置文件，比如存放口令的配置文件。

可以将其放到一个新建 .gitignore 文件中，这样 git 就不再追踪这些文件的变化，.gitignore 可以参考 GitHub 配置好的[一些模板](https://github.com/github/gitignore)

编辑好以后，将 .gitignore 文件提交到 git 即可。

如果要在 gitignore 的情况下强行 force 追踪某个文件，可以使用 -f 参数添加到暂存区：
``` git add -f xx.file ``` 
如果想知道某个文件为什么会被忽略，可以使用一下命令查看：
```git check-ignore file.name```

.gitignore 本身要可以添加到版本库中做管理

#### 为命令添加别名
- 比如将容易输错的 status 修改为 now
```git config --global alias.now status```

- 容易输错的```checkout``` 修改为 ```co```
```git config --global alias.co checkout```
```--global ``` 是全局参数，在当前用户的所有 Git 仓库都有效。

- 再比如，将某个提交到暂存区的文件撤销的命令
```git reset HEAD file.name```中的 reset HEAD 修改为 unstage
```git config --global alias.unstage 'reset HEAD'``` 两个单词要用引号
如此可以使用 unstage 命令来撤销文件的修改到工作区中
```git unstage file.name```

- 使用 last 代替 'log -1'
```git last``` 即可查看最近的一次 commit 的 log

- 或者定义一个很方便的 log 命令 lg 如下
```git config --global alias.lg "log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit"```
可以打印出漂亮详尽的 log
![自定义 lg 命令](http://upload-images.jianshu.io/upload_images/73339-fa9e2f624dbc7057.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

####  配置文件路径
每个仓库的Git配置文件都放在.git/config文件中
而当前用户的Git配置文件放在用户主目录下的一个隐藏文件.gitconfig中

如果命令的别名配置不恰当可以到对应的文件中进行删除后再重新设置即可。
