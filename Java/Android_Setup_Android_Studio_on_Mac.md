#Android: Setup Android Studio on Mac
#Mac系统Android Studio安装笔记
`Android` `Android Studio` `Mac`

##引子
办公的台式机不能访问互联网，上面安装的adt bundle能够编译打包，但是可能是由于缺少必要的驱动，始终无法连接真机进行调试，分析解决Android项目的问题相当费劲。<br/>
于是我打算在可以访问互联网的Mac上搭建一个Android的开发环境。<br/>
据说现在都开始用Google提供的Android Studio作为IDE，所以我也来凑个热闹。<br/>

##搭建Android开发环境
###下载sdk tools
前往Android Studio中文社区下载android-sdk_r24-macosx.zip，将zip包解压。Android Studio中文社区的链接：[android-studio](http://www.android-studio.org/)，下同。<br/>
###下载安装Android Studio
前往Android Studio中文社区下载android-studio-ide-135.1629389.dmg。<br/>
Android Studio要求JDK版本为7以上，如果低于该版本，请前往Oracle官网下载安装JDK7以上新版本。[JDK链接](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)<br/>
双击dmg文件，弹出一个界面，将Android studio的图标向右拖进文件夹（应用程序）即可。<br/>
###运行Android Studio
在Finder的应用程序中找到Android Studio的图标双击运行。<br/>
这一步可能出现的问题：<br/>
1. android studio unable to find a valid jvm<br/>
原因：Mac的jdk版本（我的Mac上是1.8）与Android Studio配置不一致。<br/>
解决方法：右键点击Android Studio的图标，“显示内容”，用文本编辑打开Contents里面的Info.plist文件，搜索JVMVersion，将值1.6改为1.8。<br/>
2. fetching android sdk component information<br/>
界面一直显示该内容，进度条没有任何进展。<br/>
原因：Android Studio一直尝试在线获取sdk组件信息。<br/>
解决方法：右键点击Android Studio的图标，“显示内容”，用文本编辑打开Contents/bin下的idea.properties文件，在最下面添加一行内容：disable.android.fisrt.run=true。<br/>
此时可能界面提示还没有结束，可以尝试强行退出。实在不行只能ps aux | grep Android找出对应进程，然后kill -9杀掉了。<br/>
###配置SDK路径
在初始界面选择“Configure”——“Project Defaults”——“Project Structure”。配置JDK Location，Android SDK Location。<br/>
前者通常路径为/Library/Java/JavaVirtualMachines/jdk1.8.0_31.jdk/Contents/Home，后者即为下载解压的sdk目录。<br/>
###下载SDK
在初始界面选择“Configure”——“SDK Manager”。<br/>
这一步可能出现的问题：<br/>
1. 无法下载或者下载过慢。<br/>
原因：网络无法访问google.com的任何服务。<br/>
解决方法一：在/etc/hosts文件中配置本地地址解析。试过了，依然非常慢，速度小于10KB/s。<br/>
解决方法二：使用代理服务器。点击“Android SDK Manager”菜单（好像有点bug，这个菜单有时无法显示，需要多试几次），进入“Preferences”，填写相应的代理服务器和端口。[代理服务器列表](http://tools.android-studio.org/index.php/85-tools/110-androidsdk-mirrors)。<br/>
2. url解析失败等其他错误<br/>
原因：未知。<br/>
解决方法：下载离线包，放入sdk相应目录。例如，我下载安装Android Support Library一直不成功。只好去下载了一个离线包，解压放到sdk目录里，下载地址[Android SDK Extras](http://www.androiddevtools.cn/)。<br/>

##Android HelloWorld
###新建工程
点击“Start a new Android Studio project”，输入Application name，例如HelloWorld。<br/>
选择运行环境，默认为Phone，Mini SDK为API 15。<br/>
选择Activity，默认为Blank Activity。<br/>
填写Activity Name、Layout Name、Title等等，默认即可。<br/>
###编译运行
点击“Make Project”编译，点击“Run”运行。<br/>

##相关链接<br/>
[Android Studio中文社区](http://www.android-studio.org/)<br/>
[AndroidDevTools](http://www.androiddevtools.cn/)<br/>
[Android Studio开发环境部署](http://www.cnblogs.com/duxiuxing/p/4206051.html)<br/>
[Android Studio初步使用教程](http://www.cnblogs.com/xiaoran1129/archive/2013/05/23/3095135.html)<br/>

<br/>
2015-4-26