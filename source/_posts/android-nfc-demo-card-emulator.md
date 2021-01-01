title: Android NFC Demo (2) - Card Emulator
tags:
  - Android
  - NFC
  - HCE
categories:
  - Android
  - NFC
  - HCE
author: Steven Hu
date: 2019-03-12 20:20:20
---

# Overview

我们想通过手机来模拟成智能卡(Smart Card)，很多情况下，都是通过设备上的称为 Secure Element (以下简称 SE)的安全芯片来模拟的，譬如很多运营商提供的 SIM 卡中也会内嵌有这样的 SE 芯片（例如：中国移动和银联推的 NFC-SIM 卡-云闪付）。SE 芯片一般在设备出厂前就已经内嵌在板子上了，无法替换，并且 SE 上的系统主要负责处理安全支付方面的工作。

通过下面的使用 SE 来模拟 Card 的结构图，可以看到，这里的 transactions 都是由 SE 芯片直接和 NFC 读卡器进行通信和交互的，而不需要其他 Android 应用的参与，在 transaction 完成之后，Android 应用可以查询状态并通知用户：

<!-- more -->

{% asset_img card-emulator-with-secure-element.png %}

而我们这里是想用另外一种不依赖于 SE 芯片的方案来模拟 Card，也就是基于主机的卡模拟（Host-based Card Emulator），在此方案下，交互的数据被直接路由到主机 CPU 上，而不是 SE 芯片上，这样直接运行在主机 CPU 上的 Android 应用就可以与 NFC 读卡器“直接”交互了，如下图：

{% asset_img card-emulator-with-hce.png %}

# HCE 支持的卡和协议栈

NFC 标准提供了对许多不同协议的支持，以及许多可以被模拟的卡类型。

从 Android 4.4 开始就已经支持目前市场上的一些常见类型的卡，如非接触式的支付卡，下图为 Android HCE 支持的协议栈：

{% asset_img hce-protocol-stack.png %}

上图列出了协议栈的分层，以及各层基于的规范。

目前市场上的许多NFC读卡器也都支持这些协议，包括作为读卡器的Android NFC设备。这样我们就可以基于 HCE 方案仅使用 Android 设备来构建和部署端到端NFC解决方案。

另外，Android 4.4 主要支持模拟的是基于 NFC-Forum ISO-DEP 规范（ISO/IEC 14443-4）以及处理定义于 ISO/IEC 7816-4 规范中的 APDU（Application Protocol Data Units）协议数据。Android 目前要求仅在 Nfc-A (ISO/IEC 14443-3 Type A) 技术之上模拟 ISO-DEP，对于 Nfc-B (ISO/IEC 14443-3 Type B) 技术的支持是可选的。

# HCE Service

在 Android 中 HCE 架构是基于 Service 组件的（也就是 HCE services），这样可以在后台运行，

在 ISO/IEC 7816-4 规范中定义了一种选择应用的方式，即以 Application ID（AID）为中心，AID 由最多 16 个字节组成的标识，如果要为现有的NFC读卡器设施模拟卡片，那些 reader 寻找的AID通常都是 well-known 并且是公开注册的，例如：支付领域的 Visa, MasterCard 的 AID。

如果你需要部署自己应用的新读卡器设施，就需要注册自己的 AID，注册过程参考 ISO/IEC 7816-5 规范。

## 关于 AID 组

有时，HCE 服务可能需要注册多个 AID 来实现某个应用程序，并且需要确保它是所有这些 AID 的默认处理程序（而不是这个 AID 组中的某些 AID 是另外的 HCE 服务来处理）。

一个组中的所有 AID 要么都被路由到 HCE 服务，要么所有 AID 都不会被路由到 HCE 服务。

每一个 AID 组都可以被关联到某个分类，从 Android 4.4 开始支持两个分类，分别为：*CATEGORY_PAYMENT* 和 *CATEGORY_OTHER*，关于分类，还有些规则和限制：

- 在任何给定时间，系统中只能启用 *CATEGORY_PAYMENT* 类别中的一个AID组。
- 对于仅在某个指定商家使用的卡，应该设置为 *CATEGORY_OTHER* 。此类别中的AID组可以始终处于活动状态，并且必要时可以在 AID 选择期间由 NFC 读卡器给予优先级。

## 实现 HCE 服务

为了能够使用 HCE 模式来模拟 NFC 卡，需要实现一个 [HostApduService](https://developer.android.com/reference/android/nfc/cardemulation/HostApduService.html) 接口的 Service 组件来处理 NFC 交互 transactions。

```java
public class MyHostApduService extends HostApduService { 
    @Override 
    public byte[] processCommandApdu(byte[] apdu, Bundle extras) {
       ... 
    } 
    @Override 
    public void onDeactivated(int reason) {
       ... 
    } 
}
```

在 [HostApduService](https://developer.android.com/reference/android/nfc/cardemulation/HostApduService.html) 中声明了两个需要重载和实现的抽象方法：

### processCommandApdu()

NFC 读卡器发送 APDU 到服务时调用此方法。APDU 定义在 ISO/IEC 7816-4 规范中，APDU 是 NFC 读卡器 和 HCE 服务之间传递数据的应用层协议单元，并且该协议是半双工的，即 NFC 读卡器在发送一个 APDU 命令后，会同步等待 HCE 服务返回一个 APDU 的响应。

#### APDU

下图为 APDU 协议的 command-response 结构，主要由 Header 和 Body 两个部分组成，并且 Body 部分由 Lc, Data 和 Le 组成。

{% asset_img apdu-command-response-pair.jpg %}

- *CLA*: Class byte, 表示命令的类别，占 1 个字节，8 个 bit 位的使用有一定的规范，如：Bit 8 用于区分产业还是专有分类（详细定义参考 *ISO/IEC 7816-4 5.1.1*）；

- *INS*: Instruction byte，表示处理的指令，占 1 个字节，不同指令表示不同的意义，如下图定义：

{% asset_img apdu-ins-commands.jpg %}

- *P1 & P2*: Parameter bytes，可用于指令的参数；
- *Lc*: 表示发送的 command data 字段的长度；
- *Le*：表示期待的 response data 字段的长度；
- *Command Data Field*：表示实际的指令数据；
- *Response Data Field*：响应数据；
- *SW1 & SW2*：Status bytes，根据规范，状态码均为 6xxx 和 9xxx，并且 60xx 也是无效的，其他的详细规范参考 *ISO/IEC 7816-3*，下图是规范定义状态码值结构图：

{% asset_img structural-scheme-of-sw1-sw2.jpg %}

#### 交互处理

如前所述，Android 是通过 AID 来决定 读卡器 需要交互的 HCE 服务。通常情况下， NFC 读卡器发送的第一条 APDU 就是 “选择 AID"，该条 APDU 中就包含了 读卡器 希望与之交互的 AID，Android 从该 APDU 中解析出 AID 并转发至相应的 HCE 服务。

应用层可以在 processCommandApdu 方法中返回响应 APDU，要注意的是，该方法会在主线程中被调用，因此不能被阻塞，可以在其他线程中做些复杂计算，并通过 sendResponseApdu() 方法来返回响应。

### onDeactivated()

在此过程中，Android 会持续向你的 HCE 服务转发新的 APDU，直到：

- NFC 读卡器发送了另外一个 “SELECT AID" APDU，并且解析到了其他的 HCE 服务；
- NFC 读卡器和设备之间的链路断开；

上面两种情形下 onDeactivated() 方法都会被调用，传入的参数 reason 就是指明是什么原因引起的。

# 示例

下面我们分别来写两个程序，并且分别安装在两台不同的 Android 手机上：

- *CardEmulatorSample*: 用于模拟卡，主要实现 HCE 服务；
- *CardReaderSample*: 作为 NFC 读卡器来读取 *CardEmulatorSample* 中模拟的卡信息；

## CardEmulatorSample

### 实现卡模拟服务

如下 **HostCardEmulatorService** 派生于 HostApduService 并重载实现了 processCommandApdu 和 onDeactivated 方法。另外，我们根据规范自定义了一些常量，如：成功状态，失败状态，AID 等等。

```java
public class HostCardEmulatorService extends HostApduService {

    public static final String TAG = HostCardEmulatorService.class.getName();

    // self-defined APDU
    public static final String STATUS_SUCCESS = "9000";
    public static final String STATUS_FAILED = "6F00";
    public static final String CLA_NOT_SUPPORTED = "6E00";
    public static final String INS_NOT_SUPPORTED = "6D00";
    public static final String AID = "A0000002471001";
    public static final String SELECT_INS = "A4";
    public static final String DEFAULT_CLA = "00";
    public static final int MIN_APDU_LENGTH = 12;

    @Override
    public byte[] processCommandApdu(byte[] commandApdu, Bundle extras) {

        Toast.makeText(this, "processCommandApdu", Toast.LENGTH_LONG).show();

        if (commandApdu == null) {
            return hexStringToByteArray(STATUS_SUCCESS);
        }

        final String hexCommandApdu = encodeHexString(commandApdu);
        if (hexCommandApdu.length() < MIN_APDU_LENGTH) {
            return hexStringToByteArray(STATUS_FAILED);
        }

        if (!hexCommandApdu.substring(0, 2).equals(DEFAULT_CLA)) {
            return hexStringToByteArray(CLA_NOT_SUPPORTED);
        }

        if (!hexCommandApdu.substring(2, 4).equals(SELECT_INS)) {
            return hexStringToByteArray(INS_NOT_SUPPORTED);
        }

        if (hexCommandApdu.substring(10, 24).equals(AID))  {
            return hexStringToByteArray(STATUS_SUCCESS);
        } else {
            return hexStringToByteArray(STATUS_FAILED);
        }
    }

    @Override
    public void onDeactivated(int reason) {
        Toast.makeText(this, "Deactivated - reason:" + reason, Toast.LENGTH_LONG).show();
    }
}
```

另外，这里涉及到一些十六进制和字符串的转换，都是用的如下三个方法：

```java
     public static String byteToHex(byte num, boolean upper) {
        char[] hexDigits = new char[2];
        if (upper) {
            hexDigits[0] = Character.toUpperCase(Character.forDigit((num >> 4) & 0xF, 16));
            hexDigits[1] = Character.toUpperCase(Character.forDigit((num & 0xF), 16));
        } else {
            hexDigits[0] = Character.forDigit((num >> 4) & 0xF, 16);
            hexDigits[1] = Character.forDigit((num & 0xF), 16);
        }

        return new String(hexDigits);
    }

    public static String encodeHexString(byte[] byteArray, boolean upper) {
        StringBuilder hexStringBuffer = new StringBuilder();
        for (byte aByteArray : byteArray) {
            hexStringBuffer.append(byteToHex(aByteArray, upper));
        }
        return hexStringBuffer.toString();
    }

    public static byte[] hexStringToByteArray(String s) {
        int len = s.length();
        byte[] data = new byte[len / 2];
        for (int i = 0; i < len; i += 2) {
            data[i / 2] = (byte) ((Character.digit(s.charAt(i), 16) << 4)
                    + Character.digit(s.charAt(i+1), 16));
        }
        return data;
    }
```

之后，我们还需要在 AndroidManifest.xml 中添加如下 service 配置项及相应的 Intent-Filter：

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="io.github.stevenocean.cardemulatorsample">

    <uses-permission android:name="android.permission.NFC" />

    <uses-feature android:name="android.hardware.nfc.hce" android:required="true" />

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <service
            android:name=".HostCardEmulatorService"
            android:exported="true"
            android:permission="android.permission.BIND_NFC_SERVICE">
            <intent-filter>
                <action android:name="android.nfc.cardemulation.action.HOST_APDU_SERVICE" />
            </intent-filter>
            <meta-data
                android:name="android.nfc.cardemulation.host_apdu_service"
                android:resource="@xml/apduservice" />
        </service>
    </application>

</manifest>
```

- *android:name*：启动的 HostApduService 实现类，这里为 HostCardEmulatorService；
- *android:exported*：这里设置为 true，可以被其他应用访问；
- *android:permission*：需要有绑定 NFC 服务的权限；
- *intent-filter*：当系统检测到有外部 NFC 读卡器尝试读取卡信息的时候，就会触发一个 **HOST_APDU_SERVICE** 的 action，因此我们的服务需要注册该 action；
- *meta-data*：为了让系统能够选择正确 AID 的服务来调用，需要配置 AID 的信息，这里指向 apduservice.xml 文件；

```xml
<?xml version="1.0" encoding="utf-8"?>
<host-apdu-service xmlns:android="http://schemas.android.com/apk/res/android"
    android:description="@string/servicedesc"
    android:requireDeviceUnlock="false">
    <aid-group android:description="@string/aiddescription"
        android:category="other">
        <aid-filter android:name="A0000002471001"/>
    </aid-group>
</host-apdu-service>
```

这里配置了 AID 组分类（为 CATEGORY_OTHER），以及组中的 AID（为 A0000002471001）。

## 实现 NFC 读卡器

我们这里的 NFC 读卡器示例还是基于前一篇文章中的示例进行扩展，首先在 MainActivity 的 中添加对 IsoDep 的处理，其中 onTagDiscovered(Tag tag) 方法为 NfcAdapter.ReaderCallback 回调，如下：

```java
    @Override
    protected void onResume() {
        super.onResume();
        mNfcAdapter.enableReaderMode(this, this,
                NfcAdapter.FLAG_READER_NFC_A | NfcAdapter.FLAG_READER_SKIP_NDEF_CHECK,
                null);
    }

    @Override
    protected void onPause() {
        super.onPause();
        mNfcAdapter.disableReaderMode(this);
    }

    @Override
    public void onTagDiscovered(Tag tag) {

        // Card response for IsoDep
        final StringBuilder cardResp = new StringBuilder("Card response: \n");

        // read card data of CardEmulator
        IsoDep isoDep = IsoDep.get(tag);
        try {
            isoDep.connect();
            byte [] resp = isoDep.transceive(hexStringToByteArray(DEFAULT_CLA + SELECT_INS + "0400" + LC + AID));
            String respStatus = encodeHexString(resp, true);
            if (respStatus.equals(STATUS_SUCCESS)) {
                cardResp.append("Success response");
            } else {
                cardResp.append("Failed response, code:").append(respStatus);
            }
            runOnUiThread(new Runnable() {
                @Override
                public void run() {
                    mNfcInfoText.setText(cardResp.toString());
                }
            });
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```

## 测试

分别在两台 Android 手机上安装并打开 CardEmulatorSample 和 CardReaderSample 程序，之后将两台手机靠近，可以在 CardReaderSample 中看到如下信息：

{% asset_img reader-sample-result-card-resp.png %}

## 示例代码

在 github 上可以下载和尝试本文示例代码

- [CardEmulatorSample](https://github.com/stevenocean/CardEmulatorSample)
- [CardReaderSample](https://github.com/stevenocean/CardReaderSample)


# 参考资料

- [Host-based card emulation overview](https://developer.android.com/guide/topics/connectivity/nfc/hce)
- [ISO/IEC 7816-3](http://read.pudn.com/downloads132/doc/comm/563504/ISO-IEC%207816/ISO%2BIEC%207816-3-2006.pdf)
- [ISO/IEC 7816-4](http://www.embedx.com/pdfs/ISO_STD_7816/info_isoiec7816-4%7Bed21.0%7Den.pdf)
