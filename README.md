# AndroidGMSSE
	基于Java实现的国密TLCP套件（GB/T 38636-2020）适用于各个版本安卓，简单修改后也可以用于其他JAVA应用
	目前只实现了TLCP_ECC_SM4_CBC_SM3、TLCP_ECC_SM4_GCM_SM3、TLCP_ECDHE_SM4_CBC_SM3和TLCP_ECDHE_SM4_GCM_SM3。

# GMOkHttp3
	基于国密TLCP套件进行了适配，官方原版为：3.12.14并将工程修改为gradle(6.9.2)。
	仅仅修改了3个类：TlsVersion，CipherSuite和ConnectionSpec。
	编译环境：windows11 + JDK 8

# 用法
## Android
	直接使用编译的aar

## Java
	提取aar中的jar
## 日志模式
	目前缺省日志是System.out输出，如需根据情况修改，实现接口Logger.ILogWriter，并调用
	Logger.initialize(ILogWriter logWriter)进行设置。
	
# 样例代码
	private static final String[] TEST_HOST_URL = new String[]{
            "https://demo.gmssl.cn:1443",
            "https://demo.gmssl.cn:2443",	// ECDHE不支持了！
            "https://demo.gmssl.cn:3443",
    };
	
	private static final GMProvider gmProvider = new GMProvider();
    private static final BouncyCastleProvider bcProvider = new BouncyCastleProvider();
	
	private SSLSocketFactory initTestGMSSL() {
        try {
		    Logger.initialize(new Logger.ILogWriter() {	// 可选
                @Override
                public int println(int priority, String tag, String msg) {
                    if (priority >= Logger.DEBUG) {
                        System.out.println(tag + msg);
                    }
                    return 0;
                }
            });
            GMProvider.setEnableLog(false);
            GMProvider.setCipherSuites(new GMCipherSuite[]{GMCipherSuite.TLCP_ECDHE_SM4_CBC_SM3, GMCipherSuite.TLCP_ECC_SM4_CBC_SM3});

            KeyStore keyStore = KeyStore.getInstance("PKCS12", bcProvider);
            keyStore.load(new FileInputStream(getFilesDir() + "/sm2.GMSSLClient.both.pfx"), "12345678".toCharArray()); // 提前push文件到files或者从assets提取
            KeyManagerFactory kmf = KeyManagerFactory.getInstance("PKIX", gmProvider);
            kmf.init(keyStore, "12345678".toCharArray());
            TrustManagerFactory tmf = TrustManagerFactory.getInstance("X509", gmProvider);
            tmf.init(keyStore);

            SSLContext sc = SSLContext.getInstance("TLS", gmProvider);//TLS/TLCP etc.
            sc.init(kmf.getKeyManagers(), null, null); // tmf.getTrustManagers()
            return sc.getSocketFactory();
        } catch (Exception e) {
            e.printStackTrace();
        }

        return null;
    }
	
	@SuppressLint("BadHostnameVerifier")
    private boolean testHttpsUrlConnection(String hostUrl) {
        try {
            long p0 = System.currentTimeMillis();
            HttpsURLConnection conn = (HttpsURLConnection) (new URL(hostUrl)).openConnection();
            conn.setRequestMethod("GET");
            conn.addRequestProperty("Connection", "close");
            conn.setSSLSocketFactory(ssf);
            conn.setHostnameVerifier(new HostnameVerifier() {
                @Override
                public boolean verify(String hostname, SSLSession session) {
                    return true;
                }
            });
            long p1 = System.currentTimeMillis();
            conn.connect();
            long p2 = System.currentTimeMillis();
            byte[] bytes = getBytesFromStream(conn.getInputStream());
            conn.disconnect();
            long p3 = System.currentTimeMillis();
            showMessage(MSG_NORMAL, "Read:" + bytes.length + ", Time(ms): "
                    + (p1 - p0) + "+"
                    + (p2 - p1) + "+"
                    + (p3 - p2) + "="
                    + (p3 - p0));

            return true;
        } catch (Exception e) {
            e.printStackTrace();
            showMessage(MSG_NORMAL, e.getMessage());
        }

        return false;
    }

    private boolean testGMOkHttp(String hostUrl) {
        try {
            long p0 = System.currentTimeMillis();
            OkHttpClient okHttpClient = new OkHttpClient.Builder()
                    .sslSocketFactory(ssf, new TrustAll())
                    .build();
            Request request = new Request.Builder().url(hostUrl).get().build();
            long p1 = System.currentTimeMillis();
            Call call = okHttpClient.newCall(request);
            long p2 = System.currentTimeMillis();

            Response response = call.execute();
            long p3 = System.currentTimeMillis();
            showMessage(MSG_NORMAL, "Read:" + response.body().bytes().length + ", Time(ms): "
                    + (p1 - p0) + "+"
                    + (p2 - p1) + "+"
                    + (p3 - p2) + "="
                    + (p3 - p0));
            return true;
        } catch (IOException e) {
            e.printStackTrace();
            showMessage(MSG_NORMAL, e.getMessage());
        }

        return false;
    }