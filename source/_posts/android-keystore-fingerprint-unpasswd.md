title: 通过 Android keystore 和 fingerprint 结合实现数据加密和解密
tags:
  - Android
  - Keystore
  - Fingerprint
  - 指纹
  - 免密
  - 加密
  - Crypto
categories:
  - Algorithm
  - Crypto
author: Steven Hu
date: 2019-02-24 20:20:20
---

本文目标是通过结合 Android Keystore 和 Fingerprint 来安全的对数据进行加密，并且能够通过指纹身份验证之后对数据进行解密。

# 了解 Keystore

Android Keystore 系统可以在一个安全的容器中（如：借助于系统芯片中提供的可信执行环境 TEE）存储加密密钥，在我们的加密密钥进入 Keystore 之后，可以在不用导出密钥的前提下完成加密操作，Keystore 支持的操作有：

- 生成密钥
- 导入和导出非对称密钥
- 导入原始对称密钥
- 使用适当的填充模式（padding modes）进行非对称加密和解密
- 使用摘要和适当的填充模式（padding modes）进行非对称签名和验证
- 以适当模式（包括 AEAD 模式）进行对称加密和解密
- 生成和验证对称消息验证码(Message Authentication Codes, MAC)

<!-- more -->

另外，我们可以结合身份验证系统（如指纹验证）来实现只有在用户完成身份验证之后才能使用密钥，即可以让应用指定密钥的授权使用方式，一旦生成或导入密钥，授权方式将无法更改，并且之后每次需要使用密钥时，都会由 Android Keystore 库强制执行授权。

> 本文的代码实现主要基于 Android 6.0 以上，因为从 Android 6.0 开始不仅增加了对称加密算法（AES 和 HMAC）的支持，还增加了针对硬件支持的密钥访问控制系统，访问控制的逻辑会在密钥生成期间指定，并在整个生命周期内被强制执行。
> 
> 其他参考: 
> 
> - [Keystore 功能](https://source.android.com/security/keystore/features)
> - [Keymaster 授权标记](https://source.android.com/security/keystore/tags)

我们下面就接着先看下 Android 的身份验证。

# 身份验证

在 Android 中，主要通过 Gatekeeper（用于 PIN 码/解锁图案/密码身份验证）和 Fingerprint（用于指纹身份验证）来实现身份验证。

并且 Gatekeeper 和 Fingerprint 会通过与 Keystore 组件进行沟通身份验证的状态。

> 从 Android 9 开始使用 BiometricPrompt 统一封装了指纹和其他生物识别技术。

下图展示了 Fingerprint，Keystore 等等一起参与的整个的身份验证流程：

{% asset_img authentication-flow.png %}

1. 用户可以通过 PIN/Pattern/Password 或 指纹 方式进行验证：
    - 对于 PIN/解锁图案/密码，主要通过 LockSettingsService 向 gatekeeperd 发出验证请求；
    - 对于 指纹，会通过 FingerprintService 向 fingerprintd 发起验证请求；  
2. gatekeeperd/fingerprintd 守护进程会将待验证数据发送至 TEE 中的副本 gatekeeper/fingerprint，并由相关副本生成 AuthToken（并使用了 AuthToken HMAC 签名）并发送到 gatekeeperd/fingerprintd 守护进程；
3. 守护进程 gatekeeperd/fingerprintd 收到经过签名的 AuthToken 之后，再通过 KeyStore 服务 Binder 接口的扩展程序将 AuthToken 传入 Keystore Service；
4. Keystore Service 将 AuthToken 传给 TEE 中的 Keymaster，并使用与 gatekeeper 和 fingerprint 共用的密钥进行验证；

# 关于 Fingerprint

下面我们具体看下指纹，现在很多的手机上都已配备有指纹传感器，在这些手机上我们可以注册一个或多个指纹，然后使用指纹来解锁手机或执行其他任务（例如：很多涉及转账类的 App 中的免密支付功能），Android 通过利用 Fingerprint HAL 来连接到供应商的专用库（FP vendor library）和指纹传感器。

指纹匹配的流程是：

1. 用户将“注册”过的手指放在指纹传感器上；
2. 供应商专用库会根据当前已注册的指纹模板集进行匹配；
3. 匹配结果传至 Fingerprint HAL，然后进一步将指纹身份验证结果通知给 Fingerprint 守护进程 fingerprintd；

指纹身份验证的概要交互数据流程如下图：

{% asset_img fingerprint-data-flow.png %}

- 每个应用都可以有一个 FingerprintManager 实例，该实例负责与 FingerprintService 进行通信；而 FingerprintService 为在系统中运行的单例服务，负责与 fingerprintd 守护进行通信；
- fingerprintd 守护进程会封装和调用 Fingerprint HAL 供应商的专用库，来实现指纹的注册、验证等等操作，如下图：

{% asset_img fingerprint-daemon.png %}

# 回到示例

上面大致了解了 Keystore，身份验证流程以及指纹识别匹配的流程，下面看下如何结合 Keystore 和 Fingerprint 来实现数据的加密和解密：

## 生成(对称)密钥

首先我们需要通过 Keystore 库来生成（用于后续对数据进行加密的）密钥，这里采用 AES 加密算法，并且设置这个密钥为强制授权访问方式。

这里通过 KeyStore 和 KeyGenerator 类，以及 AndroidKeyStore 提供程序来生成密钥：

```java
public void generateKey(View view) {

    // AES + CBC + PKCS7
    final KeyGenerator generator = KeyGenerator.getInstance(KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore");
    mKeyStore.load(null);
    generator.init(new KeyGenParameterSpec.Builder("FirstWallet",
            KeyProperties.PURPOSE_ENCRYPT | KeyProperties.PURPOSE_DECRYPT)
            .setUserAuthenticationRequired(true)
            .setBlockModes(KeyProperties.BLOCK_MODE_CBC)
            .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_PKCS7)
            .build());

    // Generate (symmetric) key, and store to KeyStore
    final SecretKey sk = generator.generateKey();
    Toast.makeText(this, String.format("Generate key success %s", sk.getAlgorithm()), Toast.LENGTH_LONG).show();
}
```

在生成的过程中设置了要求用户身份验证的选项，后续步骤中的加密和解密步骤我们可以通过指纹识别来完成用户身份验证并通过相应的密钥进行加解密。

其中 mKeyStore Provider 在初始时已经生成，如下代码：

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    try {
        // Register provider
        mKeyStore = KeyStore.getInstance("AndroidKeyStore");
    } catch (KeyStoreException e) {
        e.printStackTrace();
        finish();
        return;
    }

    // Fingerprint service
    mFpManager = (FingerprintManager) getSystemService(Context.FINGERPRINT_SERVICE);
    if (null == mFpManager) {
        finish();
        return;
    }

    ...
}
```

这里的 **AndroidKeyStore** 是各个 App 需要访问 Keystore 功能的 Android Framework API 和组件，也是作为 JCA API 的扩展程序实现的，是在各个 App 自有进程空间中运行的，

另外，这里也获取了与 FingerprintService 交互的 FingerprintManager 实例。

## 指纹加密

通过指纹识别验证通过后，就可以使用存储在 Keystore 中对应的 key （通过生成时指定的 key 名称获取）对内容进行加密：

```java
public void encryptData(View view) throws CertificateException, NoSuchAlgorithmException, IOException, UnrecoverableKeyException, KeyStoreException, NoSuchPaddingException, InvalidKeyException {

    mKeyStore.load(null);
    final SecretKey sk = (SecretKey) mKeyStore.getKey("FirstWallet", null);
    if (null == sk) {
        Toast.makeText(this, "Can not get key", Toast.LENGTH_LONG).show();
        return;
    }

    final Cipher cipher = Cipher.getInstance(KeyProperties.KEY_ALGORITHM_AES + "/" + KeyProperties.BLOCK_MODE_CBC + "/" + KeyProperties.ENCRYPTION_PADDING_PKCS7);
    cipher.init(Cipher.ENCRYPT_MODE, sk);

    // Need authenticate by fingerprint
    final FingerprintManager.CryptoObject cryptoObject = new FingerprintManager.CryptoObject(cipher);
    mFpManager.authenticate(cryptoObject, null, 0, new FingerprintManager.AuthenticationCallback() {
        @Override
        public void onAuthenticationError(int errorCode, CharSequence errString) {
            super.onAuthenticationError(errorCode, errString);
            Toast.makeText(MainActivity.this, "Fp auth error: " + errString, Toast.LENGTH_LONG).show();
        }

        @Override
        public void onAuthenticationHelp(int helpCode, CharSequence helpString) {
            super.onAuthenticationHelp(helpCode, helpString);
        }

        @Override
        public void onAuthenticationSucceeded(FingerprintManager.AuthenticationResult result) {
            super.onAuthenticationSucceeded(result);
            Toast.makeText(MainActivity.this, "Fp auth succ", Toast.LENGTH_LONG).show();

            // Encrypt data by cipher
            final String plainText = etPlain.getText().toString();
            final Cipher cipher = result.getCryptoObject().getCipher();
            try {
                byte [] encrypted = cipher.doFinal(plainText.getBytes());
                mIV = cipher.getIV();
                final String encryptedWithBase64 = Base64.encodeToString(encrypted, Base64.URL_SAFE);
                tvEncrypted.setText(encryptedWithBase64);
            } catch (IllegalBlockSizeException | BadPaddingException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onAuthenticationFailed() {
            super.onAuthenticationFailed();
            Toast.makeText(MainActivity.this, "Fp auth failed", Toast.LENGTH_LONG).show();
        }
    }, new Handler());
}
```

在上面代码中：

- 加密算法采用 AES+CBC+PKCS7；
- **mFpManager.authenticate** 等待用户验证指纹；
- 在指纹验证通过后回调 **onAuthenticationSucceeded** 方法，并且在输入参数 **AuthenticationResult result** 中携带加密算法对象，通过此加密算法对象完成加密工作；
- 这里还通过 **cipher.getIV()** 保存了本次加密时的初始化矢量(IV)；

## 指纹解密

在指纹验证通过后通过相同的密钥对上面加密的内容进行解密。

```java
    public void decryptWithFingerprint(View view) {

        mKeyStore.load(null);
        final SecretKey sk = (SecretKey) mKeyStore.getKey("FirstWallet", null);
        if (null == sk) {
            Toast.makeText(this, "Can not get key", Toast.LENGTH_LONG).show();
            return;
        }

        final Cipher cipher = Cipher.getInstance(KeyProperties.KEY_ALGORITHM_AES + "/" + KeyProperties.BLOCK_MODE_CBC + "/" + KeyProperties.ENCRYPTION_PADDING_PKCS7);
        cipher.init(Cipher.DECRYPT_MODE, sk, new IvParameterSpec(mIV));

        // First need authenticate by fingerprint
        final FingerprintManager.CryptoObject cryptoObject = new FingerprintManager.CryptoObject(cipher);
        mFpManager.authenticate(cryptoObject, null, 0, new FingerprintManager.AuthenticationCallback() {
            @Override
            public void onAuthenticationError(int errorCode, CharSequence errString) {
                super.onAuthenticationError(errorCode, errString);
                Toast.makeText(MainActivity.this, "Fp auth error: " + errString, Toast.LENGTH_LONG).show();
            }

            @Override
            public void onAuthenticationHelp(int helpCode, CharSequence helpString) {
                super.onAuthenticationHelp(helpCode, helpString);
            }

            @Override
            public void onAuthenticationSucceeded(FingerprintManager.AuthenticationResult result) {
                super.onAuthenticationSucceeded(result);
                Toast.makeText(MainActivity.this, "Fp auth succ", Toast.LENGTH_LONG).show();

                // Decrypt data by cipher
                final String encryptedWithBase64 = tvEncrypted.getText().toString();
                final byte [] encryptedBytes = Base64.decode(encryptedWithBase64, Base64.URL_SAFE);
                final Cipher cipher = result.getCryptoObject().getCipher();
                try {
                    byte [] decryptedBytes = cipher.doFinal(encryptedBytes);
                    String decryptedText = new String(decryptedBytes);
                    tvDecrypted.setText(decryptedText);
                } catch (IllegalBlockSizeException | BadPaddingException e) {
                    e.printStackTrace();
                }
            }

            @Override
            public void onAuthenticationFailed() {
                super.onAuthenticationFailed();
                Toast.makeText(MainActivity.this, "Fp auth failed", Toast.LENGTH_LONG).show();
            }
        }, new Handler());
    }
```

- 解密步骤中的 **cipher** 实例初始化时需要指定与加密时相同的 IV；

## 注意点

- 在用户重新设置指纹之后，之前在 Keystore 中生成的 key 将会失效，也就是需要在应用中重新生成新的 key；
- 加解密都需要获取同名的密钥；

> [本文相关示例代码](https://github.com/stevenocean/UnpasswdDecrypt)

