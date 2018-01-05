
> 资源 [廖雪峰 Git 教程](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000)


#### 标签是具体某一次 commit 的别名
表现相对 commit 的指针更便于理解和查找，一般是用于标记产品的版本

#### 创建 tag
切换到特定的分支
```git tag  v1.0``` 在当前 commit 创建 tag
```git tag v1.0 commit-id``` 以某次 commit 作为 tag
```git tag``` 查看 tag 列表
```git show v1.0``` 查看 tag 的信息

#### 添加 tag 时添加说明
```git tag -a v1.0 -m "tag desc"``` -a 对应 tag 名称， -m 对应说明

#### 为 tag 添加 gpg 签名
```git tag -s v1.0 -m "signed version released"```
-s 对应需要签名的 tag 名称， -m 对应 tag 的备注说明


#### 操作标签
```git tag -d v1.0``` 删除 tag 
```git push origin v1.0``` 将 tag 推送到远端
```git push origin --tags``` 将所有 tag 推送到远端
感觉 push tag 的速度比较慢
这样就可以在远端仓库查看到 tag 了
![](http://upload-images.jianshu.io/upload_images/73339-6cd73b5a9cfbe994.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 删除远端 tag
① 先删除本地 tag ···git tag -d v0.9```
② 推送到远端 ```git push origin :refs/tags/v0.9```
之后即可查看远端是否已移除此 tag。

