# 关于热修复和JSPatch
热修复(hot patch)是指不升级app的情况下，通过加载网络脚本来为已上线的App替换或新增功能。

JSPatch是一个基于Javascript脚本实现热修复的开源项目，JSPatch平台打包了一整套脚本上传、版本管理、脚本请求更新等一系列热修复策略，提供JSPatch.framework供简单使用。
检查脚本请求次数低于50万次每月时不收取费用。

[JSPatch官方网站](http://jspatch.com/)	
[开源项目Github-Wiki](https://github.com/bang590/JSPatch/wiki)
[实现原理详解](https://github.com/bang590/JSPatch/wiki/JSPatch-%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86%E8%AF%A6%E8%A7%A3)
	
# 基本原理和流程
通过iOS的JavascriptCore.framework将请求下载的脚本执行，以运行时的方式替换旧代码实现或者新增功能。
作用的主要流程如下：
1. 集成JSPatch.framework 
2. 检查当前App版本是否有脚本更新
3. 下载更新的脚本保存到本地
4. 执行脚本进行热修复 

# JSPatch的集成
1. 下载最新的SDK，JSPatch.framework，目前最新版本为v1.5
2.  将SDK导入工程后，需要依赖libz、JavaScriptCore两个库
3. 调试模式可以分为两种，本地导入脚本调试和线上修复（两者不能同时执行）
 
    - 本地调试：手动导入main.js脚本，
```objc
        在application:didFinishLaunchWithOptions:方法中执行：
		[JSPatch testScriptInBundle];
		则加载本地脚本执行调试性的热修复
```
    - 线上修复：将对应版本的脚本（命名为main.js）上传到JSPatch平台，
```objc
		在application:didFinishLaunchWithOptions:方法中执行：
[JSPatch startWithAppKey:kJSPatchAppKey];
[JSPatch sync];
则会在启动时检查当前版本app是否有补丁更新。
```

4. 手动调用检查补丁更新：
	如果对补丁更新检查有实时性高的要求，可以手动在必要时调用[JSPatch sync];方法进行补丁更新检查，调用次数计入JSPatch的平台访问次数，比如在aplicationWillBecomeActive:中进行检查。

# 开发预览模式
线上修复支持开发预览模式，在app中启动开发者预览模式时，只会检查和应用线上的预览模式下的补丁，而不用正式的补丁，当测试预览模式补丁通过后，可以选择进行全量的正式发布。
 
开发预览模式下不会加载非开发预览模式下补丁，非开发预览模式只会加载正式补丁。

# 灰度和条件发布：
根据实际的需要可以进行特别的补丁发布范围
 
灰度：随机选择一定比例的用户进行发布，可不断提高灰度直至全量发布
 
条件发布：针对满足条件的用户（如设备类型，系统版本，或者自定义的用户参数）进行补丁更新，条件发布命中后不可取消。（通过[JSPatch setupUserData:userData]决定当前用户参数）


# 补丁的管理
在JSPatch平台注册的app，按不同的app版本进行补丁管理，支持新增、更新和删除补丁，
新增：创建app的某一个版本，上传脚本（命名必须为main.js）


# 基本用法
编写main.js文件，定义要修复的类，导入代码需要使用的类，以Javascript的逻辑实现相关功能。
```javascript
defineClass("ViewController", { // 要修复的类
                badMethod: function() { // 要修复的方法
                    console.log("fixed badMethod 1 * 16 release”); // 打印
                    require(‘UIView') // 导入UIView
                    require(‘UIColor')  // 导入UIColor
                    self.view().setBackgroundColor(UIColor.redColor()); // 修改背景颜色
            }
        })
```
更多实现参考官方文档

# 额外功能之在线参数
在线参数是独立于JSPatch热修复之外的在线参数功能，通过在JSPatch平台设置好多个key-value后，可以在app内随时提取相关在线参数。

# 关于JSPatch的安全
- 官方描述
	[将脚本保存在七牛云，在传输过程中执行RSA签名加密，脚本在本地存储进行了对称加密。](http://jspatch.com/Docs/security)		
- 第三方分析
	[iOS远程hot patch的优点和风险](http://drops.wooyun.org/tips/13248)
	[“Hotpatch”潜在的安全风险](http://www.2cto.com/Article/201606/519426.html)
# demo
[demo地址](https://github.com/beforeold/JSPatch_Demo)
