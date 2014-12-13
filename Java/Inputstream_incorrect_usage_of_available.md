#Inputstream incorrect usage of available
`Inputstream` `available`
##问题简述
前段时间有个同事碰到一个诡异的问题：程序从远程ftp服务器上下载文件偶尔会失败，明明远程有文件，却没有下载到本地来。而且只是偶尔发生，无法准确的复现。
##问题分析
###日志分析
首先想到的是看日志，结果被告知程序没有报错，没有错误日志。而且程序为了减少不必要的IO，提高性能，也没有打印任何INFO/DEBUG日志。
###代码分析
程序已经在生产上跑着了，下次部署时可以考虑加上INFO/DEBUG日志，不过现在只能先看看代码了。于是拿来了ftp下载的代码，如下。代码中fc是sun.net.ftp.FtpClient。
```Java
TelnetInputStream fget = null;
		try {
			fc.binary();
			fget = fc.get(serverPath + sourceFileName);
			int i = fget.available();
			if (-1 == i) {
				return null;
			}
			bytes = new byte[i];
			int offset = 0;
			int numRead = 0;
			while (offset < bytes.length
					&& (numRead = fget.read(bytes, offset, bytes.length
							- offset)) >= 0) {
				offset += numRead;
			}
		} catch (IOException e) {
```
一看代码，问题就来了，写fget.available()的目的是啥？实际这个代码又做了啥？<br>
回答第1个问题：程序员想申请一个恰到好处的缓冲区，大小跟文件一样，以便将远程的文件存储进来，毕竟运行时文件到底多大我们都不知道，恰到好处就不会为预设缓冲区的大小伤脑筋了。<br>
回答第2个问题：实际这个代码不一定能够准确的反应文件的大小。<br>
###available()方法科普
jdk doc有说明，available()方法返回的是预估在不阻塞的情况下该Inputstream下一个方法调用可读（跳过）的字节数量。<br>
`Returns an estimate of the number of bytes that can be read (or skipped over) from this input stream without blocking by the next invocation of a method for this input stream.`<br>
重点就在这个“不阻塞的情况下”。所以jdk doc后面又赶紧补充说，有些实现会返回流的字节总数，但是很多不会。尝试使用这个返回值去分配缓冲区以保存流的所有数据是不正确的。<br>
`Note that while some implementations of InputStream will return the total number of bytes in the stream, many will not. It is never correct to use the return value of this method to allocate a buffer intended to hold all data in this stream. `<br>
到这里就明白为什么说上述代码不能够准确反应文件大小了吧。sun.net.TelnetInputStream的available()并没有返回文件的实际大小。[TelnetInputStream源码](http://hg.openjdk.java.net/jdk7/jdk7/jdk/file/3de7b0daf355/src/share/classes/sun/net/TelnetInputStream.java)
###available()返回文件实际大小的例子
dalvik虚拟机的FileInputStream在实现available方法时，返回文件实际大小。[android FileInputStream源码](http://netmite.com/android/mydroid/1.6/dalvik/libcore/luni/src/main/java/java/io/FileInputStream.java)<br>
`This method always returns the size of the file minus the current position.`<br>
```Java
/**
     * Returns the number of bytes that are available before this stream will
     * block. This method always returns the size of the file minus the current
     * position.
     * 
     * @return the number of bytes available before blocking.
     * @throws IOException
     *             if an error occurs in this stream.
     * @since Android 1.0
     */
    @Override
    public int available() throws IOException {
        openCheck();
        // BEGIN android-added

        // Android always uses the ioctl() method of determining bytes
        // available. See the long discussion in
        // org_apache_harmony_luni_platform_OSFileSystem.cpp about its
        // use.

        return fileSystem.ioctlAvailable(fd.descriptor);
        // END android-added 
    }
```
##问题解决
###预估缓冲区大小
由于available()并不返回文件实际大小，所以预估缓冲区大小。较为简单。
###实现available()方法
像dalvik虚拟机一样，继承Inputstream或者其子类，覆盖available()方法。较为复杂。


2014-12-13
