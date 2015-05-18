#Certificate: How to verify certificate in Java
#如何使用Java程序验证数字证书（是否可信）
`Certificate` `Verify` `Java`

##引子

数字证书是安全通信的基础，我们经常需要验证证书是否可信。<br/>
*	一种方法是使用命令行。
*	一种方法是使用程序验证。
<br/>

命令行，比如openssl的verify命令。<br/>
`openssl verify --help`
`usage: verify [-verbose] [-CApath path] [-CAfile file] [-purpose purpose] [-crl_
check] [-attime timestamp] [-engine e] cert1 cert2 ...`
<br/>
程序验证，本文介绍使用Java程序验证证书是否可信的三种方法。<br/>


##使用CertPathBuilder，尝试构造完整证书链
使用java.security.cert.CertPathBuilder类的build方法，如果能够成功构造一个完整的证书链，那么说明终端证书可信。<br/>
代码如下：
<br/>
```Java
	
    private static final String CERT_STORE_COLLECTION = "Collection";
    private static final String PKIX                  = "PKIX";
	 /**
     * 验证证书（使用CertPathBuidler实现，使用JCE默认provider）
     * 
     * @param cert 被验证证书
     * @param trustedRootCerts 受信任证书
     * @param intermediateCerts 中间证书
     * @return true，可信任；false，不可信任
     * @throws CertificateException
     * @throws NoSuchAlgorithmException
     * @throws NoSuchProviderException
     * @throws InvalidAlgorithmParameterException
     */
    public static boolean verifyCertificateByCertPathBuidler(X509Certificate cert,
            Set<X509Certificate> trustedRootCerts, Set<X509Certificate> intermediateCerts) throws CertificateException,
            NoSuchAlgorithmException, NoSuchProviderException, InvalidAlgorithmParameterException {

        X509CertSelector selector = new X509CertSelector();
        selector.setCertificate(cert);

        Set<TrustAnchor> trustAnchors = new HashSet<TrustAnchor>();
        for (X509Certificate trustedRoot : trustedRootCerts) {
            trustAnchors.add(new TrustAnchor(trustedRoot, null));
        }

        PKIXBuilderParameters pkixParams = new PKIXBuilderParameters(trustAnchors, selector);

        pkixParams.setRevocationEnabled(false);// 关闭revocation检查，TODO

        CertStore intermediateCertStore = CertStore.getInstance(CERT_STORE_COLLECTION,
                new CollectionCertStoreParameters(intermediateCerts));
        pkixParams.addCertStore(intermediateCertStore);

        CertPathBuilder builder = CertPathBuilder.getInstance(PKIX);
        try {
            builder.build(pkixParams);
            return true;
        }
        catch (CertPathBuilderException e) {
            // TODO logger
            e.printStackTrace();
        }
        return false;
    }
```
<br/>

##使用CertPathBuilder，基于BouncyCastle实现
上面的代码如果直接在BouncyCastle环境下使用会报错：“java.security.cert.CertPathBuilderException: No certificate found matching targetContraints.”。导致这个错误的原因是BouncyCastle对CertPathBuilder的实现与JCE默认实现有一点点差异。具体可以看看BouncyCastle的对应源码，[PKIXCertPathBuilderSpi](http://www.docjar.org/html/api/org/bouncycastle/jce/provider/PKIXCertPathBuilderSpi.java.html)。（说明各个provider在实现标准接口时还是有些差异的，编写跨provider的程序时需要多测试验证。）
<br/>
```Java
	
	try
	{
		targets = CertPathValidatorUtilities.findCertificates((X509CertStoreSelector)certSelect, pkixParams.getStores());
		targets.addAll(CertPathValidatorUtilities.findCertificates((X509CertStoreSelector)certSelect, pkixParams.getCertStores()));
	}
	catch (AnnotatedException e)
	{
		throw new ExtCertPathBuilderException(
			"Error finding target certificate.", e);
	}

	if (targets.isEmpty())
	{

		throw new CertPathBuilderException(
			"No certificate found matching targetContraints.");
	}
```

从BouncyCastle的源码可以看出，我们需要将被校验的证书也放入CertStore。因此，基于BouncyCastle实现的代码如下：<br/>
```Java
	
	private static final String BC_PROV               = "BC";
    private static final String CERT_STORE_COLLECTION = "Collection";
    private static final String PKIX                  = "PKIX";
	
	static {
        // 添加provider
        Security.addProvider(new org.bouncycastle.jce.provider.BouncyCastleProvider());
    }
	
	/**
     * 验证证书（使用CertPathBuidler实现，使用BoncyCastle作为provider）
     * 
     * @param cert 被验证证书
     * @param trustedRootCerts 受信任证书
     * @param intermediateCerts 中间证书
     * @return true，可信任；false，不可信任
     * @throws CertificateException
     * @throws NoSuchAlgorithmException
     * @throws NoSuchProviderException
     * @throws InvalidAlgorithmParameterException
     */
    public static boolean verifyCertificateByCertPathBuidlerUseBC(X509Certificate cert,
            Set<X509Certificate> trustedRootCerts, Set<X509Certificate> intermediateCerts) throws CertificateException,
            NoSuchAlgorithmException, NoSuchProviderException, InvalidAlgorithmParameterException {

        X509CertSelector selector = new X509CertSelector();
        selector.setCertificate(cert);

        Set<TrustAnchor> trustAnchors = new HashSet<TrustAnchor>();
        for (X509Certificate trustedRoot : trustedRootCerts) {
            trustAnchors.add(new TrustAnchor(trustedRoot, null));
        }

        PKIXBuilderParameters pkixParams = new PKIXBuilderParameters(trustAnchors, selector);

        pkixParams.setRevocationEnabled(false);// 关闭revocation检查，TODO

        // Add for BC only, or you will get exception
        // "java.security.cert.CertPathBuilderException: No certificate found matching targetContraints."
        intermediateCerts.add(cert);

        CertStore intermediateCertStore = CertStore.getInstance(CERT_STORE_COLLECTION,
                new CollectionCertStoreParameters(intermediateCerts), BC_PROV);
        pkixParams.addCertStore(intermediateCertStore);

        CertPathBuilder builder = CertPathBuilder.getInstance(PKIX, BC_PROV);
        try {
            builder.build(pkixParams);
            return true;
        }
        catch (CertPathBuilderException e) {
            // TODO logger
            e.printStackTrace();
        }
        return false;
    }
```
<br/>

##使用CertPathValidator
另外一个可以验证证书链的方法就是java.security.cert.CertPathValidator的validate方法，原理上与CertPathBuilder是一样的，代码略有差异。<br/>
```Java

	private static final String BC_PROV               = "BC";
    private static final String PKIX                  = "PKIX";
	private static final String X_509                 = "X.509";
	
	static {
        // 添加provider
        Security.addProvider(new org.bouncycastle.jce.provider.BouncyCastleProvider());
    }
	/**
     * 验证证书（使用CertPathValidator实现，使用BoncyCastle作为provider，JCE默认provider不支持）
     * 
     * @param cert 被验证证书
     * @param trustedRootCerts 受信任证书
     * @param intermediateCerts 中间证书
     * @return true，可信任；false，不可信任
     * @throws CertificateException
     * @throws NoSuchAlgorithmException
     * @throws NoSuchProviderException
     * @throws InvalidAlgorithmParameterException
     */
    public static boolean verifyCertificateByCertPathValidator(X509Certificate cert,
            Set<X509Certificate> trustedRootCerts, Set<X509Certificate> intermediateCerts) throws CertificateException,
            NoSuchAlgorithmException, NoSuchProviderException, InvalidAlgorithmParameterException {

        Set<TrustAnchor> trustAnchors = new HashSet<TrustAnchor>();
        for (X509Certificate trustedRoot : trustedRootCerts) {
            trustAnchors.add(new TrustAnchor(trustedRoot, null));
        }

        PKIXParameters pkixParams = new PKIXParameters(trustAnchors);

        pkixParams.setRevocationEnabled(false);// 关闭revocation检查，TODO

        List<X509Certificate> certs = new ArrayList<X509Certificate>();
        certs.add(cert);
        certs.addAll(intermediateCerts);

        CertificateFactory certificateFactory = CertificateFactory.getInstance(X_509, BC_PROV);
        CertPath certPath = certificateFactory.generateCertPath(certs);

        CertPathValidator validator = CertPathValidator.getInstance(PKIX, BC_PROV);

        try {
            validator.validate(certPath, pkixParams);
            return true;
        }
        catch (CertPathValidatorException e) {
            // TODO logger
            e.printStackTrace();
        }
        return false;
    }
```
<br/>

##相关链接
[x509-certificate-validation-in-java-build-and-verify-chain-and-verify-clr-with-bouncy-castle](http://www.nakov.com/blog/2009/12/01/x509-certificate-validation-in-java-build-and-verify-chain-and-verify-clr-with-bouncy-castle/)

<br/>
2015-5-18