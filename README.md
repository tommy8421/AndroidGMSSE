# AndroidGMSSE
	基于Java实现的国密TLCP套件（GB/T 38636-2020）适用于各个版本安卓，简单修改后也可以用于其他JAVA应用
	目前只实现了ECDHE_SM4_CBC_SM3和ECC_SM4_CBC_SM3，对应的GCM还没有实现（暂时不会），后续考虑参考Kona
# 用法
## Android
	直接使用编译的aar

## Java
	提取aar中的jar，然后实现下安卓下面的Log和TextUtils.isEmpty(CharSequence string)即可
	
# 样例代码
	private static final String[] TEST_HOST_URL = new String[]{
            "https://demo.gmssl.cn:443",
            "https://demo.gmssl.cn:1443",
            "https://demo.gmssl.cn:2443",
    };
	
	private static final GMProvider gmProvider = new GMProvider();
    private static final BouncyCastleProvider bcProvider = new BouncyCastleProvider();
	
	private SSLSocketFactory initTestGMSSL() {
        try {
            GMProvider.setEnableLog(false);
            GMProvider.setCipherSuites(new GMCipherSuite[]{GMCipherSuite.ECDHE_SM4_CBC_SM3, GMCipherSuite.ECC_SM4_CBC_SM3});

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
	
	private boolean testGMSSL(int testId, String hostUrl, SSLSocketFactory ssf) {
        try {
            long p0 = System.currentTimeMillis();
            URL serverUrl = new URL(hostUrl);
            HttpsURLConnection conn = (HttpsURLConnection) serverUrl.openConnection();
            conn.setRequestMethod("GET");
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
            Log.d("testGMSSL", "used cipher suite:" + conn.getCipherSuite());
            byte[] bytes = getBytesFromStream(conn.getInputStream());
            conn.disconnect();
            long p3 = System.currentTimeMillis();
            Log.d(TAG, hostUrl + "===>"
                    + (p1 - p0) + "+"
                    + (p2 - p1) + "+"
                    + (p3 - p2) + "="
                    + (p3 - p0));
            return true;
        } catch (Exception e) {
            e.printStackTrace();
        }

        return false;
    }

