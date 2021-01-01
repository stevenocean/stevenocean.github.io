title: Android NFC Demo (1) - NFC Reader
tags:
  - Android
  - NFC
categories:
  - Android
  - NFC
author: Steven Hu
date: 2019-03-05 20:20:20
---

# 关于 NFC

NFC，全称是Near Field Communication，中译为近场通信，也叫做近距离无线通信技术。2004 年，飞利浦、索尼和诺基亚创建了 NFC 论坛来推动推动 NFC 的发展普及和规范化。NFC 的工作频率为 13.56MHz，有效距离为 4cm 左右，目前所支持的数据传输速率有 106Kbps、212Kbps 和 424Kbps 三种。

NFC Forum 至今共推出了一系列的技术规范（下图为技术规范架构图）：

<!-- more -->

{% asset_img nfc-forum-spec-arch.jpg %}

> 参考: [NFC Forum Technical Specifications](https://nfc-forum.org/our-work/specifications-and-application-documents/specifications/nfc-forum-technical-specifications/)

- **协议技术规范 (Protocol Technical Specification)**

    该技术规范又包含如下的一系列技术规范：

    1. **NFC 的逻辑链路控制协议技术规范 (NFC Logical Link Control Protocol(LLCP) Technical Specification)**

    该规范定义了 OSI 模型第2层的协议,以支持两个具有 NFC 功能的设备之间的对等通信。

    2. **NFC 数字协议技术规范 (NFC Digital Protocol Technical Specification)**

    本规范强调了用于NFC设备通信所使用的数字协议，提供了在 ISO/IEC 18092 和 ISO/IEC 14443 标准之上的一种实现规范。通过对各种技术的综合应用，向开发人员展示了如何将 NFC, ISO/IEC 14443, 以及 JIS x6319-4 标准结合在一起来确保不同 NFC 设备之间 以及 NFC 设备与现有非接触式基础设施之间的全球互操作性。

    该规范定义了常见的特征集，这个特征集可以不做进一歩修改就可用于诸如金融服务和公共交通领域的重大 NFC 技术应用。该规范还涵盖了 NFC 设备作为发起者、目标、读写器和卡仿真器这四种角色所使用的数字接口以及半双工传输的协议。

    NFC 设备间可以使用该规范中给出的位级编码 (bit level coding)、比特率 (bit rates)、帧格式 (frame formats)、协议 (protocols) 和命令集 (command sets) 等来交换数据并绑定到 LLCP 协议。

    Version 2.0 of the Digital Protocol technical specification also adds ACM for P2P communication and NFC-V technology. Additionally, updates have been included based on ongoing alignment efforts with other organizations and standards, such as EMVCo, ISO/IEC 14443 and ISO/IEC 18092.

    Version 2.1 of the Digital Protocol technical specification adds support for larger RF frames for the contactless ISO-DEP protocol compliant with ISO/IEC 14443 was added to optimize the overall transaction time for ISO-DEP.

    3. **NFC 活动技术规范 (NFC Activity Technical Specification)**

    该规范解释了如何使用 NFC 数字协议规范与另–个 NFC 设备或 NFC Forum 标签来建立通信协议。

    4. **NFC 简单 NDEF 交换协议技术规范 (NFC Simple NDEF Exchange Protocol (SNEP) Technical Specification)**

    SNEP 规范允许一台 NFC 设备上的应用与另外一台 NFC 设备在 P2P 模式下交换 NDEF 消息，该协议利用 LLCP 面向连接传输模式来提供可靠的数据交换。

    5. **NFC 模拟技术规范 (NFC Analog Technical Specification)**

    该规范主要解决 NFC 设备的 RF 接口的模拟特性。

    6. **NFC 控制器接口技术规范 (NFC Controller Interface (NCI) Technical Specification)**

    NCI 规范了在 NFC 控制器和设备的主应用处理器之间定义 NFC 设备内的标准接口。

- **数据交换格式技术规范 (Data Exchange FormatTechnical Specification)**

    NDEF，定义了 NFC 设备之间以及设备与 tag 之间传输数据的一种消息封装格式。该协议认为设备之间传输的信息可以封装成一条 NDEF 消息,而一条消息可以包含一个或多个NDEF记录。

- **NFC 标签类型技术规范 (NFC Forum Tag TypeTechnical Specifications)**

    当前 NFC Forum 提出的 Tag 兼容基于 14443A 协议、Phlips 提供的 Mifare UltraLight 类型卡、Sony 提供的 Fecila 技术等等。

- **记录类型定义技术规范 (Record Type Definitionf Technical Specifications)**

    NFC Forum 上给出了多种不同类型的 RTD，分别为：

    - 文本记录 (Text RTD Technical Specification)
    - URI 记录 (URI RTD Technical Specification)
    - Verb 记录 (Verb RTD Technical Specification)
    - Smart Poster RTD Technical Specification
    - Generic Control RTD Technical Specification
    - Signature Record Type Definition Technical Specification
    - NFC Device Information RTD Technical Specification

- **参考应用技术规范 (Reference Application Technical Specifications)**

    该规范中主要包括 连接切换技术规范 (Connection Handover Technical Specification) 和 个人健康设备通信技术规范 (Personal Health Device Communication Technical Specification)

# NFC 的三种运行模式

从上面的技术规范架构图可以看到，站在用户角度来看，NFC 有三种运行模式，分别为：

- **Reader/Writer** 模式：简称 **RW**，可以读写无源 NFC tags 和 stickers；
- **Peer-to-Peer** 模式：简称 **P2P**，支持两个 NFC 设备之间交换数据，在 Android 中可以通过 Android Beam 与其他的 NFC 设备交换数据；
- **Card Emulation** 模式：简称 **CE**，可以把携带 NFC 的设备模拟成 Smart Card，这样就可以通过外部的 NFC Reader 来访问，如：手机支付、门禁等等；

本示例主要演示如何识别和解析 NFC 卡的信息，而这些信息都是会封装在 NDEF message 中的。

# NDEF Message 和 NDEF Record

NDEF 的数据都是被封装一条 NDEF Message（[NdefMessage](https://developer.android.com/reference/android/nfc/NdefMessage.html)） 中的，每条 NDEF 消息包含一条或多条 NDEF 记录（[NdefRecord](https://developer.android.com/reference/android/nfc/NdefRecord.html)），如下结构图：

{% asset_img nfc-ndef-message-structure-01.gif %}

每一条 NdefRecord 记录包含 Header 和 Payload，NDEF Record 的详细布局如下图：

{% asset_img nfc-ndef-record-structure.jpg %}

其中：

- **MB**，Message Begin，在 NDEF 消息的第一条 NDEF Record 必须设置此标志为 1；
- **ME**，Message End，在 NDEF 消息的最后一条 NDEF Record 必须设置此标志为 1；
- **CF**，Chunk Flag，表示该 NDEF Record 是否为分片 Record；
- **SR**，Short Record，如果设置此标志，则上图中的 PAYLOAD LENGHTH 仅需要一个，也就是 payload 长度会限制在 255 个字节以内；
- **IL**，ID LENGTH，表示 Header 中是否包含 ID_LENGTH 和 ID 字段；
- **TNF**，Type Name Format，指定 Payload 的类型，当前 NFC Forum 定义的一些类型如下表；

{% asset_img nfc-tnf-type-table.jpg %}

- **TYPE_LENGTH**，表示 Header 中 Type 字段的长度；
- **ID_LENGTH**，表示 Header 中 ID 字段的长度；
- **PAYLOAD_LENGTH**，表示 Payload 的长度；
- **TYPE**，表示 Payload 的类型；
- **ID**，The value of the ID field is an identifier in the form of a URI reference [RFC 3986](https://www.ietf.org/rfc/rfc3986.txt)

下面再了解下 TNF 和 RTD，这两个会和我们如何解析类型和 payload 相关。

# TNF（Type Name Format）

在上面的 *TNF Field Values* 表中已经列出了 NFC 支持的几种数据类型，下面分别了解这几种数据类型：

- **Empty**：表示该 Record 中没有数据，就是一个空记录；
- **NFC Forum Well-Known Type**：表示由 NFC Forum 定义的一些常用数据类型，如：URI, TEXT 等等，格式遵循 NFC RTD 规范；
- **Media-type**：即 MIME 类型，格式遵循 [RFC 2046](https://tools.ietf.org/html/rfc2046) 规范；
- **Absolute URI**：绝对 URI，格式遵循 [RFC 3986](https://tools.ietf.org/html/rfc3986) 规范；
- **NFC Forum external type**：扩展类型，格式同样遵循 NFC RTD 规范；
- **Unknown**：未知类型，这种类型的数据由应用自己解析；
- **Unchanged**：仅用于分片中，除第一个 Record 分片之外，其他的 Record 分片都必须设置 TNF 为 **Unchanged**；

# RTD (Record Type Definition)

在上面的 TNF 几大类型中，NFC Forum 通过 RTD 规范定义了其中的 WKT（Well-Known Type）和 External Type 两种类型的格式。其中：

- **WKT**

An NFC Forum Well-Known Type is a URN as defined by [RFC 2141](https://tools.ietf.org/html/rfc2141), with the namespace identifier (NID) “nfc”.

The Namespace Specific String of the NFC Well Known Type URN is prefixed with “wkt:”.

在编码进 NDEF message 中时，WKT 必须为 relative-URI 结构 [RFC 3896](https://tools.ietf.org/html/rfc3896)，忽略 NID 和 "wkt:" 前缀；

示例：对于完整的 WKT "urn:nfc:wkt:a" 在写入 NDEF message 时被编码为 "a"。

WKT 中的常用记录类型有：
  * URI: 主要存储 URI 数据，对应的 Type 字段为 "U"；
  * Text: 主要存储文本数据，对应的 Type 字段为 "T"；
  * Signature：主要存储数字签名，对应的 Type 字段为 "Sig"；
  * Smart Poster：主要存储与智能海报相关的信息，如：图片、简介等，对应的 Type 字段为 "Sp"；
  * Generic Control：主要用于传递控制信息，对应的 Type 字段为 "Gc"；

- **External Type**

第三方组织定义的类型，格式同样为 NSS (Namespace Specific String) 规范，但是需要 urn:nfc:ext 作为前缀，如："urn:nfc:ext:example.com:f"。

# Android 中的标签分发系统 (tag dispatch system)

Android 的 tag dispatch system 主要负责分析扫描的 NFC 标签中的信息，以及根据分发规则定位到那些“感兴趣”的应用，具体是下面几个步骤：

1. 分析出 NFC tag 中标识数据的 MIME type 或 URI；
2. 将 MIME type 或 URI 以及 payload 封装进 Intent 中 （具体就是封装成 NDEF Message 格式）；
3. 使用该 intent 启动相关应用注册（Intent Filter）的 Activity；

那么在底层封装好 intent 之后是如何分派到应用的呢？尤其是有多个应用 "感兴趣" 的时候，tag dispatch system 的工作机制如下图：

{% asset_img nfc_tag_dispatch.png %}

tag dispatch system 定义了三种 Intent Filter 类型，按照优先级顺序从高到低如下：

- **ACTION_NDEF_DISCOVERED**

当被扫描出来的 tag 包含 NDEF payload 并且能映射到系统支持的类型（如：MIME，URI 等），则该 ntent 用来启动相那些注册了 ACTION_NDEF_DISCOVERED 的 Activity，这是最高优先级的 Intent，会被最先分发启动 Activity。

Android 支持的 NFC Forum 数据格式有：

| NFC Forum 中的格式 | Android 映射后的数据格式 |
|:-----------------:|-----------------------|
| Absolute URI | URI |
| MIME | MIME |
| NFC Forum Well-Known Type | TEXT Record Type 映射成 text/plain; URI 和 Smart Poster Record Type 均映射成 URI 格式  |
| NFC Forum External Type | domain:service -> vnd.android.nfc://ext/domain:service |

- **ACTION_TECH_DISCOVERED**

如果没有 Activity 注册 ACTION_NDEF_DISCOVERED 来处理上面的 Intent，或者 NFC Tag 中的数据不能转换成系统直接支持的类型，或者数据没有使用 NDEF message 格式时，则 tag dispatch system 会尝试发送 ACTION_TECH_DISCOVERED 的 Intent。

- **ACTION_TAG_DISCOVERED**

如果仍然没有 Activity 对 ACTION_TECH_DISCOVERED 感兴趣，则 tag dispatch system 将最后尝试发送一个 ACTION_TAG_DICOVERED 的 Intent（包含一个Tag对象）。

# 示例

下面仅实现了一个简单的扫描和解析一张门禁卡的示例。这张门禁卡内部芯片采用的是 NXP 的 Mifare Classic，存储容量 1KB。


下面是 AndroidManifest.xml 文件：

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="io.github.stevenocean.nfcsample">

    <uses-permission android:name="android.permission.NFC" />

    <uses-feature android:name="android.hardware.nfc" android:required="true" />

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
            <intent-filter>
                <action android:name="android.nfc.action.TECH_DISCOVERED" />
            </intent-filter>
            <meta-data
                android:name="android.nfc.action.TECH_DISCOVERED"
                android:resource="@xml/nfc_tech_filter" />
        </activity>
    </application>
</manifest>
```

下面是 MainActivity 源码：

```java
public class MainActivity extends AppCompatActivity {

    private NfcAdapter mNfcAdapter;
    private TextView mNfcInfoText;
    private static final String TAG = MainActivity.class.getName();
    private PendingIntent mPendingIntent;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        mNfcInfoText = findViewById(R.id.nfc_info_tv);

        mNfcAdapter = NfcAdapter.getDefaultAdapter(this);
        if (null == mNfcAdapter) {
            mNfcInfoText.setText("Not support NFC!");
            finish();
            return;
        }

        if (!mNfcAdapter.isEnabled()) {
            mNfcInfoText.setText("Please open NFC!");
            finish();
            return;
        }

        if (getIntent() != null) {
            processIntent(getIntent());
        }
    }

    @Override
    protected void onNewIntent(Intent intent) {
        super.onNewIntent(intent);
        if (intent != null) {
            processIntent(intent);
        }
    }

    private void processIntent(Intent intent) {

        if (!NfcAdapter.ACTION_TECH_DISCOVERED.equals(intent.getAction())) {
            Toast.makeText(this, "Invalid action", Toast.LENGTH_LONG).show();
            return;
        }

        StringBuilder nfcInfo = new StringBuilder();

        byte[] extraId = intent.getByteArrayExtra(NfcAdapter.EXTRA_ID);

        // Id
        if (extraId != null) {
            nfcInfo.append("ID (hex): ").append(encodeHexString(extraId)).append("\n");
        }

        // Tag info
        Tag tag = intent.getParcelableExtra(NfcAdapter.EXTRA_TAG);

        // Technologies
        StringBuilder technologiesAvailable = new StringBuilder("Technologies Available: \n");

        // Card type.
        StringBuilder cardType = new StringBuilder("Card Type: \n");

        // Sector and block.
        StringBuilder sectorAndBlock = new StringBuilder("Storage: \n");

        // Sector check
        StringBuilder sectorCheck = new StringBuilder("Sector check: \n");

        int idx = 0;
        for (String tech : tag.getTechList()) {
            if (tech.equals(MifareClassic.class.getName())) {
                // Mifare Classic
                MifareClassic mfc = MifareClassic.get(tag);
                switch (mfc.getType()) {
                    case MifareClassic.TYPE_CLASSIC:
                        cardType.append("Classic");
                        break;

                    case MifareClassic.TYPE_PLUS:
                        cardType.append("Plus");
                        break;

                    case MifareClassic.TYPE_PRO:
                        cardType.append("Pro");
                        break;

                    case MifareClassic.TYPE_UNKNOWN:
                        cardType.append("Unknown");
                        break;
                }

                sectorAndBlock.append("Sectors: ").append(mfc.getSectorCount()).append("\n")
                        .append("Blocks: ").append(mfc.getBlockCount()).append("\n")
                        .append("Size: ").append(mfc.getSize()).append(" Bytes");

                try {
                    // Enable I/O to the tag
                    mfc.connect();

                    for (int i = 0; i < mfc.getSectorCount(); ++i) {
                        if (mfc.authenticateSectorWithKeyA(i, MifareClassic.KEY_DEFAULT)) {
                            sectorCheck.append("Sector <").append(i).append("> with KeyA auth succ\n");

                            // Read block of sector
                            final int blockIndex = mfc.sectorToBlock(i);
                            for (int j = 0; j < mfc.getBlockCountInSector(i); ++j) {
                                byte[] blockData = mfc.readBlock(blockIndex+j);
                                sectorCheck.append("  Block <").append(blockIndex+j).append("> ")
                                        .append(encodeHexString(blockData)).append("\n");
                            }

                        } else if (mfc.authenticateSectorWithKeyB(i, MifareClassic.KEY_DEFAULT)) {
                            sectorCheck.append("Sector <").append(i).append("> with KeyB auth succ\n");

                            // Read block of sector
                            final int blockIndex = mfc.sectorToBlock(i);
                            for (int j = 0; j < mfc.getBlockCountInSector(i); ++j) {
                                byte[] blockData = mfc.readBlock(blockIndex+j);
                                sectorCheck.append("  Block <").append(blockIndex+j).append("> ")
                                        .append(encodeHexString(blockData)).append("\n");
                            }
                        } else {
                            sectorCheck.append("Sector <").append(i).append("> auth failed\n");
                        }
                    }
                } catch (IOException e) {
                    e.printStackTrace();
                    Toast.makeText(this, "Try again and keep NFC tag below device", Toast.LENGTH_LONG).show();
                }
            } else if (tech.equals(MifareUltralight.class.getName())) {
                // Mifare Ultralight
                MifareUltralight mful = MifareUltralight.get(tag);
                switch (mful.getType()) {
                    case MifareUltralight.TYPE_ULTRALIGHT:
                        cardType.append("Ultralight");
                        break;

                    case MifareUltralight.TYPE_ULTRALIGHT_C:
                        cardType.append("Ultralight C");
                        break;

                    case MifareUltralight.TYPE_UNKNOWN:
                        cardType.append("Unknown");
                        break;
                }
            }

            String [] techPkgFields = tech.split("\\.");
            if (techPkgFields.length > 0) {
                final String techName = techPkgFields[techPkgFields.length-1];
                if (0 == idx++) {
                    technologiesAvailable.append(techName);
                } else {
                    technologiesAvailable.append(", ").append(techName);
                }
            }
        }

        nfcInfo.append("\n").append(technologiesAvailable).append("\n")
                .append("\n").append(cardType).append("\n");

        // NDEF Messages
        StringBuilder sbNdefMessages = new StringBuilder("NDEF Messages: \n");
        Parcelable[] rawMessages = intent.getParcelableArrayExtra(NfcAdapter.EXTRA_NDEF_MESSAGES);
        if (rawMessages != null) {
            NdefMessage[] messages = new NdefMessage[rawMessages.length];
            for (int i = 0; i < rawMessages.length; ++i) {
                messages[i] = (NdefMessage) rawMessages[i];
            }

            for (NdefMessage message : messages) {
                for (NdefRecord record : message.getRecords()) {
                    if (record.getTnf() == NdefRecord.TNF_WELL_KNOWN) {
                        if (Arrays.equals(record.getType(), NdefRecord.RTD_TEXT)) {
                            try {
                                // NFC Forum "Text Record Type Definition" section 3.2.1.
                                byte[] payload = record.getPayload();
                                String textEncoding = ((payload[0] & 0200) == 0) ? "UTF-8" : "UTF-16";
                                int languageCodeLength = payload[0] & 0077;
                                String languageCode = new String(payload, 1, languageCodeLength, "US-ASCII");
                                String text = new String(payload, languageCodeLength + 1,
                                        payload.length - languageCodeLength - 1, textEncoding);
                                sbNdefMessages.append(" - ").append(languageCode).append(", ")
                                        .append(textEncoding).append(", ").append(text).append("\n");
                            } catch (UnsupportedEncodingException e) {
                                // should never happen unless we get a malformed tag.
                                throw new IllegalArgumentException(e);
                            }
                        }
                    }
                }
            }
        }
        nfcInfo.append("\n").append(sbNdefMessages).append("\n")
                .append("\n").append(sectorAndBlock).append("\n")
                .append("\n").append(sectorCheck).append("\n");

        mNfcInfoText.setText(nfcInfo.toString());
    }

    private String byteToHex(byte num) {
        char[] hexDigits = new char[2];
        hexDigits[0] = Character.forDigit((num >> 4) & 0xF, 16);
        hexDigits[1] = Character.forDigit((num & 0xF), 16);
        return new String(hexDigits);
    }

    private String encodeHexString(byte[] byteArray) {
        StringBuilder hexStringBuffer = new StringBuilder();
        for (byte aByteArray : byteArray) {
            hexStringBuffer.append(byteToHex(aByteArray));
        }
        return hexStringBuffer.toString();
    }

}
```

运行之后，将门禁卡放置在手机的背面之后，通过 *ACTION_TECH_DISCOVERED* action 自动调起上面的 Activity，并调用其中的 *processIntent* 方法，传入的 intent 中包含有 tag 数据，在 *processIntent* 中将 ID, 支持的 Technologies, 卡类型, 以及扇区和内存信息都解析出来了，另外，我在这张门禁卡中还 write 了一条标准的 Text 记录，该记录会被封装在 NDEF message 中，如下截图为 demo 扫码和解析门禁卡后的信息：

{% asset_img nfc-demo-1.png %}

计划下期 demo 通过 Card Emulation 模式将手机模拟为一张门禁卡 :)

> 参考资料:
> 
> - [**Android NFC basics**](https://developer.android.com/guide/topics/connectivity/nfc/nfc)


