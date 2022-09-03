> 资源 [廖雪峰 Git 教程](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000)

#### 配置远程仓库
使用 GitHub 创建仓库，在本地配置 ssh 钥匙对
先查看本地是否有 .ssh 文件夹及钥匙对，如果有的话可以复用，否则可使用命令创建：
```
ssh-keygen -t rsa -C "youremail@example.com"
```
创建后将 id_rsa.pub 文件内容复制备份到 GitHub，即可用于从 GitHub 账号将代码提交到 GitHub 的远程仓库。

#### 关联本地仓库到远程，推送修改
本地已有 git 仓库关联到仓库，使用命令：
```
git remote add origin  "git@example.git"
```
第一次将本地代码推送到远程仓库
```
git push -u origin master
```
其中 -u 参数是针对远程仓库为空的情况，Git 会自动关联，以后的推送就不需要此参数。
以后的本地修改，就可以直接通过以下命令进行推送：
```
git push origin master
```

#### 关联远端仓库后的 Git 状态
当关联远端仓库后，使用 ```git status``` 命令可以看到的状态中，包括了与远端仓库的同步情况。

#### 仓库克隆 clone
如果远端已有仓库，如果从零开始在 GitHub 创建了仓库，可以从 GitHub 上克隆一个仓库到本地使用，命令：
```
git clone git@example.git
```

#### Git 的协议
一般有 SSH 或者 HTTPS 等协议，常用的是 SSH。

#### 创建和切换分支
创建分支，例如创建  dev 分支：
```
git branch dev
```
切换分支，例如切换到 dev 分支：
```
git checkout dev
```
查看分支列表，列表中带 * 的即为当前分支:
``` 
git branch
```
dev 分支可以进行照常的开发，不会影响 master 分支的内容，也可以通过 push 命令推送到远端：
```
git push origin dev
```
在保存到 dev 分支的内容后，可以随时切换会 ```git checkout master ``` 分支查看。

#### 合并分支 merge
合并分支是将指定的分支 merge 到当前分支下：
```
git merge dev
```
如果要 merge 到 master 分支，则应该先切换到 master 分支。如果不知道当前分支是什么，随时通过 ```git branch``` 查看。
合并完成后，也可以愉快地随时 ```git push origin master``` 到远端了。

#### 鼓励使用分支
因为分支开发更为安全，且 merge 操作的性能也很好。

#### 删除分支
分支完成其使命后可以考虑删除掉：
```
git branch -d dev
```
删除后通过 ```git branch``` 查看结果。
接着要删除远端分支：
```
git push origin -d dev
```

#### 解决冲突
将分支 merge 后出现冲突时会有提醒，此时需要手动解决冲突
- 找到 <<<< ==== >>>> 分隔符的内容，进行冲突情况下的内容修改
- 将修改进行 add commit 提交到分支


#### 分支开发策略
- master 分支是最稳定的，用于发布产品版本
- dev 分支用于新功能的开发和迭代
- 每个具体的人在自己的业务分支中开发，完成一定程度后 merge 到 dev 分支

#### Bug/hotfix 分支
针对 bug 分支，可以在 master 分支上 新增一个分支，
```git checkout master```
``` git branch issue001```
``` git checkout issue001```
bug 修复完成后 提交并 merge 到 master 即可

#### feature 分支
如果要开发一个新的功能，可以新建一个 feature 分支
``` git checkout -b feature-some```
如果该分支最后未合并，但是需要舍弃，那么使用 ```git branch -d feature-some``` 进行删除时会有错误
未合并的分支，使用 ```git branch -D feature-some``` 进行删除。

#### 备份工作区
在修改 bug 之前如果 当前分支 ，比如是 dev 分支尚未完成开发，不可以提交， 那么可以使用
```git stash ``` 命令将当前工作内容暂时 储藏到当前 分支
等到需要继续工作到，切换到当前分支 使用 ```git stash pop``` 恢复原来的工作状态
stash 命令可以多次调用，可以使用 ```git stash list``` 命令查看当前储藏列表

> 学习就是要多练习

#### 参考文章
[zengrong - Git查看、删除、重命名远程分支和tag](https://blog.zengrong.net/post/1746.html)
