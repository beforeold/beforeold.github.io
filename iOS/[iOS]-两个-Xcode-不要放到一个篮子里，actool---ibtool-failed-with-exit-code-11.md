For English readers:
**Don't put two Xcode applications into the same folder. It's recommended that do not put two Xcodes in Applciation folder, instead put the other one  in Documents or Desktop or make a new folder for it if prefered.**


**两个 Xcode 不能放在同一个文件夹，尽可能的不要在 Applications 应用程序文件夹内放两个 Xcode ，可以在 文稿 Documents 或者桌面放置，或者放到某个文件夹内**

这个应该是老司机很熟悉的了，这次解压缩一个 Xcode 9包后，重命名为 Xcode 9.0 后一股脑丢到**应用程序**文件中导致出错，即便是模板工程都无法编译成功，
出现很奇怪的问题，比如说一些 Xcode 内置的工具程序，比如 ```ibtool```、 ```actool``` 启动异常导致项目无法编译成功。错误提示：


![模板工程无法编译成功](http://upload-images.jianshu.io/upload_images/73339-cdf6495c724c2152.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
Command /Applications/Xcode.app/Contents/Developer/usr/bin/actool failed with exit code 11


ommand /Applications/Xcode.app/Contents/Developer/usr/bin/ibtool failed with exit code 11

```

解决办法：
- 新建一个文件夹存放一个 Xcode 程序后将文件夹放到应用程序文件夹内
- 或者干脆不放到 应用程序文件夹内。（**推荐**）
