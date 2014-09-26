---
layout: default
title: BKS Wrong version of key store
---
<h2>{{ page.title }}</h2>
<h3>分类</h3>
<p>Security;Certificate;安全;数字证书;</p>
<h3>关键词</h3>
<p>bks;</p>
<h3>问题现象</h3>
<p>在Android环境使用BouncyCastle生成的keystore文件时，提示异常“Wrong version of key store”。</p>

<h3>原因分析</h3>
<p>抛出异常的位置为bcprov的jar包中JDKKeyStore的engineLoad方法。该方法会读取keystore中的版本号并与jar支持的版本号对比。如果不一致，则抛出异常。详情请对比分析bcprov-jdk16-1.46.jar和bcprov-jdk16-1.47.jar中org.bouncycastle.jce.provider.JDKKeyStore类。</p>

<h3>问题处理</h3>
<p>第一步：检查Andorid平台内置的BoucnyCastle版本。Android不同的平台支持的版本不一样，因此需要运行一下查看内置的版本。执行以下代码：</p>
<p>Security.getProvider("BC").getVersion();</p>
<p>该代码会返回1.46,1.47等不同结果。</p>

<p>第二步：根据BouncyCastle版本确定支持的keystore版本。查看JDKKeyStore中STORE_VERSION常量。经过对比，bcprov-1.46及其以前的版本支持的keystore版本为1,bcprov-1.47及其以后的版本支持的keystore版本为2。</p>

<p>第三步：检查keystore文件的版本。使用DataInputStream读取keystore文件的第一个int。可能有两个结果1或者2。</p>

<p>第四步：根据Andorid平台内置BouncyCastle版本重新生成keystore文件。</p>

