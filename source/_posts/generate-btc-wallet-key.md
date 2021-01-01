title: 学习 btc 钱包私钥、公钥和地址的生成过程
tags:
  - bitcoin
  - 比特币
  - ECDSA
  - 椭圆曲线
  - 加密
  - Crypto
categories:
  - Bitcoin
  - Wallet
  - Crypto
author: Steven Hu
date: 2018-09-26 16:20:00
---

一个 Bitcoin 钱包包含了一系列的密钥对，每个密钥对都是由一对公钥(public key)和私钥(private key)组成。私钥(k)通常是随机选出的一串数字串，之后我们就可以通过椭圆曲线密码学(ECC)算法来产生一个公钥(K)，然后再通过单向的 Hash 算法来生成 Bitcoin 地址。

如下图所示，描述了生成过程及主要的算法，以及整个过程的每一步都是不可逆的。

{% asset_img generate-process.jpg %}

# 如何生成私钥(private key)

本质上私钥就是一串随机选出的 256 个 bit 的 01 数字（32 字节 * 8 = 256 bits），但是这串数字却控制着你的比特币账号的所有权，因此这串数字相当重要，要具有足够的随机性，一般采用密码学安全的**伪随机数生成器(CSPNG)**，并且需要有一个来自具有足够熵值的源的种子(seed)。

<!-- more -->

> 为什么会选择 32 个字节？因为 Bitcoin 使用的是 ECDSA 算法，并且使用的是 secp256k1 曲线。

譬如：对于 Java 实现我们可以使用 **java.security.SecureRandom** 来生成随机数，如下为 SecureRandom 的默认构造方法，没有设置 seed，使用缺省的（支持 RNG 算法的）provider 来生成：

```java
    /**
     * Constructs a secure random number generator (RNG) implementing the
     * default random number algorithm.
     *
     * <p> This constructor traverses the list of registered security Providers,
     * starting with the most preferred Provider.
     * A new SecureRandom object encapsulating the
     * SecureRandomSpi implementation from the first
     * Provider that supports a SecureRandom (RNG) algorithm is returned.
     * If none of the Providers support a RNG algorithm,
     * then an implementation-specific default is returned.
     *
     * <p> Note that the list of registered providers may be retrieved via
     * the {@link Security#getProviders() Security.getProviders()} method.
     *
     * <p> See the SecureRandom section in the <a href=
     * "{@docRoot}openjdk-redirect.html?v=8&path=/technotes/guides/security/StandardNames.html#SecureRandom">
     * Java Cryptography Architecture Standard Algorithm Name Documentation</a>
     * for information about standard RNG algorithm names.
     *
     * <p> The returned SecureRandom object has not been seeded.  To seed the
     * returned object, call the {@code setSeed} method.
     * If {@code setSeed} is not called, the first call to
     * {@code nextBytes} will force the SecureRandom object to seed itself.
     * This self-seeding will not occur if {@code setSeed} was
     * previously called.
     */
    public SecureRandom() {
        /*
         * This call to our superclass constructor will result in a call
         * to our own {@code setSeed} method, which will return
         * immediately when it is passed zero.
         */
        super(0);
        getDefaultPRNG(false, null);
    }
```

也可以指定 SecureRandomSpi 实现类(implementation)和 provider，以及算法：

```java
    /**
     * Creates a SecureRandom object.
     *
     * @param secureRandomSpi the SecureRandom implementation.
     * @param provider the provider.
     */
    protected SecureRandom(SecureRandomSpi secureRandomSpi,
                           Provider provider) {
        this(secureRandomSpi, provider, null);
    }

    private SecureRandom(SecureRandomSpi secureRandomSpi, Provider provider,
            String algorithm) {
        super(0);
        this.secureRandomSpi = secureRandomSpi;
        this.provider = provider;
        this.algorithm = algorithm;

        // BEGIN Android-removed: this debugging mechanism is not supported in Android.
        /*
        if (!skipDebug && pdebug != null) {
            pdebug.println("SecureRandom." + algorithm +
                " algorithm from: " + this.provider.getName());
        }
        */
        // END Android-removed
    }
```

> 上面源码是 Android 中 Java 的源码实现，因此会出现 // BEGIN Android-removed 等移除 Java 原始实现代码的注释。


下面是使用缺省构造生成的随机数示例：

```java
    public static String generateRandomSeed(int seedLen) {
        SecureRandom secureRandom = new SecureRandom();
        byte[] seed = new byte[seedLen];
        secureRandom.nextBytes(seed);
        String seedHexStr = HexUtils.encodeHexString(seed);
        Log.i(TAG, "seed is " + seedHexStr);
        return seedHexStr;
    }
```

以下是上面的函数生成的一个随机的以 16 进制串表示的私钥：

```
f7387afb5ae15aac9acccc8d8823c8a1c2443b72a042b2274fc7433b4f0dd2f4
```

也可以通过 bitcoin-cli 来生成一个私钥和地址，如下：

```shell
$ bitcoin-cli -testnet --datadir=/var/bitcoin/testnet getnewaddress
2NBqPd4bmXEXXt3SqjxdMEYv3q46j6KMszq

$ bitcoin-cli -testnet --datadir=/var/bitcoin/testnet dumpprivkey 2NBqPd4bmXEXXt3SqjxdMEYv3q46j6KMszq
cQmW1c27ZFYy6XQxvUD6yMgbTm4cDSJYaaCM8NPKZZA4oMMdGB5X

$ bx base58check-decode cQmW1c27ZFYy6XQxvUD6yMgbTm4cDSJYaaCM8NPKZZA4oMMdGB5X
wrapper
{
    checksum 2258650717
    payload 5f115a837d4c782cd0e5214fe67a0434895dd7264e9949c40fe1cc129f1d9d2701
    version 239
}
```

其中：

  - **getnewaddress** 命令来生成钱包地址（内部已生成和保存了私钥）；
  - **dumpprivkey** 命令输出对应钱包的 Base58Check 的 WIF 钱包导入格式的私钥；
  - 最后一行命令是将解码 Base58Check 格式的私钥至 16 进制格式

> 注：其中的 -testnet --datadir 等选项参数是在测试网络里使用的。使用主网的可以忽略这些选项参数。


# 如何生成公钥

Bitcoin 的公钥是通过 椭圆曲线密码学算法（K = k * G）来生成，其中公式中的：

  - **K**：公钥；
  - **k**：私钥，为上一段生成的 32 字节的字节数组（16 进制串表示）；
  - **G**：为一个生成点；

Bitcoin 使用了 secp256k1 标准定义的一种特殊的椭圆曲线和一系列的数学常量。如上公式，以私钥 k 为起点，与预定的生成点 G 相乘来生成公钥 K，并且因为所有 Bitcoin 用户的生成点 G 都是相同的（常量），所以由一个确定的私钥 k 生成一个确定的公钥 K，并且是单向的。

> [Bitcoin 中使用的 secp256k1 标准细节参考](https://en.bitcoin.it/wiki/Secp256k1)
> [详细的椭圆曲线公式的数学解释与参考](https://hackernoon.com/what-is-the-math-behind-elliptic-curve-cryptography-f61b25253da3)

下面为使用 spongycastle 库中提供的 EC 算法库来生成公钥，分别支持 16 进制串和字节数组格式的私钥：

```java
    public static String generatePublicKey(byte [] privateKey, boolean compressed) {
        ECNamedCurveParameterSpec spec = ECNamedCurveTable.getParameterSpec("secp256k1");
        ECPoint pointQ = spec.getG().multiply(new BigInteger(1, privateKey));
        byte [] publicKeyBytes = pointQ.getEncoded(compressed);
        String publicKeyHexStr = HexUtils.encodeHexString(publicKeyBytes);
        Log.i(TAG, "==> public key is 0x" + publicKeyHexStr);
        return publicKeyHexStr;
    }

    public static String generatePublicKey(String privateKeyHexStr, boolean compressed) throws HexDecodeException {
        byte [] privateKeyBytes = HexUtils.decodeHex(privateKeyHexStr);
        return generatePublicKey(privateKeyBytes, compressed);
    }
```

如下为一个生成示例(16 进制串表示，且手工加上了 0x 前缀)：

```
private key: de97fdbdb823a197603e1f2cb8b1bded3824147e88ebd47367ba82d4b5600d73
public key:  047c91259636a5a16538e0603636f06c532dd6f2bb42f8dd33fa0cdb39546cf449612f3eaf15db9443b7e0668ef22187de9059633eb23112643a38771c630db911
```

## 压缩格式的公钥

从上面的输出示例中可以看到 public key 一共有 130 个 16 进制的字符，共 520 个字节，其中的前缀为 **04**，这里的 04 表示该公钥为 **非压缩格式**，即完整存储了 x 和 y 坐标（各 256 个 bits），但是从 secp256k1 的椭圆曲线方式可以看到，只要知道其中一个坐标值，另外一个坐标值都是可以通过解方程得出的，因为可以只存储其中一个坐标，这样就可以节约 256 个 bits，从而引入了 **压缩格式** 的公钥。

上面的 04 前缀表示 **非压缩格式**，如果为压缩格式，则前缀为 **02** 或 **03**，有两个前缀主要是因为方程（y² = x³ + ax + b）的左侧的 y 为平方根，可能为正或者为负。

在上面提供的 **generatePublicKey** 方法中支持是否输出压缩格式，如下为一个与上面示例对应的压缩格式的公钥值：

```
private key: de97fdbdb823a197603e1f2cb8b1bded3824147e88ebd47367ba82d4b5600d73
public key compressed: 037c91259636a5a16538e0603636f06c532dd6f2bb42f8dd33fa0cdb39546cf449
```

# 如何生成地址

Bitcoin 的地址由公钥经过单向的加密哈希算法 SHA256 和 RIPEMD160 生成，公式如下：

```
A = RIPEMD160(SHA256(K))
```

其中：
    - **K** 为公钥
    - **A** 为最终生成的地址；

下面为根据 公钥 生成的 bitcoin 钱包的地址(16 进制表示)的代码：

```java
    // RIPEMD160 ( SHA256 (publicKey) )
    public static String generateAddress(byte [] publicKey) throws NoSuchAlgorithmException {

        MessageDigest digest = MessageDigest.getInstance("SHA-256");
        byte [] pubKeySha256 = digest.digest(publicKey);
        Log.i(TAG, "==> sha256: 0x" + HexUtils.encodeHexString(pubKeySha256));

        digest = MessageDigest.getInstance("RIPEMD160");
        byte [] bytesAddr = digest.digest(pubKeySha256);
        Log.i(TAG, "==> sha256: 0x" + HexUtils.encodeHexString(bytesAddr));

        return HexUtils.encodeHexString(bytesAddr);
    }

    public static String generateAddress(String publicKeyHex) throws HexDecodeException, NoSuchAlgorithmException {
        return generateAddress(HexUtils.decodeHex(publicKeyHex));
    }
```

如下为生成的地址示例，地址的长度为 40 个 16 进制串，即 160 个bits：

```
private key: de97fdbdb823a197603e1f2cb8b1bded3824147e88ebd47367ba82d4b5600d73
public key compressed: 037c91259636a5a16538e0603636f06c532dd6f2bb42f8dd33fa0cdb39546cf449
address: 52dab5e951ef4848a31b7ead8437df8184acbc54
```

# Base58, Base58Check 以及压缩格式

我们通常看到的 Bitcoin 地址都是经过 Base58Check 编码后的地址，Base58Check 编码也用于私钥，加密的密钥以及脚本中，用来提高可读性和录入的正确性。下图描述了通过 公钥生成 Base58Check 编码格式的地址的整个过程：

{% asset_img generate-addr-from-pubkey.jpg %}

其中 Public Key Hash 我们在上面已经生成的地址，之后就是通过 Base58Check 编码生成 Bitcoin 的地址格式。

（摘自 wiki）相比Base64，Base58不使用数字"0"，字母大写"O"，字母大写"I"，和字母小写"l"，以及"+"和"/"符号。
设计Base58主要的目的是：

- 避免混淆。在某些字体下，数字0和字母大写O，以及字母大写I和字母小写l会非常相似。
- 不使用"+"和"/"的原因是非字母或数字的字符串作为帐号较难被接受。
- 没有标点符号，通常不会被从中间分行。
- 大部分的软件支持双击选择整个字符串。

如下为 Base58 符号映射表：

{% asset_img base58-symbol-chart.jpg %}

为了进一步增加安全性，Base58Check 格式又在 Base58 的基础上新增了内置检查错误的校验和（checksum），该校验和是添加到末尾的额外 4 个字节，校验和的生成算法如下：

```
checksum = SHA256(SHA256(prefix+data))
```

- **data** 为原始的数据；
- **prefix** 前缀是个版本字段，是用来识别编码的数据的类型，如：Bitcoin 地址（也是 public key hash）的前缀为 0（即 0x00），当前支持的如下类型：

{% asset_img base58check-version-prefix.jpg %}

> 完整的 [address prefix 参考](https://en.bitcoin.it/wiki/List_of_address_prefixes)

其中 私钥 的前缀为 128（即 0x80），对应的编码后前缀为 5，如下为我们的私钥编码后的：

```shell
$ bx base58check-encode --version 128
de97fdbdb823a197603e1f2cb8b1bded3824147e88ebd47367ba82d4b5600d73
5KWKSRnmzxCjUP1NKR4dNyyHhaZWSGRTbGzBnm1vwgwpoe2AVGQ
```

下图为 Base58Check 的编码过程：

{% asset_img base58-encoding-process.jpg %}

在 bitcoin 中大多数需要向用户展示的数据都是使用的 Base58Check 编码格式。

如下为根据 public key 生成 address 的实现代码：

```java
    static byte[] generateBase58CheckSum(byte[] data) throws NoSuchAlgorithmException {
        MessageDigest digest = MessageDigest.getInstance("SHA-256");
        byte [] dataOneHash = digest.digest(data);
        byte [] dataDoubleHash = digest.digest(dataOneHash);
        byte [] checkSum = Arrays.copyOf(dataDoubleHash, 4);
        Log.i(TAG, "==> base58check sum: " + HexUtils.encodeHexString(checkSum));
        return checkSum;
    }

    // Base58Check(RIPEMD160(SHA256(publicKey))
    public static String generateAddressWithBase58Check(byte [] publicKey) throws NoSuchAlgorithmException, HexDecodeException {
        String addrHex = generateAddress(publicKey);
        byte [] addrBytes = HexUtils.decodeHex(addrHex);
        byte [] checksum = generateBase58CheckSum(addrBytes);
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        baos.write(0);  // address version prefix
        baos.write(addrBytes, 0, addrBytes.length);
        baos.write(checksum, 0, checksum.length);
        String addressWithBase58Check = Base58.encode(baos.toByteArray());
        Log.i(TAG, "==> address with base58check format: " + addressWithBase58Check);
        return addressWithBase58Check;
    }
```

下面为私钥、公钥以及生成的 Base58Check 格式的地址信息：


```
private key hex: de97fdbdb823a197603e1f2cb8b1bded3824147e88ebd47367ba82d4b5600d73
private key base58check: 5KWKSRnmzxCjUP1NKR4dNyyHhaZWSGRTbGzBnm1vwgwpoe2AVGQ
public key compressed: 037c91259636a5a16538e0603636f06c532dd6f2bb42f8dd33fa0cdb39546cf449
checksum: 4caf1695
base58check address: 18Z6R1VF7Do8RTHneeGzdVdbgjtXDVPmfS
```

# 完整示例代码

如下为上面示例中生成私钥, 公钥以及地址的完整代码：

```java

public class BitcoinUtils {

    public static final String TAG = BitcoinUtils.class.getName();
    public static final boolean IS_PRODUCTION = false;

    public static NetworkParameters getNetParams() {
        return (IS_PRODUCTION ? MainNetParams.get() : TestNet3Params.get());
    }

    public static String generateRandomSeed(int seedLen) {
        SecureRandom secureRandom = new SecureRandom();
        byte[] seed = new byte[seedLen];
        secureRandom.nextBytes(seed);
        String seedHexStr = HexUtils.encodeHexString(seed);
        Log.i(TAG, "==> seed is 0x" + seedHexStr);
        return seedHexStr;
    }

    public static String generatePublicKey(byte [] privateKey, boolean compressed) {
        ECNamedCurveParameterSpec spec = ECNamedCurveTable.getParameterSpec("secp256k1");
        ECPoint pointQ = spec.getG().multiply(new BigInteger(1, privateKey));
        byte [] publicKeyBytes = pointQ.getEncoded(compressed);
        String publicKeyHexStr = HexUtils.encodeHexString(publicKeyBytes);
        Log.i(TAG, "==> public key is 0x" + publicKeyHexStr);
        return publicKeyHexStr;
    }

    public static String generatePublicKey(String privateKeyHexStr, boolean compressed) throws HexDecodeException {
        byte [] privateKeyBytes = HexUtils.decodeHex(privateKeyHexStr);
        return generatePublicKey(privateKeyBytes, compressed);
    }

    // RIPEMD160 ( SHA256 (publicKey) )
    public static String generateAddress(byte [] publicKey) throws NoSuchAlgorithmException {

        MessageDigest digest = MessageDigest.getInstance("SHA-256");
        byte [] pubKeySha256 = digest.digest(publicKey);
        Log.i(TAG, "==> sha256: 0x" + HexUtils.encodeHexString(pubKeySha256));

        RIPEMD160Digest d = new RIPEMD160Digest();
        d.update(pubKeySha256, 0, pubKeySha256.length);
        byte [] bytesAddr = new byte[d.getDigestSize()];
        d.doFinal(bytesAddr, 0);
        Log.i(TAG, "==> ripemd160: 0x" + HexUtils.encodeHexString(bytesAddr));

        return HexUtils.encodeHexString(bytesAddr);
    }

    public static String generateAddress(String publicKeyHex) throws HexDecodeException, NoSuchAlgorithmException {
        return generateAddress(HexUtils.decodeHex(publicKeyHex));
    }

    static byte[] generateBase58CheckSum(byte[] data) throws NoSuchAlgorithmException {
        MessageDigest digest = MessageDigest.getInstance("SHA-256");
        byte [] dataOneHash = digest.digest(data);
        byte [] dataDoubleHash = digest.digest(dataOneHash);
        byte [] checkSum = Arrays.copyOf(dataDoubleHash, 4);
        Log.i(TAG, "==> base58check sum: " + HexUtils.encodeHexString(checkSum));
        return checkSum;
    }

    // Base58Check ( RIPEMD160 ( SHA256 (publicKey) )
    public static String generateAddressWithBase58Check(byte [] publicKey) throws NoSuchAlgorithmException, HexDecodeException {
        String addrHex = generateAddress(publicKey);
        byte [] addrBytes = HexUtils.decodeHex(addrHex);
        byte [] checksum = generateBase58CheckSum(addrBytes);
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        baos.write(0);  // address version prefix
        baos.write(addrBytes, 0, addrBytes.length);
        baos.write(checksum, 0, checksum.length);
        String addressWithBase58Check = Base58.encode(baos.toByteArray());
        Log.i(TAG, "==> address with base58check format: " + addressWithBase58Check);
        return addressWithBase58Check;
    }

    public static String generateAddressWithBase58Check(String publicKeyHex) throws HexDecodeException, NoSuchAlgorithmException {
        return generateAddressWithBase58Check(HexUtils.decodeHex(publicKeyHex));
    }
}
```

# 参考资料

- **[Master Bitcoin 2nd](https://github.com/bitcoinbook/bitcoinbook)**
- **[Bitcoin developer guide](https://bitcoin.org/en/developer-guide)**
- **[Base58 Wiki](https://zh.wikipedia.org/wiki/Base58)**
- **[Base58Check Wiki](https://en.bitcoin.it/wiki/Base58Check_encoding)**

