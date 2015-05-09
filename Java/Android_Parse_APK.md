#Android: Parse APK
#Android 解析APK文件
`Android` `APK` `Parse`

##引子
开发者通过网站填写应用描述、截图、协议等信息，然后上传APK文件。我们需要校验签名，解析/存储包名、版本等信息。<br/>
包名、版本等信息都在APK文件里面，不需要开发者手工填写，而且他们也可能会填错了。<br/>
那么问题来了，程序如何获得APK文件的包名、版本等信息。<br/>
读取AndroidManifest.xml文件。<br/>
<br/>

##AndroidManifest.xml
在Android工程和应用程序中，AndroidManifest.xml位于根目录，定义了应用程序及其组件的结构和元数据。<br/>

##Android Binary Xml (ABX)
在一个Android工程里面AndroidManifest.xml文件是以文本XML的形式存储的。但是当你打包成一个APK文件后，APK文件里面的AndroidManifest.xml是以二进制XML的形式存储的。你可以用文本编辑器打开试试看，都是乱码。<br/>

##解析二进制AndroidManifest.xml
###使用Prasanta Paul的Android_BX2工具类
最开始在网上找到的是Prasanta Paul的Android_BX2工具类。相应代码可以参见示例工程。<br/>
但是在使用中发现该类在将ABX转为文本XML时存在较多BUG，转换结果中经常出现乱码。<br/>
###使用apk-parser
Liu Dong在github上提供了一个非常好的apk解析包：[apk-parser](https://github.com/xiaxiaocao/apk-parser)。目前为止没有发现BUG。<br/>
使用非常简单，如下：<br/>
<br/>
```Java
	
	public static AndroidManifest parseAndroidManifestXmlByApkParser(File apk)
			throws Exception {

		AndroidManifest manifest = new AndroidManifest();
		ApkParser apkParser = null;
		try {
			apkParser = new ApkParser(apk);
			String xml = apkParser.getManifestXml();
			logger.debug("AndroidManifest.xml: {}", xml);
			ApkMeta apkMeta = apkParser.getApkMeta();
			manifest.setPackageName(apkMeta.getPackageName());
			manifest.setVersionCode(String.valueOf(apkMeta.getVersionCode()));
			manifest.setVersionName(apkMeta.getVersionName());
		} catch (Exception e) {
			// TODO: handle exception
			throw e;
		} finally {
			if (apkParser != null) {
				apkParser.close();
				apkParser=null;
			}
		}
		return manifest;
	}
```
<br/>
还有一个更简单的apk-parser（只有15k的jar包），由joakime提供，[android-apk-parser](https://github.com/joakime/android-apk-parser)。


##示例工程下载
[ParseApkExample.zip](https://github.com/duanjfeng/trickle/blob/master/attaches/ParseApkExample.zip)<br/>

##参考链接
二进制XML不是本文讨论的主要内容，列出几个链接，供参考。<br/>
oschina的介绍：[二进制的XML规范](http://www.oschina.net/p/xdbx)，“Binary XML是XML数据紧凑的二进制表示形式，显著地降低了XML数据的冗余性，使得XML数据的解析也变得容易很多，减轻了处理XML数据的系统的运算工作，降低了XML数据传输时所占的带宽。Binary XML首先由无线应用领域提出并使用，先后有一些不同的规范出现，包括 Wbxml（WAP Binary XML）、Fast Infoset（X.891）和 EXI（Efficient XML Interchange）等。”<br/>
国外友人的分析：[The Performance Woe of Binary XML](http://www.codeproject.com/Articles/23857/The-Performance-Woe-of-Binary-XML)，[Why Binary XML?](http://www.oss.com/xml/products/binary-xml-technology.html)。<br/>

<br/>
2015-5-9