---
layout:       post
title:        "如何解决 CocoaPods could not find compatible versions for pod RxSwift？"
author:       "beforeold"
header-style: text
catalog:      true
tags:
    - CocoaPods
---

# 背景
在 pod install 时常遇到无法找到可用版本 pod 的问题，主要有以下情况
# 情况 1： 本地 pod repo 没有更新
如果使用了 pod install --fast-mode 命令，那么本地 spec repo 不会更新，这时的解决方案是更新 spec repo，可以直接尝试：
- 1、```pod install --verbose```
- 2、或者先执行 ```pod repo update```再 ```pod update --fast-mode```

# 情况 2： 有一些 pod 的版本号是字符串版本（pre-release）
如果有一些间接引入的 pod 版本为字符串版本，比如 ```2.3-SNAPSHOT```，或者 ```1.0-rc.2```，此时 pod install 日志会有以下错误信息。
```Plain Text
  In snapshot (Podfile.lock):
    TTTracker (= 1.1.11-rc.0, ~> 1.1.1)

  In Podfile:
    TTPushSDK (from `git url`, commit `575fad426b1e7581458991f529fe42e81095c297`) was resolved to 0.1.3, which depends on
      TTTracker

There are only pre-release versions available satisfying the following requirements:

	'TTTracker', '= 1.1.11-rc.0, ~> 1.1.1'

	'TTTracker', '>= 0'

You should explicitly specify the version in order to install a pre-release version
```

比如这个 [CocoaPods GitHub Issue](https://github.com/CocoaPods/CocoaPods/issues/8216) 中提到的例子，其实是某个 pod A 的名称不是正式版，而有另外两个 pod B 和 C 间接依赖了 pod A 导致的，这种情况有两种方案。
1、执行 pod update，这种会忽略 pre-release 版本的冲突，pod update 有带入其他 pod 变更的风险，建议下面的方案二
2、显式地在 podfile 中声明该 pod A，而不是单纯地通过 podspec 间接依赖引入
```
pod 'PodA', 'ABC-rc.0'
```

# 情况 3：依赖发生了显著变更（break change）
比如比如修改某个库的版本后新建了间接依赖后会出现这种情况。
此时需要对这个 pod 进行更新，执行如下命令
```
pod update PodA
```

# 小结
通过学习 pod install 和 pod update ，以及相关参数的意义可以更加便利地解决各种 pod 的依赖管理问题。

可学习参考资料：[CocoaPods 官方网站文档 pod install vs. pod update](https://guides.cocoapods.org/using/pod-install-vs-update.html)