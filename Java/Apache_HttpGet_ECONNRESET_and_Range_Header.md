#Apache HttpGet ECONNRESET and Range Header
#HttpGet连接被重置与HTTP协议Range头部
`HttpClient` `HttpGet` `ECONNRESET` `Range`

##引子
最近在尝试使公司内部应用中心系统（AppStore）使用CDN，以减少公司的互联网带宽占用，同时提高应用下载速度。<br/>
应用中心服务器架设的公司数据中心，android客户端在互联网请求应用列表、详情，并下载应用。<br/>
这种访问架构下，应用下载会占用公司的互联网带宽，极端情况下会影响主营业务。而且，天南海北的用户都要直连服务器请求文件，客户端下载效果也不是很好。<br/>
<br/>
##问题简述
在测试中发现，使用CDN时所有请求都会出现ECONNRESET异常，应用无法下载。
```Java
	
	java.net.SocketException: recvfrom failed: ECONNRESET (Connection reset by peer)
		at libcore.io.IoBridge.maybeThrowAfterRecvfrom(IoBridge.java:542)
		at libcore.io.IoBridge.recvfrom(IoBridge.java:506)
		at java.net.PlainSocketImpl.read(PlainSocketImpl.java:488)
```
<br/>
##问题分析
###ECONNRESET (Connection reset by peer)为何方神圣？
对方复位连接。相对于应用中心，对方是CDN服务器，也有可能有中间网络设备。<br/>
关于TCP reset报文，这里有篇文章写得比较详细：[TCP异常终止（reset报文）](http://www.vants.org/?post=22)。作者指出，利用reset报文可以达到两个效果：一、网络安全设备阻断异常连接。二、攻击主机，例如实现会话劫持。<br/>
###服务器端有没有报错？
CDN节点服务器并没有出现异常情况。<br/>
会不会是中间网络设备阻断了？<br/>
###HttpGet请求有何特殊之处？
仔细阅读代码发现，为了提高下载的效率，程序员使用了多线程下载，将文件分成几段，每个线程下载一段。这需要用到HTTP的Range头部。[HTTP Range](http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.35)。<br/>
```Java
	
	if (startPos != -1 && endPos != -1) {
		httpGet.setHeader("Range", "bytes=" + startPos + "-" + endPos);
	}
```
###模拟下载是否报错？
CDN工程师使用curl模拟了下载，并无异常。[cURL](http://curl.haxx.se/)<br/>
```Bash
	
	curl "url" -voa -H "Range: bytes=0-1000"
```
在公司WiFi环境下模拟下载，发现异常。<br/>
使用手机热点模拟下载，并无异常。<br/>
###公司网络不支持Range头部
原来为了防止员工使用多线程下载软件占用公司带宽，公司网络禁止了Range头部。
##问题解决
联系公司网络管理人员开放特定账号，允许Range头部。
##小结
为了这个问题前后纠结了两三天，中间做过很多尝试，补充说明一下。<br/>
有人声称碰到过类似的问题，结果发现Range头部是大小写敏感的，使用“RANGE”就可以。（HTTP标准里面是“Range”哦）<br/>
有人声称是服务器端的问题，连接未及时关闭，需要设置系统属性http.keepAlive为false。[链接](http://stackoverflow.com/questions/11207394/getting-socketexception-connection-reset-by-peer-in-android)。<br/>

<br/>
2015-4-19
