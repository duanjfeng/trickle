#Java Security: Why you should store sensitive inforamtion in char array instead of String ?
#Java安全：为什么应该使用char数组而不是String存储敏感信息？
`Java` `Security` `sensitive information` `char array` `String`

##引子
本主题源于stackoverflow的一个问题：[why-is-char-preferred-over-string-for-passwords](http://stackoverflow.com/questions/8881291/why-is-char-preferred-over-string-for-passwords)。<br/>
由于stackoverflow的回答和讨论比较凌乱，所以整理一下放在这里。<br/>
这个问题的一个回答是String不及char数组安全，因为String是不可变量，容易输出到控制台或者文件。
<br/>

##String是不可变量
String是不可变量（immutable），没有任何方法使得程序员可以在使用完String之后修改或者清空String的内容。因此String将一直存在于内存中，直到gc回收。<br/>
Jvm使用String pool存储String数据以节省内存、提高性能，因此String数据会很高概率地驻留在内存中不被回收。<br/>
这两点给敏感信息造成了安全隐患，恶意攻击者能够从内存dump中直接获得敏感信息的明文。
<br/>
Oracle在Java™ Cryptography Architecture (JCA) Reference Guide的[Using Password-Based Encryption](http://docs.oracle.com/javase/8/docs/technotes/guides/security/crypto/CryptoSpec.html\#PBEEx)一文中给出了一个官方的说明，如下。<br/>
	It would seem logical to collect and store the password in an object of type java.lang.String. However, here's the caveat: Objects of type String are immutable, i.e., there are no methods defined that allow you to change (overwrite) or zero out the contents of a String after usage. This feature makes String objects unsuitable for storing security sensitive information such as user passwords. You should always collect and store security sensitive information in a char array instead.
##String容易输出
相比char数组，String更容易输出到控制台或者文件。<br/>

```Java
	
	String strPwd = new String("secure");
	char[] charPwd = new char[] { 's', 'e', 'c', 'u', 'r', 'e' };
	System.out.println("Password:" + strPwd);
	System.out.println("Password:" + charPwd);
	System.out.println(strPwd);
	System.out.println(charPwd);
```
<br/>
上面代码的输出为：
	Password:secure
	Password:\[C@18f5824
	secure
	secure
<br/>
可以看出char数组的toString方法并不会将数组的内容输出，从而在很大程度上降低了程序员或者各种拦截器、动态代理等有意无意将敏感信息明文输出的可能。<br/>

##seccodeguide-Confidential Information
Oracle在Secure Coding Guidelines for Java SE的[Confidential Information](http://www.oracle.com/technetwork/java/seccodeguide-139067.html\#2)一节对敏感信息的处理提出了3点建议与本文主题都多少有些关联，简要说明一下。<br/>
指导思想：<br/>
*保证敏感数据只在受限上下文中可读；
*保证可信数据不可被篡改；
*保证特权代码不被恶意执行。
<br/>
实操指南：<br/>
*不要将敏感信息暴露到异常中。
*不要将敏感信息记录到日志中。
*使用完后将敏感信息从内存中清除。
<br/>
第一个例子：FileNotFoundException包含文件路径信息，直接抛出给调用方可能会造成文件系统结构泄露。<br/>
第二个例子：本文提到的password明文。<br/>
第三个例子：本文提到的password明文。<br/>

##参考链接
[What is the Java string pool and how is “s” different from new String(“s”)?](http://stackoverflow.com/questions/2486191/what-is-the-java-string-pool-and-how-is-s-different-from-new-strings)

<br/>
2015-5-10