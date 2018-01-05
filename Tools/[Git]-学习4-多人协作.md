> 资源 [廖雪峰 Git 教程](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000)


、#### 查看远程信息
``` git remote``` 查看名称
``` git remote -v``` 查看远程详情，如果有 push 权限则可以看到 push 的地址

#### 推送分支
```git push origin master```
```git push origin dev```

#### 推送哪些分支
```master ``` 分支要保持同步
```dev``` 分支要保持同步
```bug``` 分支是本地修改 bug 用，如果是独立修改，可以不 push 到远端
```feature``` 分支，也取决于是否与同事协作开发，决定是否 push 到远端

#### 抓取分支
在本地创建对应远程分支的分支，```git checkout -b dev origin/dev```
``` git pull``` 将远程代码 拉取到本地
```git --set--upstream origin/dev``` 将本地分支与远程分支进行关联

#### 常用合作流程
- 首先，可以试图用git push origin branch-name推送自己的修改；
- 如果推送失败，则因为远程分支比你的本地更新，需要先用git pull试图合并；
- 如果合并有冲突，则解决冲突，并在本地提交；
- 没有冲突或者解决掉冲突后，再用git push origin branch-name推送就能成功！
- 如果git pull提示“no tracking information”，则说明本地分支和远程分支的链接关系没有创建，用命令git branch --set-upstream branch-name origin/branch-name。
