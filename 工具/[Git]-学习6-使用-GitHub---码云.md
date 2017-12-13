> 资源 [廖雪峰 Git 教程](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000)


#### 参与开源
找到感兴趣的开源项目仓库，点击右上角的 fork 按钮，将该仓库 fork 分叉到自己的账号下。
![Fork](http://upload-images.jianshu.io/upload_images/73339-0259ebe962a8bf6a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 在 fork 仓库开发
fork 到自己的仓库了，就可以按照常规的策略进行开发，并可以推送到远端

#### 创建 Push Request
如果想要将自己 fork 下开发的代码 push 到官方仓库，则需要新建一个 Push Request，点击分支旁边的 New Push Request 按钮：
![New Push Request](http://upload-images.jianshu.io/upload_images/73339-58fb8a7379889c41.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如此可以看到 fork 下历史提交：
![新建 Push Request](http://upload-images.jianshu.io/upload_images/73339-0f1d7d2718f11e0a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
点击 create pull request 可以发起一次请求，添加 title 和 comment 即提交到官方仓库的维护者了。
这样就可以从官方仓库看到提交的 push request 了，比如，[Push Request 2537](https://github.com/michaelliao/learngit/pull/2537) 。
![](http://upload-images.jianshu.io/upload_images/73339-993c8744e8753b96.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 关联多个远程仓库
``` git remote``` 查看当前关联的远程仓库
```git remote -v``` 查看当前关联的远程仓库的详情
``` git remote add github git@dfssf``` 添加一个新的远程仓库

#### 使用码云 [Gitee](www.giteee.com)
码云提供免费的公开仓库，也提供免费的私有仓库，小团队前期可以尝试使用做多人协作。
使用方法是与 GitHub 是一致的。

![关联多个远程仓库](http://upload-images.jianshu.io/upload_images/73339-a3169cea31f99b07.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




