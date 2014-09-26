---
layout: default
title: OpenSSL verify unable to get local issuer certificate (illegal KeyUsage)
---
<h2>{{ page.title }}</h2>
<h3>分类</h3>
<p>Security;Certificate;安全;数据证书</p>
<h3>关键词</h3>
<p>OpenSSL;verify;</p>

<h3>问题现象</h3>
<p>在使用OpenSSL的verify指令验证证书时，提示异常“unable to get local issuer certificate”。</p>

<h3>原因分析</h3>
<p>1.根据verify指令的文档（http://www.openssl.org/docs/apps/verify.html），含义是找不到颁发者的证书。可是颁发者的证书已经通过-CAfile参数指定。一个值得怀疑的地方是颁发者的证书密钥用法不包含CertSign，会不会与此相关？只能想办法看verify指令所对应的代码。
</p>
<p>
2.verify指令文档提示错误代码定义于x509_vfy.h中（本文使用代码版本为openssl-1.0.1h）。打开openssl-1.0.1h\include\openssl\x509_vfy.h，内容为“../../crypto/x509/x509_vfy.h”。打开对应的x509_vfy.c，发现一个重要的函数：X509_verify_cert。打开对应的文档（http://www.openssl.org/docs/crypto/X509_verify_cert.html），该函数为verify指令对应的代码。
</p>
<p>
3.分析X509_verify_cert函数，发现抛出错误的调用栈：X509_verify_cert->find_issuer->check_issued->X509_check_issued。其中X509_check_issued位于v3_purp.c文件中（找这个函数所在的文件花了一会儿时间，就不多说了）。
</p>
<p>
4.X509_check_issued中如果发现颁发者密钥用法不包含KU_KEY_CERT_SIGN就会报错X509_V_ERR_KEYUSAGE_NO_CERTSIGN。
</p>


<h3>问题处理</h3>
<p>1.从规范和安全的角度看，你不应该允许一个密钥用法不包含CertSign的实体签发证书。这样就不会出现上面的问题。实在是被逼的，那请看2，3。</p>
<p>2.修改平台使用的OpenSSL代码，重新编译，更新平台库。但是很有可能你动不了平台OpenSSL库。</p>
<p>3.调用OpenSSL基础函数自己写一个X509_verify_cert，给自己的程序调用。在这个X509_verify_cert函数中，你可以规避X509_check_issued的检查。</p>
