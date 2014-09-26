
#OpenSSL verify unable to get local issuer certificate (illegal KeyUsage)

`OpenSSL` `verify`

##问题现象
在使用OpenSSL的verify指令验证证书时，提示异常“unable to get local issuer certificate”。

##原因分析
* 根据[verify指令的文档](http://www.openssl.org/docs/apps/verify.html "verify指令的文档")，含义是找不到颁发者的证书。可是颁发者的证书已经通过-CAfile参数指定。一个值得怀疑的地方是颁发者的证书密钥用法不包含CertSign，会不会与此相关？只能想办法看verify指令所对应的代码。

* verify指令文档提示错误代码定义于x509\_vfy.h中（本文使用代码版本为openssl-1.0.1h）。打开openssl-1.0.1h\include\openssl\x509\_vfy.h，内容为“../../crypto/x509/x509\_vfy.h”。打开对应的x509\_vfy.c，发现一个重要的函数：X509\_verify\_cert。打开[X509\_verify\_cert文档](http://www.openssl.org/docs/crypto/X509_verify_cert.html "X509_verify_cert文档")，该函数为verify指令对应的代码。

* 分析X509\_verify\_cert函数，发现抛出错误的调用栈：X509\_verify\_cert->find\_issuer->check\_issued->X509\_check\_issued。其中X509\_check\_issued位于v3_purp.c文件中（找这个函数所在的文件花了一会儿时间，就不多说了）。

* X509\_check\_issued中如果发现颁发者密钥用法不包含KU\_KEY\_CERT\_SIGN就会报错X509\_V\_ERR\_KEYUSAGE\_NO_CERTSIGN。



##问题处理
* 从规范和安全的角度看，你不应该允许一个密钥用法不包含CertSign的实体签发证书。这样就不会出现上面的问题。实在是被逼的，那继续往下看。
*
* 修改平台使用的OpenSSL代码，重新编译，更新平台库。但是很有可能你动不了平台OpenSSL库。

* 调用OpenSSL基础函数自己写一个X509\_verify\_cert，给自己的程序调用。在这个X509\_verify\_cert函数中，你可以规避X509\_check_issued的检查。

<br>2014-07-21
