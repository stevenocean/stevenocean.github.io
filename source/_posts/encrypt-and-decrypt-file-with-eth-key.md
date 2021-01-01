title: 使用以太坊的公钥和私钥对数据加解密
tags:
  - ECDSA
  - Crypto
  - Ethereum
  - 以太坊
  - ECC
  - BouncyCastle
categories:
  - Algorithm
  - Crypto
author: Steven Hu
date: 2018-08-09 16:20:00
---

# 概述

本篇主要尝试了采用 Ethereum 生成的私钥和公钥来对数据进行加解密。在进入示例之前先简单了解一下 Ethereum 私钥和公钥的生成过程。

# 密钥对的生成

其实无论是 Ethereum 还是 Bitcoin，他们的私钥本质上就是一个 256 个二进制位的随机数字（2^256 ~ 10^77，目前可见宇宙中估计只含有 10^80 个原子），譬如：你可以自己选择 256 个 0 作为自己的私钥，但是这明显是不安全的，而要选择足够安全的随机数，也就是要找到足够安全的熵源，即：随机性来源，最好选择密码学安全的伪随机数生成器（CSPRNG）。下面我们看下 Ethereum(go-ethereum) 的源码中如何生成公私钥对的：

<!--more-->

```go
// cmd/ethkey/generate.go
var commandGenerate = cli.Command{
    Name:      "generate",
    Usage:     "generate new keyfile",
    ArgsUsage: "[ <keyfile> ]",
    ...
    Action: func(ctx *cli.Context) error {
        ...
        var privateKey *ecdsa.PrivateKey
        var err error
        if file := ctx.String("privatekey"); file != "" {
            // Load private key from file.
            privateKey, err = crypto.LoadECDSA(file)
            if err != nil {
                utils.Fatalf("Can't load private key: %v", err)
            }
        } else {
            // If not loaded, generate random.
            // 这里生成 私钥
            privateKey, err = crypto.GenerateKey()
            if err != nil {
                utils.Fatalf("Failed to generate random private key: %v", err)
            }
        }

        // 下面是生成 account address 和 keystore

        // Create the keyfile object with a random UUID.
        id := uuid.NewRandom()
        key := &keystore.Key{
            Id:         id,
            Address:    crypto.PubkeyToAddress(privateKey.PublicKey),
            PrivateKey: privateKey,
        }

        // Encrypt key with passphrase.
        passphrase := getPassPhrase(ctx, true)
        keyjson, err := keystore.EncryptKey(key, passphrase, keystore.StandardScryptN, keystore.StandardScryptP)
        if err != nil {
            utils.Fatalf("Error encrypting key: %v", err)
        }
        ...
    }
```

继续看下 crypto.GenerateKey 的实现：

```go
// crypto/crypto.go
func GenerateKey() (*ecdsa.PrivateKey, error) {
    return ecdsa.GenerateKey(S256(), rand.Reader)
}
```

这里直接调用了 go 的 ecdsa 库中的 [GenerateKey](https://golang.org/pkg/crypto/ecdsa/#GenerateKey) 方法来生成。

**[ECDSA](https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm)(Elliptic Curve Digital Signature Algorithm)** 是使用 **ECC(Elliptic Curve Cryptography)** 椭圆曲线加密算法对 **DSA** 数字签名算法的模拟，ECDSA 是 1999 年成为 ANSI 标准，并于 2000 年成为 IEEE 和 NIST 标准。与普通的离散对数问题（Discrete Logarithm Problem **DLP**）和大数分解问题（Integer Factorization Problem **IFP**）不同，椭圆曲线离散对数问题（Elliptic Curve Discrete Logarithm Problem **ECDLP**）没有亚指数时间的解决方法。因此椭圆曲线密码的单位比特强度要高于其他公钥体制。

在调用 ecdsa.GenerateKey 时，第一个参数为选择的椭圆曲线，这里为 secp256k1：

```go
var theCurve = new(BitCurve)

func init() {
    // See SEC 2 section 2.7.1
    // curve parameters taken from:
    // http://www.secg.org/collateral/sec2_final.pdf
    theCurve.P, _ = new(big.Int).SetString("FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFC2F", 16)
    theCurve.N, _ = new(big.Int).SetString("FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141", 16)
    theCurve.B, _ = new(big.Int).SetString("0000000000000000000000000000000000000000000000000000000000000007", 16)
    theCurve.Gx, _ = new(big.Int).SetString("79BE667EF9DCBBAC55A06295CE870B07029BFCDB2DCE28D959F2815B16F81798", 16)
    theCurve.Gy, _ = new(big.Int).SetString("483ADA7726A3C4655DA4FBFC0E1108A8FD17B448A68554199C47D08FFB10D4B8", 16)
    theCurve.BitSize = 256
}

// S256 returns a BitCurve which implements secp256k1.
func S256() *BitCurve {
    return theCurve
}
```

上面的几个常亮的定义都是 secp256k1 曲线函数 Fn 中的参数。

secp256k1 标准也是由美国国家标准和技术研究院 NIST 设立，secp256k1 的一些[细节参考](https://en.bitcoin.it/wiki/Secp256k1)

关于 **ECDSA** 的底层原理也可以参考这篇 [UNDERSTANDING HOW ECDSA PROTECTS YOUR DATA](https://www.instructables.com/id/Understanding-how-ECDSA-protects-your-data/)

# 示例

下面示例是采用公钥对一个原始的文本文件 source.txt 进行加密生成 cipher.txt，然后再使用 私钥 对加密后的文本文件 cipher.txt 进行解密生成 decrypt.txt。

加解密库基于 BouncyCastle 库（下面简称 BC 库），BC 库是一个开源的，支持 Java 和 C# 的安全库，Java 版本的实现中提供了 JCE 和 JCA 的 provider，读写 ASN.1 编码对象的库，OCSP 的生成/处理器，OpenPGP(RFC 2440)的生成/处理器等等，提供了大量的加密算法，包括 椭圆曲线，AES 等等算法。

> 注：里面很多常量都是 secp256k1 中定义的。

下面为完整的使用 公钥加密文件，并采用私钥解密文件的示例源码：

```java
import org.bouncycastle.crypto.params.ECDomainParameters;
import org.bouncycastle.crypto.params.ECPublicKeyParameters;
import org.bouncycastle.jcajce.provider.asymmetric.ec.BCECPrivateKey;
import org.bouncycastle.jcajce.provider.asymmetric.ec.BCECPublicKey;
import org.bouncycastle.jce.provider.BouncyCastleProvider;
import org.bouncycastle.jce.spec.ECNamedCurveSpec;
import org.bouncycastle.jce.spec.IESParameterSpec;
import org.bouncycastle.math.ec.custom.sec.SecP256K1Curve;
import org.bouncycastle.math.ec.custom.sec.SecP256K1FieldElement;
import org.bouncycastle.math.ec.custom.sec.SecP256K1Point;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;
import java.math.BigInteger;
import java.security.*;
import java.security.spec.*;

import javax.crypto.Cipher;
import javax.crypto.CipherInputStream;
import javax.crypto.CipherOutputStream;
import javax.crypto.NoSuchPaddingException;


public class EncryptWithPubKey {

    public static void main(String [] args) throws NoSuchProviderException, NoSuchAlgorithmException, InvalidAlgorithmParameterException, NoSuchPaddingException, InvalidKeyException, IOException {

        // ECDSA secp256k1 algorithm constants
        BigInteger pointGPre = new BigInteger("79be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798", 16);
        BigInteger pointGPost = new BigInteger("483ada7726a3c4655da4fbfc0e1108a8fd17b448a68554199c47d08ffb10d4b8", 16);
        BigInteger factorN = new BigInteger("fffffffffffffffffffffffffffffffebaaedce6af48a03bbfd25e8cd0364141", 16);
        BigInteger fieldP = new BigInteger("fffffffffffffffffffffffffffffffffffffffffffffffffffffffefffffc2f", 16);

        Security.addProvider(new BouncyCastleProvider());
        Cipher cipher = Cipher.getInstance("ECIES", "BC");
        IESParameterSpec iesParams = new IESParameterSpec(null, null, 64);

        //----------------------------
        // Encrypt with public key
        //----------------------------

        // public key for test
        String publicKeyValue = "30d67dc730d0d253df841d82baac12357430de8b8f5ce8a35e254e1982c304554fdea88c9cebdda72bf6be9b14aa684288eb3ba6f9cb7b7872b6e41d2b9706fc";
        String prePublicKeyStr = publicKeyValue.substring(0, 64);
        String postPublicKeyStr = publicKeyValue.substring(64);

        EllipticCurve ellipticCurve = new EllipticCurve(new ECFieldFp(fieldP), new BigInteger("0"), new BigInteger("7"));
        ECPoint pointG = new ECPoint(pointGPre, pointGPost);
        ECNamedCurveSpec namedCurveSpec = new ECNamedCurveSpec("secp256k1", ellipticCurve, pointG, factorN);

        // public key
        SecP256K1Curve secP256K1Curve = new SecP256K1Curve();
        SecP256K1Point secP256K1Point = new SecP256K1Point(secP256K1Curve, new SecP256K1FieldElement(new BigInteger(prePublicKeyStr, 16)), new SecP256K1FieldElement(new BigInteger(postPublicKeyStr, 16)));
        SecP256K1Point secP256K1PointG = new SecP256K1Point(secP256K1Curve, new SecP256K1FieldElement(pointGPre), new SecP256K1FieldElement(pointGPost));
        ECDomainParameters domainParameters = new ECDomainParameters(secP256K1Curve, secP256K1PointG, factorN);
        ECPublicKeyParameters publicKeyParameters = new ECPublicKeyParameters(secP256K1Point, domainParameters);
        BCECPublicKey publicKeySelf = new BCECPublicKey("ECDSA", publicKeyParameters, namedCurveSpec, BouncyCastleProvider.CONFIGURATION);

        // begin encrypt

        cipher.init(Cipher.ENCRYPT_MODE, publicKeySelf, iesParams);
        String cleartextFile = "test/source.txt";
        String ciphertextFile = "test/cipher.txt";
        byte[] block = new byte[64];
        FileInputStream fis = new FileInputStream(cleartextFile);
        FileOutputStream fos = new FileOutputStream(ciphertextFile);
        CipherOutputStream cos = new CipherOutputStream(fos, cipher);

        int i;
        while ((i = fis.read(block)) != -1) {
            cos.write(block, 0, i);
        }
        cos.close();

        //----------------------------
        // Decrypt with private key
        //----------------------------

        // private key for test, match with public key above
        BigInteger privateKeyValue = new BigInteger("eb06bde0e1e9427b3e23ab010a124e8cea0d9242b5406eff295f0a501b49db3", 16);

        ECPrivateKeySpec privateKeySpec = new ECPrivateKeySpec(privateKeyValue, namedCurveSpec);
        BCECPrivateKey privateKeySelf = new BCECPrivateKey("ECDSA", privateKeySpec, BouncyCastleProvider.CONFIGURATION);

        // begin decrypt
        String cleartextAgainFile = "test/decrypt.txt";
        cipher.init(Cipher.DECRYPT_MODE, privateKeySelf, iesParams);
        fis = new FileInputStream(ciphertextFile);
        CipherInputStream cis = new CipherInputStream(fis, cipher);
        fos = new FileOutputStream(cleartextAgainFile);
        while ((i = cis.read(block)) != -1) {
            fos.write(block, 0, i);
        }
        fos.close();
    }
}
```

在 Cipher.getInstance 方法中第一个参数指定加密算法（Cipher Algorithm Name)，这里使用了 **[ECIES(Elliptic curve integrated encryption scheme)](https://cryptopp.com/wiki/Elliptic_Curve_Integrated_Encryption_Scheme)** 算法标准，**ECIES** 是一种集成了 KEM(Key Encapsulation Mechanism，密钥封装机制) 和 DEM(Data Encapsulation Mechanism，数据封装机制) 的混合加密系统规范（参考：ANSI X9.63, IEEE 1363a, ISO/IEC 18033-2, SECG SEC-1）；第二个参数指定 Provider，这里的 "BC" 就是 BouncyCastle Provider。

IESParameterSpec 为 IES 的算法引擎参数设置，这里的三个参数分别为：

- derivation: 可选，用于 KDF 的推导向量；
- encoding: 可选，用于 KDF 的编码向量；
- macKeySize: 设置 MAC 的 key size，这里设置为 64 个 bits；

> **[KDF(Key Derivation Function)](https://en.wikipedia.org/wiki/Key_derivation_function)** derives one or more secret keys from a secret value such as a master key, a password, or a passphrase using a pseudorandom function.

下图为加密之前的 source.txt 文本文件：

{% asset_img clear-source.jpg %}

下图为通过 public key 加密之后的 encrypt.txt 文本文件，非文本格式，无法识别：

{% asset_img encrypt-source.jpg %}

下图为通过 private key 对 encrypt.txt 解密之后的 decrypt.txt 文本文件，与原始的 source.txt 一致：

{% asset_img decrypt-source.jpg %}

# 常见术语

- **EC**, Elliptic Curve, 椭圆曲线
- **ECC**, Elliptic Curve Cryptography, 椭圆曲线密码学
- **ECDSA**, Elliptic Curve Digital Signature Algorithm, 椭圆曲线数字签名算法
- **DH**, Diffie-Hellman Key Exchange, Diffie-Hellman 密钥交换
- **ECDH**, Elliptic Curve Diffie-Hellman Key Exchange, 椭圆曲线 Diffie-Hellman 密钥交换
- **IES**, Integrated Encryption Schema, 集成加密框架
- **ECIES**, Elliptic Curve Integrated Encryption Schema, 椭圆曲线集成加密框架
- **KDF**, Key Derivation Function, 密钥(私钥)生成函数

